# 🔧 adb-auto-enable

**English** | [中文说明](README_zh.md)

**Automatically enable wireless ADB debugging, switch to your chosen port (default 5555), and execute custom ADB shell commands on every boot — no root required!**

Android 14+ introduces enhanced ADB security mechanisms that disable wireless debugging and randomize the listening port after reboot or sleep, breaking automation setups (e.g. Home Assistant, automated testing, media center controls). **ADB Auto-Enable** solves this by automatically re-enabling wireless ADB on boot, discovering the randomized port, binding ADB to your desired target port, and executing user-defined custom ADB shell commands.

> [!WARNING]
> **SECURITY WARNING:** This application enables Android Debug Bridge (ADB) on your configured port (default 5555), which provides remote access to your device with full system privileges. While ADB connections require RSA key authentication (users must accept the connection on first pairing), **once a computer is authorized, it has permanent unrestricted access** to install applications, access all data, execute shell commands, and take complete control of your device without further prompts.
> **This app should ONLY be used on isolated or trusted networks** (such as a home network behind a firewall with no port forwarding) and **NEVER on public WiFi, guest networks, or any network you do not fully control**. Never expose the ADB port to the public Internet.

<p align="center">
  <img src="gui2.png" alt="ADB Auto-Enable Web GUI" width="700">
</p>

---

## ✨ Features

- 🚀 **Zero-Touch Boot Activation**: Automatically writes `adb_wifi_enabled = 1` and disables ADB key revocation on boot.
- 🔄 **Fixed Target Port Switching**: Discovers the randomized ADB port via mDNS or a 64-thread parallel socket sweep (`32768–60999`) and switches to your chosen port (default `5555`).
- ⚡ **Auto-Execution of Custom ADB Commands**: Define multi-line ADB shell commands that run automatically in sequence once the target ADB port is ready.
- 💻 **Interactive Web Control & Terminal**:
  - Embedded web UI on port `9093` for initial pairing, status overview, and live Logcat streaming.
  - Custom command management (save, test, and run on demand).
  - Built-in interactive Web ADB shell terminal for executing commands and viewing real-time outputs.
- 🛡️ **TCL / Android TV Auto-Start Protection**: Built-in support for manufacturer-specific background policies (`AppBootPolicy` and `AUTO_START`).
- 🤫 **Silent Mode**: Option to disable the web server on future boots to run purely in the background with minimal resource footprint.

---

## 🚀 Quick Start

### Requirements
- Android 14+ (tested on Chromecast with Google TV, Android TV boxes, and modern Android builds)
- Active Wi-Fi or Ethernet connection

---

### 1. Installation

