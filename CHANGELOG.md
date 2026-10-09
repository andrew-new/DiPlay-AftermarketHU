# CHANGELOG

All notable changes to the andrewN build series. Based on
[DiPlay-Geely P21](https://github.com/xikai6282/DiPlay-Geely-Android43) (source `0.2.13-H52-P16`, code 30).

## [0.2.13-H52-P21.andrew3] — 2026-10-08 (code 32)

### Added
- **Low-latency decode for API 23-29**: `tryConfigure` gains the string key `low-latency=1`
  (the existing branch was gated to API 30+ and never ran on older units). Old MediaTek OMX
  decoders commonly buffer 2-3 frames internally; unrecognized keys are ignored, and a
  `BAD_VALUE` falls through the existing `tuned=false` retry chain.
- `MediaFormat.KEY_PRIORITY = 0` (realtime decode priority, API 23+).

### Changed
- Version naming now distinguishes this build series (`andrewN` suffix, versionCode increments) —
  earlier builds 8/9/10 were indistinguishable in diagnostic reports.

### Verified
- 617 unit tests pass (shared + common).

## [0.2.13-H52-P21.andrew2] — 2026-10-08 (code 31)

### Removed
- All Geely-specific settings UI (`DiPlayActivity`): Bluetooth-connection toggle, vendor
  diagnostics opt-in + vendor check button, H52 audio routing block, `geelyAudioControl`.
  Generic toggles (audio focus, advanced audio channel mapping, diagnostic export) preserved.
- `GeelyBluetoothDiagnostics` probe invocation in `showBluetoothStackStatus` (fixed to the
  pure-Android Bluetooth status path).

### Changed
- Distinct version string (fixes the inability to tell builds apart in reports).

## [0.2.13-H52-P16-andrew.1] — 2026-10-08 (code 30, "10号")

### Added
- **VideoCatchUp** (ported from the H3 adaptation, byte-identical to its field-tested original):
  decode catch-up with stale-output skipping. Backlog threshold 250 ms → 750 ms;
  destructive flush only for sustained backlog.
- **VideoOutputDrain**: hold-one-buffer drain providing `newerTimestampUs` for render decisions.
- `VideoStats.decodeOnly` counter (5 s report line).

### Fixed
- Low-latency decode keys actually reached old units (see andrew3 — initial port still gated to API 30).
- `H52WirelessDiagnostics.channel()` called a non-existent no-arg overload (source-zip drift);
  replaced with pure parameter logic.

### Removed
- Geely `usbprotect` no-root shell (`execSystem`), `switchUsbConfiguration`,
  `readHostapdChannel` (including the **hard-coded channel-36 fallback** — wrong for other head units).
- `H52NoRootBridge` rebuilt as a generic performance helper (`boostThreadPriority`,
  `tuneVideoSocket`: TCP_NODELAY + 512 KB receive buffer — the P16 pieces the P21 APK missed
  due to its stale `classes2` packaging).

### Verified
- 617 unit tests pass; dex audit: new paths present, `usbprotect` 0 hits, MFi assets bundled.
