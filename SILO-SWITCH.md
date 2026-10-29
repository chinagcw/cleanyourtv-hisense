# SILO 广播切换 HDMI 信号源（海信 VIDAA）

本文档记录海信 VIDAA 电视（Android 8.0 / SDK 26 / 型号 NOVA72671）上，通过 SILO 广播**无浮层、免 root** 直接切换 HDMI 信号源的技术原理与验证结果。这是 CleanYourTV 海信定制版的核心。

## 问题背景

海信 VIDAA 的输入源切换面板（信号源浮层）由"聚好看"（`com.jamdeo.tv.vod`）承载。精简电视时如果停用/禁用聚好看，遥控器信号源键（`KEYCODE_TV_INPUT`，值 178）将无法呼出信号源浮层，HDMI 输入无法切换。

同时，该电视的 `TvInputManager` 服务完全为空：

```
$ dumpsys tv_input
  inputMap: inputId -> TvInputState     # 空
  packageSet:                            # 空
  mHardwareList:                         # 空
  mHdmiDeviceList:                       # 空
  mInputMap: inputId -> inputInfo        # 空
```

系统未注册任何标准 TV 输入服务，因此依赖 `TvInputManager.getTvInputList()` 的通用输入切换方式（Android TV 标准做法）在此电视上不可用。

## SILO 广播

海信使用私有广播 `com.jamdeo.tv.policy.SILO_SWITCH_EVENT` 切换输入源，通过 int extra `silo_input_src_id` 指定目标源。

### 源 ID 对照

| `silo_input_src_id` | 信号源 |
|---|---|
| `1281` | **HDMI 1** |
| `1282` | **HDMI 2 / ARC** |
| `513`、`600` | 其他（AV/分量等，视机型而定） |
| `2562` | 当前/应用源 |

### 发送方式（ADB 验证）

```bash
# 切到 HDMI 1
adb shell am broadcast -a com.jamdeo.tv.policy.SILO_SWITCH_EVENT --ei silo_input_src_id 1281

# 切到 HDMI 2 / ARC
adb shell am broadcast -a com.jamdeo.tv.policy.SILO_SWITCH_EVENT --ei silo_input_src_id 1282
```

**关键验证结论**：
- **无需 root**：普通 shell（不带 `su`）直接发送即有效
- **无需特殊权限**：第三方 APK 用 `Context.sendBroadcast()` 即可发送
- **无浮层**：切换时**不弹出**信号源选择浮层，直接切换到目标 HDMI
- **不依赖聚好看**：即使聚好看被停用，广播依然有效
- 切换成功判据：屏幕出现 HDMI 提示框（如 "HDMI 1 3840×2160@60Hz" / "HDMI 2/ARC 1920×1080@60Hz"）

### 在 APK 中发送

```java
// HDMI 1
sendBroadcast(new Intent("com.jamdeo.tv.policy.SILO_SWITCH_EVENT")
        .putExtra("silo_input_src_id", 1281));

// HDMI 2 / ARC
sendBroadcast(new Intent("com.jamdeo.tv.policy.SILO_SWITCH_EVENT")
        .putExtra("silo_input_src_id", 1282));
```

无需在 manifest 中声明接收者（只发不收），也无需任何特殊权限。

## 为什么不能用标准方式

| 方式 | 结果 |
|---|---|
| `TvInputManager.getTvInputList()` 列输入 | 返回空（无 TV 输入服务） |
| `input keyevent 178`（呼出信号源浮层） | 聚好看停用后失效（浮层在聚好看包内） |
| `TvContract.buildChannelUriForPassthroughInput()` | 无可用 Live TV 处理 |
| `SourceManager`/`HSTVService` 直调 | 需系统签名，第三方 APK 无法调用 |
| APK 内 exec `su` | `error=13`（`/system/xbin/su` 权限 `root:shell`，第三方 uid 无法执行） |
| **SILO 广播** | ✅ 有效，免 root 免权限无浮层 |

## 与 CleanYourTV 的集成

在 `MainActivity.java` 中新增：

```java
/** Hisense VIDAA SILO input-switch broadcast. sourceId 1281 = HDMI 1, 1282 = HDMI 2/ARC. */
private void switchSiloInput(int sourceId) {
    try {
        sendBroadcast(new Intent("com.jamdeo.tv.policy.SILO_SWITCH_EVENT")
                .putExtra("silo_input_src_id", sourceId));
    } catch (Exception e) {
        Toast.makeText(this, R.string.not_available, Toast.LENGTH_SHORT).show();
    }
}
```

`showInputs()` 中，当 `silo_sources` 配置开启且无 TvInput 可列时，注入固定项：

```java
if (rows.getChildCount() == 0 && siloSources) {
    addInputRow(rows, popup, R.drawable.ic_input, "HDMI 1", -1, () -> switchSiloInput(1281));
    addInputRow(rows, popup, R.drawable.ic_input, "HDMI 2", -1, () -> switchSiloInput(1282));
}
```

## 验证截图要点

- 切换 HDMI 1：OSD 显示 "HDMI 1 3840×2160@60Hz"
- 切换 HDMI 2/ARC：OSD 显示 "HDMI 2/ARC 1920×1080@60Hz"
- 均无浮层、无聚好看依赖、无需 root

## 相关参考

- 逆向资料：`com.jamdeo.tv.policy` 广播、`SiloConstants`
- 聚好看 APK（`com.jamdeo.tv.vod`）：信号源浮层（SignalViewService/SignalBroadcastReceiver）在其包内
