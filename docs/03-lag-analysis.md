# 03 — 卡顿分析：三层瓶颈与 VideoCatchUp 移植

症状：CarPlay 页面滑动切换视图卡顿。60% 画质档（解码量 -64%）依然卡。

## 瓶颈分层模型

```
iPhone 渲染 → 无线传输（热点带宽） → 车机解码（MediaCodec） → 车机渲染（Surface 合成）
                                                    ↑ touch2frame 全链路延迟
```

滑动是链路最恶劣场景：全屏重绘 → H.264 码率瞬间 10-20Mbps → 哪层跟不上哪层卡。

## 排除法（实测数据）

| 假设 | 实验 | 结果 |
|---|---|---|
| 解码吞吐不足 | 60% 档（614×360，解码量 -64%）实测 | **依然卡 → 排除** |
| 反复 flush 循环 | VideoCatchUp（见下）生效后 | 仍卡 → 非主因 |
| 无线链路抖动 | 待 `touch2frame`/`rx` 实测数据 | **当前主嫌疑** |
| 端到端延迟 | 待 `touch2frame avg/max` 实测 | **当前主嫌疑** |

## 核心机制：Geely P21 的 250ms 积压恶性循环

```
滑动 → UI 线程抢占 CPU → 解码变慢 → 队列积压超 250ms
  → discardFrames + recover(flush 解码器) → 请求 iPhone 重发关键帧（iAP2 往返）
  → 画面停顿 → 恢复后又积压 → 再 flush → 循环
```

DiPlay-Geely 源码（P21）：

```kotlin
is VideoJob.Frame -> {
    if (System.nanoTime() - job.receivedNs > MAX_FRAME_AGE_NS) {  // 250ms
        queue.discardFrames()
        recover("video backlog exceeded 250 ms")
    } else feed(job.nalus)
}
```

**5 号包（H3 适配）作者实车报告原话**：「19 次恢复，全部来自旧 250ms 积压阈值」。

## 修复：VideoCatchUp（解码追赶 + 跳过旧输出）

原理：**解码全部参考帧（保持参考链完整），只跳过确认过期的输出；极端积压（750ms）才 flush**。

两个自包含组件（零依赖，直接移植）：

- `VideoCatchUp`（127 行）：环形表记录每帧到达时间。
  - 输出时：帧龄 >250ms 且已有更新帧到达 → `releaseOutputBuffer(index, render=false)`（解码继续）
  - **最后一帧永远渲染**——静止画面不黑屏
  - 输入积压 >750ms → needsResync → 才走 discard + recover
- `VideoOutputDrain`（47 行）：drain 循环「持有一帧」换得下一帧 PTS 作为
  `newerTimestampUs` 传给 shouldRender；finally 保证 EOS/退出时缓冲恰好归还一次。

接线五处（`AndroidMediaSink.kt`）：

1. worker Frame 分支：阈值 250→750ms（`VideoCatchUp.needsResync`）
2. `feed(nalus)` → `feed(nalus, receivedNs)`；输入 PTS 用 `catchUp.onQueued(receivedNs)`
   单调时间戳（输出端才能按到达时间判帧龄）
3. `drainOutput` 重写为 `VideoOutputDrain.drain` 回调式 + `catchUp.shouldRender(...)`
4. `recover()` 加 `catchUp.clear()`
5. `VideoStats` 加 `decodeOnly` 跳帧计数进 5 秒报告（**验证机制是否生效就看它**）

> PTS 被单调时间戳替换是安全的：视频独立通道、无音画对齐逻辑，5 号作者实车同做法。

## 验证与验收

诊断报告 `video stats` 行新增 `decodeOnly=N`：
- `decodeOnly > 0` 且 `recoveries` 下降 → 机制生效
- `recoveries` 仍频繁（积压常超 750ms）→ 瓶颈不是积压策略，是解码本身太慢 → 降画质档

## 对标官方（互相印证）

| | 实现 | 状态 |
|---|---|---|
| 本文（VideoCatchUp，来源 H3 适配） | 250ms 跳帧追赶 / 750ms flush | 已实车（本机） |
| [官方 #347](https://github.com/shihabal3amri/DiPlay/pull/347)（BYD 适配者寒叙） | 宽限间隔 + 积压硬上限，唐 DM-i 实车验证 | PR open 未合并 |

两个互不相识的作者在同一问题上收敛到同一解法。

## 遗留与后续

- 低延迟解码：`tryConfigure` 的 `KEY_LOW_LATENCY` 被 `SDK_INT >= API 30` 门槛挡住，
  老 MTK OMX 常默认内部攒 2-3 帧（66-100ms）。下探到 API 23-29（字符串键 `low-latency=1`，
  BAD_VALUE 走 tuned=false 回退链）+ `KEY_PRIORITY=0`。vendor 依赖，效果待实测。
- 官方 0.2.14 的 Smooth video（自适应延迟调帧）**会加触摸响应延迟**，对触控镜像是反向优化，不建议开。
- 若 `touch2frame > 150ms` 且低延迟无效 → 无线链路（2.4GHz 抖动）是主因，软件层接近天花板。
