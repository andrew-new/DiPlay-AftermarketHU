# 02 — 音频修复：根因在协商层，不在播放层

症状：CarPlay 连接成功、视频正常，点播放 → **iPhone 端瞬间自动暂停**，车机无任何声音。

## 诊断链路（可复用的方法论）

```
症状：点播放，iPhone 瞬间暂停
  ↓ 导出诊断报告（App 内置）
证据 1：全程零音频日志 —— grep -iE "audio|opus|aac|pcm" 无任何 Audio: ready 行
  ↓
证据 2：SETUP 只协商了两种流
        streams=[{type=130, ...}]   ← 数据隧道
        streams=[{type=110, ...}]   ← 视频
        type=100(主音频)/101/102(AAC媒体) 一个都没有，两次会话完全一致
  ↓ 代码审查
排除：服务端 AirPlayInfoPlist 声明了 9 种 audioFormats（含 100/101/102），
      AirPlaySession.handleStreams() 的 when(type) 分派正确
  ↓ 结论
根因：**协商声明有缺陷 → iPhone 判定接收端不支持音频 → 根本不发音频流**
```

## 为什么是协商层

DiPlay 0.2.10/0.2.7 的 `audioFormats()` 声明中，**Opus 被清零**（API18 无 Opus 硬解时的
「协商尝试」）。iPhone 侧要求接收端声明可用的压缩音频格式；声明里只剩 PCM 时，iOS 27 判定
该接收端不具备音频能力，直接不建立 `type=100` 流——于是播放端（AudioTrack）永远等不到数据。

**DiPlay-Geely P21 的修复原话**：

> 撤回 API18 清零 Opus 的协商尝试，**恢复手机建立无线音频流**。
> API21 以下以 Concentus 软件解码导航 Opus，保留 AAC-LC 音乐及既有媒体/导航音频通道。

即：**保留 Opus 声明**（软解兜底），让 iPhone 愿意发流。P21 在吉利博瑞 H52 实车确认
音乐与导航都有声——协议层修复跨车型通用。

## 7 号包的教训（反面教材）

我们最初的修复方向：`AudioTrack.getMinBufferSize` 失败 → 回退 44.1kHz 重试、
解码器配置失败 → 换采样率重试。**这些代码全部正确，但一次都没执行过**——因为音频流
从未建立。教训：

> 修复前先确认「问题发生在链路的哪一段」。日志里连一行该层的日志都没有，说明上游根本没到这。

## 实操要点

- 音频流建立的判据：SETUP response 中出现 `type=100`
- 接收端声明（`/info` 的 `audioFormats`）决定 iPhone 是否发起流
- 老平台无 Opus 硬解 ≠ 不能声明 Opus —— Concentus 软解（纯 Java/Kotlin Opus 实现）兜底
- AAC-LC（音乐）与 Opus（导航语音）双通道可并存

## 对标

- 官方 [DiPlay #456](https://github.com/shihabal3amri/DiPlay/pull/456)（2026-10-08）：低延迟解码，
  但门槛 Android 11+
- [DiPlay-Geely P21](https://github.com/xikai6282/DiPlay-Geely-Android43)：本文所述协商修复的实现与实车验证
