# Anti-Cheat Detector H5 v2

基于 WPK (WePoker) 逆向分析的反作弊检测方案 H5 复现。

**定位**: PoC / 检测能力演示。不适合直接用于生产环境的拦截/放行决策。

## 架构

两阶段检测：

1. **设备画像** — 多信号交叉分类（iPhone / Android / PC / 模拟器 / 云真机）
   - 硬证据短路：x86 Android、软件渲染 GPU 等确定性信号直接判定 100% 模拟器
   - 无硬证据时走加权评分
2. **针对性检测** — 基于设备类型选择规则集，避免跨平台误判

## 检测项

| 类别 | 检测项 | 状态类型 |
|------|--------|---------|
| 环境验证 | WebGL 渲染器、GPU/平台一致性、电池、传感器、DeviceMotion、硬件信息、屏幕、媒体设备 | pass/risk |
| 指纹采集 | Canvas、AudioContext、字体、UA/Platform、时区语言、WebGL 参数 | info(采集) |
| 网络与位置 | Geolocation、WebRTC IP、数据中心 IP、Proxy/VPN | pass/risk |
| 工具检测 | 端口扫描(WPK 7125/7126 + Frida + ADB)、自动化框架、Headless 深度、DevTools、浏览器版本 | pass/risk |
| 完整性 | 原生函数、Console 劫持、iframe、Service Worker | pass/risk |
| 行为分析 | WebView、多开检测、属性一致性(Lie Detection)、Storage、页面生命周期、FP/FCP/LCP | pass/risk/info |
| 深度指纹 | Math 精度、WebGL 渲染、AudioContext 延迟、屏幕物理尺寸、Vibration、Performance 计时、平台架构、WebGL 扩展、时间源精度 | pass/risk/info |
| 综合判定 | 设备类型概率、云真机判定 | pass/risk |

## 评分

- `pass`: 检测通过，计入安全分
- `risk`: 检测到风险，计入风险分
- `info`: 采集/跳过/不可用，**不影响评分**
- 综合评分 = `100 × pass / (pass + risk)`
- 最终判定结合设备画像：画像=模拟器时无论通过率多高都标风险

## 使用

需通过 HTTP 服务运行（`file://` 下部分检测受 CORS 限制）：

```bash
# 方式1: Python
python -m http.server 8000

# 方式2: Node.js
npx serve .

# 方式3: GitHub Pages
# https://linusteller.github.io/anticheat-h5/
```

## 局限性

- 纯采集型指纹（Canvas/Audio/Math/WebGL渲染）没有基线样本库，无法做真正的判定
- MuMu 等 ARM 模拟器在浏览器中没有硬矛盾信号（armv81 + Adreno 640 与真机一致），当前检测能力有限
- 无自动化测试、无 CI、无多设备回归样本
- 端口扫描会在浏览器控制台产生连接失败日志（预期行为）

## 技术来源

- WPK APK v5.8.19 逆向分析（AppDome v5.16 + JS 层解密）
- WPK H5 实测分析（CDP Runtime.evaluate 穷举 639+ 函数）
- FingerprintJS BotD 开源检测方法
- CreepJS Lie Detection 思想
