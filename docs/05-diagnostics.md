# 05 — 诊断报告字段解读指南

DiPlay 的 `Settings → Diagnostics → Save diagnostic report`（各分支入口名略有差异）生成的
`.txt` 报告是定位问题的唯一数据源（改装车机 adb 通常不可用）。本文按字段速查。

## 报告头部（版本与环境）

```
车机无线 CarPlay 0.2.13-H52-P21.andrew3     ← 版本（用于区分改装包）
Package=com.shihab.diplay.repair1 Edition=universal
Android=6.0 API=23                          ← 运行时系统版本（注意：可能与设置页显示不一致，见 07）
Model=evb3561sv_w_65_m0 manufacturer=alps   ← Build.MODEL（报告以此为准，不是设置页文案）
Profile=manual hotspot, H.264, 100 percent, 30 fps, playback and touch
```

> `Profile` 里的 percent 就是画质档；排查卡顿先确认被测时的档位。

## 连接协商段（音频无声先看这里）

```
airplay SETUP keys=[streams]
airplay SETUP response streams=[{type=130, streamID=1, dataPort=...}]   ← 数据隧道
airplay SETUP response streams=[{type=110, dataPort=...}]              ← 视频
```

| 流类型 | 含义 |
|---|---|
| 100 / 101 / 102 | 主音频 / 备用音频 / AAC 媒体 |
| 110 / 111 | 主屏 / 副屏视频 |
| 130 | iAP2 数据隧道 |

**无声问题的判据**：响应里没有 `type=100/101/102` → iPhone 根本没发起音频流 → 查协商层
（见 [02-audio-fix](02-audio-fix.md)），不要去查 AudioTrack。

## video stats 行（卡顿问题先看这里）

```
video stats rx=28.5fps shown=28.0fps maxGap=180ms kbps=4200 recoveries=1
            touch2frame avg=95ms max=210ms n=40 touchSendMax=12ms decodeOnly=6
```

| 字段 | 含义 | 卡顿判读 |
|---|---|---|
| `rx` | 接收帧率 | 明显 <30fps → 无线链路喂不饱（带宽/干扰/距离） |
| `shown` | 实际渲染帧率 | rx 正常但 shown 低 → 解码或跳帧异常 |
| `maxGap` | 相邻帧到达最大间隔 | 大幅波动（>200ms）→ 无线抖动 |
| `kbps` | 实际码率 | 与画质档交叉验证 |
| `recoveries` | 5 秒内解码器恢复次数 | >2 持续出现 → 积压策略仍在触发，查触发路径 |
| `touch2frame avg/max` | **触摸到画面响应的实测延迟** | avg>150ms → 延迟是「卡感」主因，与帧率无关 |
| `decodeOnly` | 解码了但没显示的帧数（catch-up 跳帧） | >0 且 recoveries 低 = 跳帧机制在工作 |

## 定位决策树

```
滑动卡顿
├─ rx < 30fps            → 无线链路（频段/信道/距离/干扰）
├─ rx≈30, shown 低       → 解码吞吐（降画质档）或跳帧过多（查 recoveries）
├─ touch2frame avg>150ms → 端到端延迟（低延迟解码 / 无线 RTT）
├─ recoveries 频发       → 积压策略触发路径未除尽（报告发社区，带 Profile 行）
└─ 全部正常仍卡          → 帧率正常但跟手性差 = 投屏方案固有 RTT，属硬件/架构上限
```

## 导出路径速记

- Android 9-：走文件选择器（部分改装机裁掉 DocumentsUI 会失败——需改包硬编码路径）
- Android 10+：自动存 `Downloads/DiPlay/`
- 导报告前**先复现问题**（滑动几下）——stats 是 5 秒滚动窗口，导早了数据就溜走了
