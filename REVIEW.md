# Anti-Cheat Detector H5 — 审查报告

> 审查日期: 2026-05-12
> 审查版本: commit 69696fd
> 审查人: Claude Opus 4.6
> 项目规模: 单文件 839 行 / 40+ 检测项 / 零依赖

---

## 综合评分: 5.5 / 10

| 维度 | 分数 | 一句话 |
|------|------|--------|
| PoC / 演示价值 | 7.0 | 单文件零依赖, 界面完整, 适合展示检测思路 |
| 真实反作弊价值 | 4.0 | **模拟器得 70-80 分, Headless Chrome 得 100 分, 核心功能失效** |
| 检测覆盖面 | 8.0 | 8 类 40+ 检测项, 信号采集广度好 |
| 检测准确性 | 4.0 | 判定标准过宽, 6 项永远 pass, info 逃逸严重 |
| 架构设计 | 7.0 | 两阶段架构清晰, 但单文件 839 行难维护 |
| 代码健壮性 | 5.0 | 存在未声明变量崩溃、异步检测漏报等真实 bug |
| 安全性 | 6.0 | 安全检测工具自身存在 DOM XSS 风险, 外部 API 单点依赖 |
| 工程化 | 2.5 | 无测试、无 CI、无模块拆分、无设备样本回归 |
| UI/UX | 8.0 | 暗色主题美观, 实时进度, 交互完整 |

**定位为 PoC 演示有价值, 但距离生产级反作弊有明显差距。**

---

## 一、致命问题: 模拟器得分 70-80

用 MuMu / 雷电两款主流模拟器实测, 得分 70-80, 被判定为"存在风险"甚至"设备安全"。对于反作弊检测器来说, **放过模拟器等于功能失效**。

根因是三个系统性设计缺陷叠加:

### 1.1 `info` 逃逸 — 关键检测失败不扣分

评分公式 `score = 100 × pass / (pass + risk)`, `info` 完全不参与计算。

模拟器上大量本应判 `risk` 的检测走了 `info`:

| 检测项 | 行号 | 模拟器表现 | 当前判定 | 正确判定 | 原因 |
|--------|------|-----------|---------|---------|------|
| 电池状态 | 281 | API 不可用 | `info` | `risk` | 真 Android 必有 Battery API |
| 传感器 | 311 | 2/3 失败 | `info` | `risk` | 门槛 `miss>=3` 太宽松, 2个就够 |
| DeviceMotion | 330 | 无数据 | 取决于画像 | `risk` | 若画像误判为非移动端则逃逸 |
| Geolocation | 428 | 超时 | `info` | `info` | 可接受 |
| WebRTC IP | 442 | 发现内网 IP | `info` | `info` | 可接受 |
| 屏幕特征 | 349 | DPR=1 | `info` | `risk` | 模拟器 DPR=1 是强信号 |
| 浏览器版本 | 539 | Chrome 115-134 | `info` | 至少 `info` | 可接受 |

**量化影响**: 假设 40 项中 25 个 pass、5 个 risk、10 个 info, 分数 = 100×25/30 = **83 分**。10 个 info 等于白检测。

### 1.2 六项检测永远返回 `pass` — 给模拟器送分

以下检测无论设备真假, 永远返回 `pass`:

| 检测项 | 行号 | 问题 |
|--------|------|------|
| Math 精度指纹 | 670 | 注释写"只采集不判定", 永远 `pass` |
| WebGL 渲染指纹 | 691 | 只输出哈希, 无基线对比, 永远 `pass` |
| Performance 计时 | 741 | "不同设备性能差异大", 放弃判定, 永远 `pass` |
| 时间源精度 | 786 | 精度受限判为正常, 永远 `pass` |
| Canvas 指纹 | 373 | 只输出哈希, 无对比, 永远 `pass` |
| AudioContext 指纹 | 385 | 同上 |

这 6 个白送的 pass 让模拟器分数凭空高出 ~15%。

### 1.3 阶段 1 画像误判 → 阶段 2 豁免链条崩溃

新版模拟器 (MuMu 12, 雷电 9) 的伪装能力:

```
UA:       Mozilla/5.0 (Linux; Android 12; ...) Chrome/120 ...
Platform: Linux armv8l         ← 翻译层模拟出 ARM
GPU:      Adreno 630           ← 虚拟 GPU, 名字对
Touch:    5                    ← 模拟触摸点
Cores:    4-8                  ← 宿主机分配
```

