# Anti-Cheat Detector H5

基于 WPK (WePoker) 逆向分析的反作弊检测方案，H5 网页端复现。

单个 HTML 文件，零依赖，双击即开。

## 检测项 (20+)

| 类别 | 检测项 |
|------|--------|
| 模拟器检测 | WebGL 渲染器、电池状态、传感器、DeviceMotion、屏幕特征、Performance 精度、硬件信息 |
| 设备指纹 | Canvas 指纹、AudioContext 指纹、UA/Platform 一致性、时区语言、WebGL 参数指纹 |
| 位置与网络 | Geolocation、WebRTC IP 泄露、网络连接信息 |
| 工具检测 | 端口扫描 (7125/7126/Frida/ADB)、自动化框架、DevTools、Proxy/VPN |
| 注入检测 | 触摸事件来源 (isTrusted)、按键事件注入 |
| 完整性检测 | 原生函数完整性、toString 防护、iframe 沙盒 |
| 行为分析 | 插件数量、navigator 属性异常、Storage 可用性 |

## 使用

浏览器打开 `index.html`，自动运行所有检测，显示综合评分和每项结果。

支持导出 JSON 报告。

## 技术来源

基于对 WPK 5.8.19 APK 的完整逆向分析，还原了以下机制：
- `FindEmulator.java` 模拟器特征扫描
- `MotionEventsManager` 触摸事件 Hook
- `KeyEventsManager.onKeyPreImeAntiKeyInjection()` 按键注入检测
- `BatteryChangedReceiver` 电池状态监控
- JS 层本地端口扫描 (7125/7126)
- `DeviceUuidFactory` 设备指纹生成
- AppDome 原生函数完整性校验
