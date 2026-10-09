# DiPlay-AftermarketHU

[DiPlay](https://github.com/shihabal3amri/DiPlay)（GPL-3.0，CarPlay receiver for Android head units）在**通用改装安卓大屏**上的适配实战笔记。

> 目标硬件：XY AUTO 改装大屏（MTK MT6737 / A53 四核 / 2GB RAM / 1024×600，诊断报告报 `alps evb3561sv_w_65_m0`，API 23）。
> iPhone：iOS 27.0.1。连接方式：车机热点（manual hotspot），非 Wi-Fi Direct。

## 当前状态

| 功能 | 状态 | 包 |
|---|---|---|
| 无线连接（热点自动恢复） | ✅ 稳定 | 基于 Geely P21 |
| 音乐/导航音频 | ✅ **有声**（协商层修复） | andrew2 起 |
| 触控 | ✅ 正常 | — |
| 视频 | ⚠️ 有画面；软件侧优化到位（实测 333 窗口零恢复循环），残留卡顿定位为**无线链路抖动**（2.4GHz 物理特性）。下一步验证有线 USB | andrew3 加低延迟解码；[实测结论](docs/03-lag-analysis.md) |

## 免责声明

- 本仓库是**适配笔记与文档**，不是 Apple 认证产品，也不分发认证材料。
- DiPlay 相关 APK 使用社区从 Carlinkit 公开固件提取的**实验性 MFi 身份**，苹果认证适用性未解决。
- 文档中不包含任何私有密钥、签名文件或认证资产。

## 文档导航

| 文档 | 内容 |
|---|---|
| [01 — 攻关历程](docs/01-journey.md) | 三天、八个包、两条技术路线的完整复盘 |
| [02 — 音频修复](docs/02-audio-fix.md) | 根因：协商层（audioFormats）而非播放层；Opus 声明恢复 |
| [03 — 卡顿分析](docs/03-lag-analysis.md) | 三层瓶颈定位 + VideoCatchUp 移植实录（对标官方 #347/#456） |
| [04 — 去吉利专用化](docs/04-degeely.md) | usbprotect 免 root shell、吉利蓝牙 Binder、H52 音频路由的移除清单 |
| [05 — 诊断报告解读](docs/05-diagnostics.md) | `video stats` 字段速查（touch2frame / rx / shown / decodeOnly / recoveries） |
| [06 — 构建指南](docs/06-build.md) | JDK 25 + SDK 37 + NDK r28.2 + MFi 资产注入的环境与命令 |
| [07 — 兼容性边界](docs/07-compatibility.md) | Android 版本之谜（6.0 报告 vs 8.1 设置）、热点频段、已知限制 |

## 相关社区仓库

- [shihabal3amri/DiPlay](https://github.com/shihabal3amri/DiPlay) — 官方上游（7003★）
- [xikai6282/DiPlay-Geely-Android43](https://github.com/xikai6282/DiPlay-Geely-Android43) — 吉利博瑞 H52 / API18 适配（本包音频修复的来源，P21 实车验证有声）
- [qingshen027/DiPlay-H6](https://github.com/qingshen027/DiPlay-H6) — 哈弗 H6 / 德赛西威 Android 8.1
- [Hanxu4131/BYD-CarPlay](https://github.com/Hanxu4131/BYD-CarPlay) — 比亚迪唐 DM-i 适配（官方 #347 积压修复贡献者）

## License

文档内容以 [MIT](LICENSE) 提供。若后续发布二进制或源码补丁，将遵循上游 DiPlay 的 GPL-3.0 并附对应源码。