阶段 1 对此的评分:
- `pAndroid += 25` (UA 含 android + platform 含 linux arm)
- `pAndroid += 15` (Mobile GPU 非 Apple)
- `pAndroid += 10` (有 Battery + Touch > 0 + UA android)
- `pEmu` 可能只得 0-15 分 (如果 Chrome >= 115 且 platform 伪装成 ARM)

结果: **Android 50 vs 模拟器 15**, 画像输出 "Android 置信度 77%"。

画像一旦定为 Android, 阶段 2 的 `isAndroid()=true, isEmu()=false`, 所有"桌面端豁免"不生效, 但"移动端严格检查"又被 `info` 逃逸绕过。

### 1.4 模拟器得分模拟推演

以 MuMu 12 为例, 逐项推演 40 项检测的预期结果:

```
环境验证 (8项):
  WebGL 渲染器      → pass  (Adreno 630 不是软件渲染)
  GPU vs 平台一致性  → pass  (ARM GPU + Linux ARM 一致)
  电池状态           → pass/info (MuMu 有 Battery API 但值可疑)
  传感器             → info  (2/3 失败, 门槛要3个才risk)
  DeviceMotion       → risk  (加速度全 null)
  硬件信息           → pass  (4核 Touch=5)
  屏幕特征           → pass/info (DPR=1 但只标info)
  媒体设备           → risk  (0摄像头0麦克风)

指纹采集 (5项):
  Canvas 指纹        → pass  (永远pass)
  AudioContext 指纹  → pass  (永远pass)
  字体指纹           → pass  (模拟器字体够多)
  UA/Platform 一致性 → pass  (伪装一致)
  时区与语言         → pass  (正常)

网络与位置 (4项):
  Geolocation        → info  (超时或假坐标)
  WebRTC IP          → info  (内网IP)
  数据中心 IP        → pass  (非数据中心)
  Proxy/VPN          → pass  (时区语言一致)

工具检测 (6项):
  端口扫描           → pass  (默认不开 Frida)
  自动化框架         → pass  (不是 Selenium)
  Headless 深度检测  → pass  (有 window.chrome)
  DevTools 检测      → pass  (没开)
  浏览器版本         → pass/info (Chrome 120)
  (注: 端口扫描串行8个×1.2s, 最坏9.6秒)

完整性 (4项):
  原生函数           → pass  (未 Hook)
  Console 劫持       → pass  (正常)
  iframe 环境        → pass  (非iframe)
  Service Worker     → pass  (无SW)

行为分析 (7项):
  WebView 检测       → pass  (非WebView)
  多开检测           → pass  (单标签)
  属性一致性         → pass  (伪装一致)
  Storage 可用性     → pass  (全部可用)
  页面生命周期       → pass  (正常)
  FP/FCP/LCP         → pass  (正常)
  (注: 以上全部 pass — 行为分析对模拟器完全无效)

深度指纹 (9项):
  Math 精度指纹      → pass  (永远pass)
  WebGL 渲染指纹     → pass  (永远pass)
  AudioContext 延迟  → pass  (有音频设备)
  屏幕物理尺寸       → pass  (估算在范围内)
  Vibration API      → pass/risk (取决于模拟器是否实现)
  Performance 计时   → pass  (永远pass)
  平台架构验证       → pass  (ARM伪装一致)
  WebGL 扩展数量     → pass  (模拟器扩展够多)
  时间源精度         → pass  (永远pass)

综合判定 (2项):
  设备类型概率       → pass  (Android 高概率)
  云真机判定         → pass  (信号不足3个)
```

**统计: ~30 pass, ~3 risk, ~7 info → 分数 = 100×30/33 ≈ 91 分**

最好情况也有 70-80 分。问题一目了然。

---

## 二、真实 Bug (代码缺陷)

### 2.1 `DeviceMotionEvent` 未声明导致整页崩溃

位置: `index.html:170`, `index.html:296`, `index.html:318`

```javascript
typeof DeviceMotionEvent?.requestPermission === 'function'
```

`DeviceMotionEvent` 不是属性访问, 是直接引用全局变量。如果浏览器不存在该全局变量, 可选链 `?.` **无法救未声明变量**, 会抛 `ReferenceError`。170 行位于 `identifyDevice()` 初始化阶段, 崩溃后整个检测停止, 页面白屏。

