# 04 — 去吉利专用化清单

DiPlay-Geely（xikai6282）的适配包含大量只在**吉利 H52 固件**上有效的代码。移植到通用改装大屏时，
需要逐项甄别：哪些删、哪些留、哪些有隐藏风险。

## 甄别原则

1. **依赖吉利固件服务**（usbprotect / ECARX Binder / BYD SomeIp）→ 在其他车机上必然失败或永不触发
2. **失败路径有副作用**（如 fallback 硬编码吉利默认值）→ 必须移除，不只是「反正不会执行」
3. **纯标准 Android API 的性能优化** → 保留，跨机型有效

## 移除清单

### `H52NoRootBridge.java`（130 行 → 40 行）

| 函数 | 原实现 | 处置 |
|---|---|---|
| `execSystem(cmd)` | 反射取吉利固件 `usbprotect` 服务（Neusoft，transaction 0x16），**免 root 执行系统 shell** | 删除。其他车机无此服务，永远失败 |
| `switchUsbConfiguration(target, path)` | 经 usbprotect 写 sysfs 切 USB 配置 | 删除，调用点改为直接抛标准异常 |
| `readHostapdChannel(observed, configured)` | usbprotect 读 hostapd.conf；**失败时 fallback 硬编码信道 36（H52 5GHz 默认）** | 改纯参数逻辑。⚠️ **fallback 36 是隐藏炸弹**：非吉利车机热点信道若非 36，会被强制错信道 |
| `boostThreadPriority()` | `Process.setThreadPriority(THREAD_PRIORITY_URGENT_DISPLAY)` | **保留**（纯标准 API，P16 流畅度核心） |
| `tuneVideoSocket(s)` | `TCP_NODELAY` + 512KB 接收缓冲 | **保留**（同上） |

### `H52WirelessDiagnostics.java`

`channel()` 内的 usbprotect/hostapd 读取路径删除，只留 `observed > 0 ? observed : configured` 纯逻辑。

### `H52UsbConfigurationFix.java`

`select()` 的 Apple sysfs + usbprotect fallback 分支删除（`LegacyUsbNative.configure` 失败时
直接走标准错误路径）。

### `GeelyBluetoothDiagnostics.kt`（474 行）

吉利 ECARX ANW 蓝牙 Binder 只读诊断。**设置页 UI 入口全部移除**（连接区「吉利蓝牙连接」开关、
诊断区 opt-in 开关 + 厂商检测按钮、H52 音频路由区块），`showBluetoothStackStatus` 固定走
纯 Android 蓝牙状态显示。

### 保留不动的（有自然门控）

| 组件 | 门控机制 |
|---|---|
| `BydHudBridge`（BYD SomeIp HUD） | bind 失败仅记日志不崩溃；服务不存在自然跳过 |
| `CarPlayHostActivity` 的 H52 音频路由块 | `geelyAudioCapabilities != null` 才渲染；非吉利车检测返回 null |
| `AnwBluetoothBackend` / `separateGeelyFocus` | `geelyBluetoothEnabled` / `separateGeelyFocus` 默认 false |

## 审计方法（防漏网）

- 源码级：`grep -rniE "geely|h52|usbprotect"` 全量过一遍 main 源集，逐条判定
- **dex 级**：`strings classes*.dex | grep -cE "usbprotect|geely_bt_connection_title"` = 0 才算干净
- ⚠️ 注意 Geely 源码树是 **Kotlin + Java 混合**：只搜 `*.kt` 会漏掉 `.java` 文件
  （`H52NoRootBridge.java` 就是这么漏掉的）
- ⚠️ 设置页 UI 可能分散在**多个 Activity**（`DiPlayActivity` 清干净后，`CarPlayHostActivity`
  还有一块——但有 capabilities 门控，非吉利车不渲染）

## 版本可区分性教训

8/9/10 号包 versionName 全部相同（`0.2.13-H52-P16`, code 30）→ 装机后**无法从诊断报告区分版本**。
从 11 号起：`andrewN` 后缀 + versionCode 递增（31, 32, ...），报告第 1-3 行立即可验。
