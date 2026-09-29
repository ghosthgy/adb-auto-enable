# 🔧 adb-auto-enable

[English](README.md) | **中文说明**

**每次开机自动开启无线 ADB 调试并切换至指定端口（默认 5555），无需 Root 权限，支持开机自动执行自定义 ADB Shell 命令！**

Android 14+ 引入了更严格的 ADB 安全机制，在设备休眠或重启后会自动关闭无线调试或随机分配端口，导致自动化工具（如 Home Assistant、智能家居脚本等）无法稳定连接。**ADB Auto-Enable** 能够在每次开机后自动重新启用无线 ADB、自动绑定固定端口，并执行自定义 shell 命令，确保无人值守设备的长效稳定连接。

> [!WARNING]
> **安全警告：** 本应用会在您配置的端口（默认 5555）上启用 Android Debug Bridge (ADB)，该服务赋予远程连接者完整的系统权限。虽然 ADB 首次连接需要 RSA 密钥配对授权，但**一旦计算机获得授权，将拥有永久无限制的访问权限**（包括安装卸载应用、读写数据、执行任意 Shell 指令等）。
> **本软件仅建议在隔离或受信任的局域网环境（如家庭内网）中使用**，**切勿在公共 Wi-Fi 或未经防火墙保护的网络中开启 ADB**，切勿将 ADB 端口映射至公网。

---

## ✨ 核心特性

- 🚀 **开机自动开启无线 ADB**：无感开机唤醒，自动写入 `adb_wifi_enabled = 1` 并关闭连接密钥超时机制。
- 🔄 **自动固定目标端口**：开机后自动通过 mDNS 或 64 线程扫描找到随机端口，并切换至目标端口（如 `5555`）。
- ⚡ **自动执行自定义 ADB 命令**：支持配置多行 ADB Shell 命令，端口就绪后自动在后台顺序执行。
- 💻 **交互式 Web 控制台 & ADB 终端**：
  - 在线单次配对与状态监控。
  - 在线保存/修改自定义命令脚本并即时运行。
  - 内置交互式 Web ADB 终端，可随时输入命令执行并查看回显。
  - 实时 Logcat 日志流查看。
- 🛡️ **TCL / Android TV 自启保活适配**：内置针对 TCL 等特定厂商系统的自启保护策略（`AppBootPolicy` 与 `AUTO_START`）。
- 🤫 **完全静默模式**：支持随时关闭后台 Web 服务，仅保留开机纯后台静默配置流程。

---

## 🚀 快速上手

### 系统要求
- Android 14+（已在 Chromecast with Google TV、各类 Android TV 电视盒子及现代 Android 设备上测试）
- 已连接 Wi-Fi 或以太网有线网络

---

### 1. 安装应用