**修复**: 改为 `typeof window.DeviceMotionEvent?.requestPermission === 'function'` 或先 `typeof DeviceMotionEvent !== 'undefined'` 守卫。

### 2.2 WebRTC IP 超时不调用 `report()`

位置: `index.html:446`

```javascript
setTimeout(()=>{try{pc.close()}catch(e){}rv()},3000);
```

超时分支只调用了 `rv()` (resolve Promise), 没有调用 `report()`。如果浏览器没有触发 `onicecandidate(null)`, 该检测项永远显示"检测中", 进度条无法到 100%, 评分统计缺项。

**修复**: 超时时必须 `report('WebRTC IP', 'info', '检测超时', {})` 后再 `rv()`。所有异步检测都应保证 exactly-once 上报。

### 2.3 DOM XSS — 环境字符串直接拼接 HTML

位置: `index.html:241`

```javascript
tags.innerHTML = p.tags.map(t=>`<span class="tag">${t}</span>`).join('');
```

`p.tags` 包含 GPU renderer、platform 等浏览器环境字符串。在反作弊场景中, 这些值可能被自动化框架或注入脚本伪造为含 HTML/JS 的恶意字符串。**安全检测工具自身存在注入漏洞**是严重的信誉问题。

**修复**: 改用 `document.createElement()` + `textContent`。所有浏览器环境字段按不可信输入处理。

### 2.4 `file://` 下 CORS 失败, README 声称不符

位置: `README.md:5`, `index.html:454`

README 写"双击即开", 但 `fetch('https://ipapi.co/json/')` 在 `file://` 环境下触发 CORS 拦截, 导致数据中心 IP 检测失败。

### 2.5 README 声称的检测项未实现

README 提及"触摸事件来源 isTrusted"、"按键事件注入"、"插件数量"等检测, 源码中均未找到对应实现。文档夸大了检测范围。

---

## 三、评分模型缺陷

### 2.1 线性等权模型不合理

当前所有检测项权重相等: Canvas 指纹 pass = GPU 一致性 pass = 自动化框架 pass。

但实际上:
- "自动化框架检测到 Selenium" 应该一票否决
- "GPU vs 平台不一致" 几乎 100% 确认模拟器
- "Canvas 指纹正常" 只能说明浏览器能画图, 没有安全意义

### 2.2 缺少"一票否决"机制

某些信号应直接判定高风险, 不受 pass 数量稀释:

- 自动化框架被检出 (Selenium/Phantom/Puppeteer)
- GPU 与平台矛盾 (ARM GPU on x86)
- navigator.webdriver = true
- Frida 端口开放

当前实现: 这些检测报 `risk` 后被 30 个 `pass` 稀释到 30/33 = 91 分。

### 2.3 `info` 状态定义模糊

`info` 被用于两种完全不同的场景:
1. **真正的信息采集** (Canvas 哈希、时区) — 不应影响分数, 合理
2. **检测失败的降级** (电池不可用、传感器超时) — 对移动端是异常, 应扣分

混用同一个状态导致评分失真。

---

## 四、检测逻辑问题 (逐项)

### 3.1 判定标准过宽

| 检测项 | 行号 | 问题 | 建议 |
|--------|------|------|------|
| 电池状态 | 281 | Android 无 Battery API 给 `info` | Android 应给 `risk` |
| 电池状态 | 289 | 只检查 `level===1 && charging && chargingTime===0` | 增加 `dischargingTime===Infinity` 组合 |
| 传感器 | 311 | 门槛 `miss>=3` | 降为 `miss>=2` |
| 屏幕特征 | 349 | 移动端 DPR=1 给 `info` | 结合分辨率应给 `risk` |
| Headless | 514 | 单信号给 `info` | 应给 `risk` |
| 浏览器版本 | 539 | Chrome 115-134 给 `info` | 结合 Android + 模拟器信号提升权重 |

### 3.2 检测无实际判定 (只采集)

