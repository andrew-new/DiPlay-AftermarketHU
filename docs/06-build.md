# 06 — 构建指南

以 DiPlay-Geely P21 源码（`DiPlay-H52-P21-source.zip`）为基线，产出 `andrewN` 系列包。

## 环境

| 组件 | 版本 | 路径示例 |
|---|---|---|
| JDK | 25 | `toolchain/jdk-25.jdk` |
| Android SDK | compileSdk 37 | `toolchain/sdk` |
| NDK | **r28c（28.2.13676358）** | `toolchain/ndk/android-ndk-r28c` |
| Gradle | wrapper（`./gradlew`） | — |

**NDK 版本陷阱**：`shared/build.gradle` 写死 `ndkVersion = '28.2.13676358'`，但 SDK 目录里的
NDK 可能是旧版（如 `ndk/25.2.9519653`）。Gradle 按**目录名**查找，需要：

```sh
ln -sfn $TOOLCHAIN/ndk/android-ndk-r28c $TOOLCHAIN/sdk/ndk/28.2.13676358
```

## 认证资产（不入仓库）

DiPlay 构建需要 MFi 身份文件（`identity.pk8` + `certificate.p7b`，社区从 Carlinkit 公开固件
提取的实验性身份）。**这些文件绝不进 git**，通过环境变量注入：

```sh
export DIPLAY_AUTH_ASSETS_DIR=/path/to/mfi-assets   # 目录下须有 offline-mfi/{identity.pk8,certificate.p7b}
```

不带此变量时 `assembleDebug` 为 source-only 构建（无认证资产，无法通过 CarPlay 认证）。
README 原话：*"Standalone car builds require DIPLAY_AUTH_ASSETS_DIR; assembleDebug alone is source-only."*

## Release 签名（可选）

```sh
export ANDROID_KEYSTORE_PATH=/path/to/keystore
export ANDROID_KEYSTORE_PASSWORD=...
export ANDROID_KEY_ALIAS=...
export ANDROID_KEY_PASSWORD=...
./gradlew :mobile:assembleRelease
```

## 构建命令

```sh
./gradlew :mobile:assembleDebug            # debug variant（Geely 树：包名带 .repair1 后缀）
./gradlew :mobile:assembleUniversalRelease # 上游 NineThree 树的变体名不同
./gradlew :shared:testDebugUnitTest :common:testDebugUnitTest   # 617 项单元测试
```

## 本树相对 P21 原始源码的改动清单

| 文件 | 改动 |
|---|---|
| `shared/.../media/VideoCatchUp.kt` | **新增**（127 行，零依赖，自 H3 适配移植） |
| `shared/.../media/VideoOutputDrain.kt` | **新增**（47 行，零依赖，同上） |
| `shared/.../media/AndroidMediaSink.kt` | catch-up 五处接线 + 低延迟键门槛下探（API 23-29）+ `KEY_PRIORITY=0` |
| `shared/.../media/VideoStats.kt` | `onDecodeOnly` 跳帧计数 |
| `shared/.../compat/H52NoRootBridge.java` | 通用化：删 usbprotect shell/USB 写/hostapd 读取（含信道 36 fallback） |
| `shared/.../compat/H52WirelessDiagnostics.java` | channel() 纯参数逻辑（顺带修一个编译错误：原调用了不存在的无参重载） |
| `shared/.../transport/H52UsbConfigurationFix.java` | 删 usbprotect fallback 分支 |
| `common/.../DiPlayActivity.kt` | 移除吉利蓝牙 probe 块（-114 行）与四处吉利设置 UI |

## 构建排障备忘

| 报错 | 原因 | 解法 |
|---|---|---|
| `Unresolved reference 'H52NoRootBridge'` | 只搜了 `*.kt`，实为 `.java` 文件 | 全文件类型搜索 |
| `Unresolved reference: readHostapdChannel()` | Geely source.zip 与 APK 不一致，调用了不存在的无参重载 | 改为两参调用或纯参数逻辑 |
| `Address already in use`（adb server） | 本机 5037 被占 | `kill` 旧 server 或无视 |
| fine-grained PAT 建仓 `Resource not accessible` | token 缺 Administration/Contents 写权限 | GitHub token 编辑页补权限，或网页手动建仓 |
| BSD/macOS `sed -i ''` 在 zsh 中失效 | 参数解析差异 | 用 Python 做行级编辑更稳 |
