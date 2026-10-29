# CleanYourTV 海信 VIDAA 定制版

基于 [CleanYourTV](https://github.com/Gathin23/cleanyourtv)（Android TV/Google TV 精简桌面）为**海信 VIDAA 系统**（Android 8.0，SDK 26）做的定制适配。

## 背景

海信 VIDAA 电视的**信号源浮层（输入源切换面板）由"聚好看"应用（`com.jamdeo.tv.vod`）承载**。精简/停用聚好看后，遥控器信号源键（按键 178）将无法呼出输入源面板，HDMI 信号源无法切换。

**关键发现**：该电视的 `TvInputManager` 服务完全为空（`dumpsys tv_input` 的 `inputMap`/`mHardwareList` 均为空），系统没有注册任何标准 TV 输入服务。因此：

- CleanYourTV 原版依赖 `TvInputManager.getTvInputList()` 枚举 HDMI 输入——在本机得到空列表，标准 TV 输入通道**不可用**。
- 必须改用海信私有的 **SILO 广播**（`com.jamdeo.tv.policy.SILO_SWITCH_EVENT`）来切换输入源。

## 核心改动

### 1. 信号源切换改为 SILO 广播（`MainActivity.java`）

新增 `switchSiloInput(int sourceId)` 方法，发送海信私有广播：

```java
sendBroadcast(new Intent("com.jamdeo.tv.policy.SILO_SWITCH_EVENT")
        .putExtra("silo_input_src_id", sourceId));
```

- `silo_input_src_id = 1281` → **HDMI 1**
- `silo_input_src_id = 1282` → **HDMI 2 / ARC**

**验证结论**：此广播**无需 root**、无需特殊权限、**不弹出信号源浮层**、**不依赖聚好看**，可直接无浮层切换 HDMI 信号源。

### 2. Inputs 菜单注入固定 HDMI 1/2（`MainActivity.java` + `Config.java`）

新增 `silo_sources` 配置开关（默认开启）。当开启且 TvInputManager 无输入可列时，Inputs 菜单直接注入固定的 **HDMI 1** / **HDMI 2** 两项，点击分别发 SILO 广播。

### 3. 中文界面与顶栏设置按钮

- 顶栏设置按钮文字：`简洁桌面设置` → `设置`（改 `values-zh-rCN/strings.xml`）
- 顶栏设置按钮点击后直接进入 CleanYourTV 桌面设置（`launcher_settings_button`）

### 4. 应用库默认开启（`config.json`）

电视原有默认 `dock_apps`（Netflix/YouTube 等）均未安装导致桌面应用显示不全，通过 `config.json` 开启应用库：

```json
{
  "launcher_settings_button": true,
  "app_library": true,
  "library_button": true
}
```

## 目录结构

```
cleanyourtv-hisense/
├── README.md                 # 本说明
├── config.json               # 电视端运行时配置（推送到 /sdcard/Android/data/com.gathin.tvhome/files/）
├── SILO-SWITCH.md            # SILO 广播切换 HDMI 的完整原理与验证记录
└── patches/
    └── (待补：源码改动 patch 或 diff 说明)
```

## 安装方法

### 1. 编译 CleanYourTV

原版 launcher 源码位于 `cleanyourtv-main/launcher/`，本定制版需应用以下源码改动后编译：

- `app/src/main/java/com/gathin/tvhome/MainActivity.java` — 加入 `switchSiloInput()` 方法与 Inputs 菜单 SILO 注入
- `app/src/main/java/com/gathin/tvhome/Config.java` — 新增 `silo_sources` 配置项
- `app/src/main/res/values/config.xml` — 新增 `<bool name="silo_sources">true</bool>`
- `app/src/main/res/values-zh-rCN/strings.xml` — 设置按钮文字改为"设置"

构建要求：JDK 17、Android SDK（compileSdk 34、minSdk 26）、Gradle 8.9。

```bash
cd cleanyourtv-main/launcher
./gradlew assembleRelease   # 或 gradlew.bat (Windows)
# 产物：app/build/outputs/apk/release/app-release.apk
```

### 2. 安装到电视

```bash
adb connect <电视IP>:5555
adb push app-release.apk /data/local/tmp/tvhome.apk
adb shell pm install -r /data/local/tmp/tvhome.apk
adb shell am start -n com.gathin.tvhome/.MainActivity
```

### 3. 设为默认桌面

```bash
adb shell cmd package set-home-activity --user 0 com.gathin.tvhome/.MainActivity
```

### 4. 推送 config.json

```bash
adb push config.json /sdcard/Android/data/com.gathin.tvhome/files/config.json
adb shell am force-stop com.gathin.tvhome
adb shell am start -n com.gathin.tvhome/.MainActivity
```

## 已验证功能

- ✅ 无浮层直接切换 HDMI 1 / HDMI 2（SILO 广播，无需 root）
- ✅ 设为默认桌面后按 Home 进入 CleanYourTV
- ✅ 应用库显示全部已安装应用
- ✅ 顶栏设置按钮进桌面设置
- ✅ 兼容已精简/停用"聚好看"的电视

## 致谢

- [CleanYourTV](https://github.com/Gathin23/cleanyourtv) — 原作者
- 海信 VIDAA 社区逆向资料

## 免责声明

本定制版仅供学习与技术折腾使用。修改系统桌面与信号源切换行为可能影响电视正常使用，操作前请备份重要数据，风险自负。