| 检测项 | 行号 | 现状 | 可改进方向 |
|--------|------|------|-----------|
| Math 精度指纹 | 652-671 | 只输出哈希, 永远 `pass` | 对比已知 x86/ARM 基线值 |
| WebGL 渲染指纹 | 673-692 | 只输出哈希, 永远 `pass` | 建立模拟器 GPU 渲染基线库 |
| Performance 计时 | 733-742 | 只计时, 永远 `pass` | ARM 翻译层比原生慢 3-5x, 可设阈值 |
| Canvas 指纹 | 367-374 | 只输出哈希, 永远 `pass` | 收集已知模拟器哈希黑名单 |
| AudioContext 指纹 | 376-387 | 只输出哈希, 永远 `pass` | 同上 |

这些检测项的存在制造了"40+ 检测很全面"的错觉, 实际对模拟器识别贡献为零且反向稀释分数。

### 3.3 阶段 1 画像遗漏的模拟器信号

阶段 1 `identifyDevice()` (152-231行) 缺少以下可在同步阶段采集的模拟器信号:

| 信号 | 真 Android | 模拟器 | 检测方法 |
|------|-----------|--------|---------|
| Battery API 存在性 | 必有 | 可能没有 | `!!navigator.getBattery` |
| Generic Sensor API | 有 | 通常没有 | `!!window.Accelerometer` (已有但未充分利用) |
| Vibration API | 必有 | 通常没有 | `!!navigator.vibrate` |
| navigator.maxTouchPoints | 5-10 | 0-5 (可伪装) | 已有 |
| USB/Bluetooth/NFC API | 有 | 没有 | `!!navigator.usb` 等 |
| Screen orientation lock | 支持 | 通常不支持 | `screen.orientation.lock` 异常 |

### 3.4 缺少的高价值检测

| 检测方向 | 原理 | 难度 |
|---------|------|------|
| WebGL 参数指纹 | `MAX_TEXTURE_SIZE`, `MAX_RENDERBUFFER_SIZE` 等参数组合区分真实/模拟 GPU | 低 |
| 触摸事件真实性 | 监听触摸事件的时间间隔和压力值, 模拟器触摸事件缺乏真实噪声 | 中 |
| 陀螺仪数据模式 | 真实陀螺仪有微小噪声, 模拟器返回固定值或全零 | 中 |
| 并行度测试 | `SharedArrayBuffer` + `Atomics` 测试真实并行能力, 翻译层表现不同 | 高 |
| 字体渲染像素对比 | 同一字体在不同 GPU 上的亚像素渲染不同 | 中 |

---

## 五、代码质量问题

### 4.1 DOM 操作

**`innerHTML +=` 反模式** (138-139行):
```javascript
c.innerHTML += `<div class="cat">...`;
c.innerHTML += `<div class="cd" ...`;
```
每次 `+=` 重新解析整个 DOM 子树, 40+ 卡片导致 O(n^2) 解析。应累积字符串后一次赋值。

### 4.2 哈希碰撞风险

`H(name)` 函数 (109行) 用简易哈希生成 DOM id:
```javascript
function H(n){let h=0;for(let i=0;i<n.length;i++)h=((h<<5)-h+n.charCodeAt(i))|0;return Math.abs(h)}
```
40+ 中文检测名有碰撞概率, 碰撞后 `renderCard` 更新错误卡片。应使用索引或 slugify。

### 4.3 端口扫描串行

(481行) 8 个端口逐个扫描, 每个 1.2s 超时, 最坏 9.6 秒:
```javascript
for(const p of ports) if(await scan(p)) open.push(p);
```
应改为 `Promise.all` 并行, 降至 1.2s。

### 4.4 变量命名

大量单字母/双字母变量降低可读性:
```
cv, gl, pf, d, b, h, s, r, m, t, x, v, f, e, n, p, a, el, ck, fn, ws, ac, osc, comp, buf, loc, pg, vs, fs
```

### 4.5 CSS 压缩混写

样式直接压缩写在 `<style>` 中 (7-47行), 无法单独维护。关键选择器如 `.cd-b`, `.b-p`, `.b-r` 含义不明。

### 4.6 外部 API 单点依赖

(454行) `ipapi.co` 无备选方案, 被墙/限流后数据中心 IP 检测完全失效。且为 HTTP 明文调用 (实际是 HTTPS, 可接受), 无鉴权。

---

## 六、安全性

### 5.1 无严重漏洞

- 无用户输入直接注入 DOM 的路径 (XSS)
- 无后端交互, 无 SQL 注入面
- 导出 JSON 为纯客户端操作

### 5.2 检测可被主动绕过

攻击者可针对性绕过:

1. **Hook `report` 函数**: 在页面加载前注入脚本覆盖 `report`, 所有检测返回 `pass`
2. **伪造 API 返回值**: 在模拟器中 Hook `navigator.getBattery`, `Accelerometer` 等返回真实值
3. **禁用 WebSocket**: 阻止端口扫描, 所有端口返回 closed = `pass`
4. **时序攻击**: `setTimeout` 超时后默认走 `info` 路径, 可通过阻塞触发超时

**缓解建议**: 关键函数的完整性校验应在执行前而非执行中进行; 超时应视为异常而非正常。

### 5.3 隐私考虑

- Geolocation 请求会弹授权框, 可能引起用户警觉
- `ipapi.co` 会将用户真实 IP 发送到第三方
- WebRTC IP 泄露检测本身会暴露内网 IP

---

## 七、架构建议

### 6.1 短期 (不改架构)

1. **修正评分模型**: `info` 按设备类型加权参与; 增加一票否决机制
2. **收紧 6 项判定标准**: 电池/传感器/屏幕/Headless 的 `info` → `risk`
3. **消除永远 pass**: 无判定能力的检测改为 `info` 或移除
4. **并行化端口扫描**: `for...await` → `Promise.all`
5. **阶段 1 增加模拟器信号**: Battery/Sensor/Vibration 存在性预检

预期效果: 模拟器分数从 70-80 降至 20-40。

### 6.2 中期 (小改架构)

1. **检测项分级**: critical / major / minor, 不同级别不同权重
2. **建立指纹基线库**: 收集已知模拟器的 Canvas/Audio/Math/WebGL 哈希, 做黑名单匹配
3. **增加行为检测**: 触摸事件真实性、陀螺仪噪声分析
4. **多 IP 服务备选**: ipapi.co 失败时 fallback 到 ip-api.com 等

### 6.3 长期 (重构)

1. **模块化拆分**: 检测项按类别拆文件, 构建时合并
2. **自动化测试**: 用 Puppeteer 模拟各种设备/模拟器环境, 回归测试
3. **服务端验证**: 客户端采集 + 服务端决策, 防止客户端篡改
4. **对抗升级机制**: 检测指纹版本化, 模拟器更新后能快速迭代检测规则

---

## 八、亮点

不全是问题, 以下设计值得保留:

1. **x86 Android 检测思路正确** — "真机 Android 都是 ARM" 是坚实的底层逻辑
2. **iOS 豁免设计得当** — 正确识别 iOS 不提供 Battery/Connection/Accelerometer
3. **UA 交叉验证** — UA + Platform + GPU 三方校验, 单项伪造无法通过
4. **零依赖单文件** — 适合嵌入任何 H5 场景, 无供应链风险
5. **原生函数完整性检测** — `[native code]` 校验思路正确
6. **两阶段架构** — 先画像后针对性检测的思路是对的
7. **JSON 导出** — 便于收集数据和分析

---

## 九、Headless Chrome 实测补充

PROJECT_REVIEW.md 中记录了 Headless Chrome 实测结果:

- 43 项检测全部完成
- 41 个通过, 0 个风险, 2 个信息
- **最终评分 100 分**

Headless Chrome 是最基础的自动化工具, 反作弊检测器对它评 100 分, 比模拟器 70-80 分更严重。说明当前检测体系对自动化环境几乎没有识别能力。

---

## 十、结论

项目作为 H5 反作弊检测 PoC 演示有价值: 界面完整、信号采集广度好、部署简单、零依赖。

**但核心功能失效**: 模拟器得分 70-80, Headless Chrome 得分 100。反作弊检测器放过了它最应该拦截的目标。

问题不在检测项数量 (40+ 已足够), 而在:

1. **评分模型** — `info` 逃逸 + 等权平均 + 无一票否决, 导致风险被稀释
2. **判定标准** — 6 项永远 pass 送分, 关键检测门槛过宽
3. **代码缺陷** — `DeviceMotionEvent` 未声明崩溃、WebRTC 超时漏报、DOM XSS
4. **工程化缺失** — 无测试、无设备样本回归、文档与代码不一致

**建议定位为 PoC**, 在修复评分模型 + 判定标准 + 代码 bug 后再考虑生产使用。最小修复路径: ~100 行代码改动, 不增加新检测项, 可将模拟器/Headless 得分降至 20-40。