从 [Releases 页面](https://github.com/ghosthgy/adb-auto-enable/releases) 下载最新 APK，或通过源码构建：

```bash
adb install ADB-auto-enable-v*.apk
```

> [!IMPORTANT]
> **首次安装须知：** 安装 APK 后，**必须在设备上打开一次本应用**（可通过 *设置 → 应用 → 查看所有应用 → ADB Auto-Enable* 打开，或在启动器中打开），以初始化后台服务。之后在局域网电脑/手机浏览器中访问 `http://设备IP:9093`。

---

### 2. 首次配对（仅需一次）

1. 在同局域网的电脑/手机浏览器中打开：`http://<设备IP>:9093`。
2. 在 Android 设备上：
   - 进入 **设置 → 开发者选项 → 无线调试**。
   - 点击 **“使用配对码配对设备”**。
3. 保持配对码弹窗显示，将页面上显示的 **6位配对码** 与 **5位配对端口** 输入到 Web 控制台的配对卡片中。
4. 点击 **“🔗 Pair Device”**。
5. 配对成功后，应用将自动利用本地 ADB 为自身授权 `WRITE_SECURE_SETTINGS` 权限。

---

### 3. 配置自定义 ADB 命令（可选）

在 Web 控制台的 **⚡ Custom ADB Commands** 卡片中：
1. 勾选 **“Run automatically after ADB is ready on boot & port switch”**。
2. 在输入框中输入每次开机需要自动执行的 shell 命令（支持以 `#` 开头的注释），例如：
   ```bash
   # 保持常亮（接入电源时不休眠）
   settings put global stay_on_while_plugged_in 3
   # 设置屏幕分辨率/DPI
   wm density 320
   # 禁用特定系统应用或广告服务
   # pm disable-user --user 0 com.example.adservice
   echo "ADB init completed successfully"
   ```
3. 点击 **“💾 Save Commands”** 保存；也可以点击 **“▶️ Run Saved Commands Now”** 立即执行并查看回显。

---

### 4. 验证与连接

1. 可在 Web 控制台点击 **“🔀 Switch Target Port Now”** 测试端口切换。
2. 重启设备，等待 30~60 秒（系统启动与网络稳定延迟）。
3. 在电脑端直接连接固定端口：
   ```bash
   adb connect <设备IP>:5555
   ```

---

## 🛠️ Web API 接口说明

内置 NanoHTTPD 服务（端口 9093）提供了一套完整的 REST API：

| 接口路径 | 请求方式 | 说明 |
| :--- | :---: | :--- |
| `/api/status` | `GET` | 获取当前配对状态、目标端口、当前可用端口、自定义命令配置等 |
| `/api/pair` | `POST` | 提交 `port` 和 `code` 执行本机的无线配对 |
| `/api/port` | `POST` | 设置目标固定端口（参数 `port=5555`） |
| `/api/switch` | `GET` | 立即触发一次端口发现并切换到目标端口 |
| `/api/custom_commands` | `POST` | 保存自定义命令脚本（参数 `commands` 和 `onBoot=true/false`） |
| `/api/run_custom_commands` | `POST` | 立即执行已保存的自定义命令列表并返回结果 |
| `/api/exec` | `POST` | 实时执行单条 ADB 命令（参数 `command`），返回执行输出 |
| `/api/logs` | `GET` | 读取当前应用的 Logcat 实时日志 |
| `/api/reset` | `POST` | 清除配对信息与本地 RSA 密钥，重新进入未配对状态 |
| `/api/webserver` | `POST` | 开启或关闭 Web 服务（参数 `enabled=true/false`） |
| `/api/test` | `GET` | 模拟发送开机广播，测试完整开机自启与配置流程 |

---

## ⚙️ 工作原理

```text
设备开机 (LOCKED_BOOT_COMPLETED / BOOT_COMPLETED)
  ↓
BootReceiver 触发并立即启动 AdbConfigService 前台服务
  ↓
启动 NanoHTTPD Web 配置服务 (端口 9093，若已启用)
  ↓
第 0 步：立即向 Settings.Global 写入 adb_wifi_enabled = 1
  ↓
第 1 步：等待 Wi-Fi / 以太网网络连接成功 (最长 60 秒)
  ↓
第 2 步：等待系统完全稳定 (30 秒安全缓冲)
  ↓
第 3 步：发现随机生成的 ADB 端口 (优先 mDNS 广播解析 → 64 线程 Socket 端口扫描兜底)
  ↓
第 4 步：使用本地 RSA 密钥连接 ADB (127.0.0.1 环回地址 → 设备 LAN IP 降级重试)
  ↓
第 5 步：发送 tcpip:<target_port> 指令切换至目标端口 (默认 5555)
  ↓
第 6 步：自动连接新端口并执行自定义 ADB Shell 命令（若已配置）
  ↓
配置完成！外部设备可随时连接固定端口。
```

---

## 🔨 编译构建

### 方式一：GitHub Actions 云端编译（推荐）
本项目自带完整自动化编译工作流：
1. Fork 或推送代码到您自己的 GitHub 仓库。
2. 进入 GitHub 仓库页面，点击 **Actions** 标签。
3. 选择 **Build and Release APK** 点击 **Run workflow** 手动触发（或推送 Tag 如 `v0.3.4` 自动触发）。
4. 构建完成后，在下方 **Artifacts** 区域即可直接下载编译生成的 APK。

### 方式二：本地 Gradle 编译
需安装 **JDK 17+** 与 **Android SDK**：
```bash
# 构建 Debug 版本
./gradlew assembleDebug

# 构建 Release 版本
./gradlew assembleRelease
```
输出路径：`app/build/outputs/apk/`。

---

## ❓ 常见问题排查 (Troubleshooting)

### 1. 配对失败
- 确保设备与电脑在同一局域网网段内。
- 在 Android 设备上打开“使用配对码配对”弹窗后，**切勿离开或关闭该弹窗**，否则配对码会立即失效。
- 如仍失败，可通过命令行查看日志：
  ```bash
  adb logcat -s "ADBAutoEnable:*"
  ```

### 2. 重启后未自动切换端口
- 检查应用是否具有自启权限（部分国产 TV 系统需在管家中放行自启）。
- 查看 Web 控制台或 Logcat 中是否有 `Successfully configured ADB on port 5555!`。

### 3. 无法打开 Web 界面
- 确认手机/电脑与电视在同一个局域网，且没有开启 AP 隔离。
- 检查设备 IP 是否正确。

---

## 📄 开源许可与致谢

- 遵循 [MIT License](LICENSE)。
- 感谢 [NanoHTTPD](https://github.com/NanoHttpd/nanohttpd) 提供嵌入式 HTTP 服务支持。
- 感谢 [libadb-android](https://github.com/MuntashirAkon/libadb-android) 提供 ADB 协议通信支持。