Download the latest APK from the [Releases](https://github.com/ghosthgy/adb-auto-enable/releases) page, or build from source:

```bash
adb install ADB-auto-enable-v*.apk
```

> [!IMPORTANT]
> **First-Time Installation Note:** After installing the APK, you **must open the app once on your device** (via *Settings → Apps → See All Apps → ADB Auto-Enable* or through your app launcher) to initialize the background service before accessing `http://<device-ip>:9093`.

---

### 2. Initial Pairing (One-Time Setup)

1. Open your browser and navigate to `http://<device-ip>:9093`.
2. On your Android device:
   - Go to **Settings → Developer Options → Wireless Debugging**.
   - Tap **"Pair device with pairing code"**.
3. Keep the pairing dialog open on your device screen.
4. Enter the **Pairing Code** (6 digits) and **Pairing Port** (5 digits) into the web interface.
5. Click **"🔗 Pair Device"**.
6. Upon successful pairing, the app will automatically use local ADB to grant itself the required `WRITE_SECURE_SETTINGS` permission.

---

### 3. Custom ADB Commands Configuration (Optional)

In the **⚡ Custom ADB Commands** card on the web interface:
1. Ensure **"Run automatically after ADB is ready on boot & port switch"** is checked.
2. Enter your custom shell commands (one command per line; lines starting with `#` are comments):
   ```bash
   # Keep screen awake while charging
   settings put global stay_on_while_plugged_in 3
   # Set custom display density / DPI
   wm density 320
   # Disable unwanted background services or ads
   # pm disable-user --user 0 com.example.adservice
   echo "ADB boot configuration completed"
   ```
3. Click **"💾 Save Commands"** to save your setup. You can also click **"▶️ Run Saved Commands Now"** to test immediately.

---

### 4. Verify & Connect

1. Click **"🔀 Switch Target Port Now"** in the web interface to test manually.
2. Reboot your device and wait 30–60 seconds (service initialization + 30s system stabilization delay).
3. Connect from your computer:
   ```bash
   adb connect <device-ip>:5555
   ```

---

## 🛠️ Web API Reference

The embedded HTTP server (port 9093) exposes a comprehensive REST API:

| Endpoint | Method | Description |
| :--- | :---: | :--- |
| `/api/status` | `GET` | Returns pairing state, target port, current active port, and custom command settings |
| `/api/pair` | `POST` | Pairs with local wireless debugging using `port` and `code` parameters |
| `/api/port` | `POST` | Sets the target ADB port (`port=5555`) |
| `/api/switch` | `GET` | Triggers port discovery and switches ADB to target port |
| `/api/custom_commands` | `POST` | Saves custom commands list (`commands` and `onBoot=true/false`) |
| `/api/run_custom_commands` | `POST` | Executes saved custom commands immediately and returns outputs |
| `/api/exec` | `POST` | Executes a single interactive ADB shell command (`command`) and returns output |
| `/api/logs` | `GET` | Retrieves live Logcat logs for `ADBAutoEnable` |
| `/api/reset` | `POST` | Clears stored RSA keys and resets pairing state |
| `/api/webserver` | `POST` | Enables or disables the web server (`enabled=true/false`) |
| `/api/test` | `GET` | Simulates a boot event to test the complete initialization pipeline |

---

## ⚙️ How It Works

```text
Device Boot (LOCKED_BOOT_COMPLETED / BOOT_COMPLETED)
  ↓
BootReceiver starts AdbConfigService Foreground Service
  ↓
Start Web Server (port 9093, if enabled)
  ↓
Step 0: Immediately write adb_wifi_enabled = 1 & disable key revocation
  ↓
Step 1: Wait for Wi-Fi / Ethernet connection (up to 60s)
  ↓
Step 2: Wait for system stabilization (30s)
  ↓
Step 3: Discover randomized ADB port (mDNS → 64-thread socket sweep fallback)
  ↓
Step 4: Connect to ADB daemon (127.0.0.1 loopback → LAN IP fallback)
  ↓
Step 5: Send tcpip:<target_port> command to switch port
  ↓
Step 6: Execute custom ADB shell commands (if configured)
  ↓
Done! ADB is accessible on your target port for external connections.
```

---

## 🔨 Building from Source

### GitHub Actions Cloud Build (Recommended)
This repository includes a fully automated GitHub Actions workflow:
1. Fork or push changes to your GitHub repository.
2. Go to **Actions → Build and Release APK → Run workflow** (or push a tag like `v0.3.4`).
3. Download the compiled APKs directly from the **Artifacts** section or GitHub Release assets.

### Local Gradle Build
Requires **JDK 17+** and the **Android SDK**:
```bash
# Build Debug APK
./gradlew assembleDebug

# Build Release APK
./gradlew assembleRelease
```
Outputs are generated in `app/build/outputs/apk/`.

---

## ❓ Troubleshooting

### Pairing Fails
- Ensure your device and computer are connected to the same Wi-Fi network subnet.
- Keep the "Pair device with pairing code" dialog open on your TV / device while clicking "Pair Device" in the web browser.
- Check live logs in the web interface or via ADB:
  ```bash
  adb logcat -s "ADBAutoEnable:*"
  ```

### Port Does Not Switch on Boot
- Check if your device OS restricts background auto-start (e.g. TCL, Xiaomi, or Huawei battery managers).
- Verify logs in `http://<device-ip>:9093` for `"Successfully configured ADB on port 5555!"`.

### Web Interface Not Accessible
- Ensure the device IP is correct and AP isolation is disabled on your router.
- Verify the service is running:
  ```bash
  adb shell dumpsys activity services | grep AdbConfigService
  ```

---

## 📄 License & Acknowledgments

- Licensed under the [MIT License](LICENSE).
- Uses [NanoHTTPD](https://github.com/NanoHttpd/nanohttpd) for the embedded web server.
- Uses [libadb-android](https://github.com/MuntashirAkon/libadb-android) for ADB wire protocol implementation.
