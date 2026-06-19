# Silent Monitor — CTF Writeup

> **Silent Monitor** — Enumerate a running internal service, exploit a vulnerable web application, pivot through the system, and crack your way to root.

---

## Table of Contents

- [Challenge Overview](#challenge-overview)
- [Step 1 — Nmap Scan](#step-1--nmap-scan)
- [Step 2 — Directory Enumeration with Dirb](#step-2--directory-enumeration-with-dirb)
- [Step 3 — SQL Injection: Login Bypass](#step-3--sql-injection-login-bypass)
- [Step 4 — Intercepting the Request & Session Cookie Analysis](#step-4--intercepting-the-request--session-cookie-analysis)
- [Step 5 — OS Command Injection via Health Check Endpoint](#step-5--os-command-injection-via-health-check-endpoint)
- [Step 6 — SSH Access as sysadmin & Flag 1](#step-6--ssh-access-as-sysadmin--flag-1)
- [Step 7 — Privilege Escalation: Race Condition in Backup Script](#step-7--privilege-escalation-race-condition-in-backup-script)
- [Step 8 — Root Access & Final Flag](#step-8--root-access--final-flag)
- [Flags Summary](#flags-summary)
- [Vulnerability Summary](#vulnerability-summary)
- [Attack Chain](#attack-chain)
- [Key Takeaways](#key-takeaways)

---

## Challenge Overview

| Field       | Details                                                       |
|-------------|----------------------------------------------------------------|
| Name        | Silent Monitor                                                  |
| Platform    | TryHackMe                                                       |
| Category    | Web Exploitation (SQLi + Command Injection) → Linux Privilege Escalation |
| Target IP   | 10.49.150.232                                                   |
| Attacker IP | 10.49.125.62                                                    |
| Objective   | Exploit an internal monitoring web app, pivot to a system account, and escalate to root |

---

## Step 1 — Nmap Scan

Started with a full TCP port scan including service/version detection and default script scanning.

**Command:**
```bash
nmap -sV -p- -sC -sS 10.49.150.232
```

> **Command Breakdown:**
> - `-sV` — Probes open ports to determine service/version information
> - `-p-` — Scans all 65,535 TCP ports (not just the default top 1000)
> - `-sC` — Runs Nmap's default script scan (NSE) for additional enumeration
> - `-sS` — Performs a SYN ("stealth") scan

**Scan Results:**

| Port    | State | Service | Version                                       |
|---------|-------|---------|-------------------------------------------------|
| 22/tcp  | open  | ssh     | OpenSSH 8.9p1 Ubuntu 3ubuntu0.15                |
| 5050/tcp| open  | http    | Werkzeug httpd 2.0.2 (Python 3.10.12)           |

**Additional Info:**
```
http-title: CorpNet — Network Operations Centre
OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

> **Key Observations:**
> - Port **5050** is running a **Python Flask application** (identified by the Werkzeug development server banner) titled **"CorpNet — Network Operations Centre"**
> - This is a non-standard web port — likely an internal monitoring/ops tool not meant for public exposure
> - SSH is open on port 22 — a probable target for credential-based access later

---

## Step 2 — Directory Enumeration with Dirb

Ran Dirb against the web application on port 5050 to discover hidden directories.

**Command:**
```bash
dirb http://10.49.150.232:5050/
```

> **Command Breakdown:**
> - `dirb` — A web content scanner that brute-forces directory and file names against a target URL using a wordlist
> - Default wordlist used: `/usr/share/dirb/wordlists/common.txt`

**Result:**

| Path        | Status Code | Size  |
|-------------|-------------|-------|
| /internal   | 200         | 8770  |

> The `/internal` path returned a **200 OK**, revealing a hidden login page — clearly intended for internal staff (e.g., NOC operators) rather than public access.

---

## Step 3 — SQL Injection: Login Bypass

Navigating to `/internal` presented a login form. Tested the username/password fields with a classic SQL injection authentication bypass payload.

**Payload used (example):**
```sql
' OR '1'='1' -- -
```

> **How this works:** If the backend builds a query like:
> ```sql
> SELECT * FROM users WHERE username='<input>' AND password='<input>'
> ```
> Injecting `' OR '1'='1' -- -` turns the condition into something that is always true, while `--` comments out the rest of the original query (including the password check) — bypassing authentication entirely without knowing valid credentials.

This successfully bypassed the login form and granted a valid authenticated session, logged in with the **`operator`** role under the username **`netops`**.

---

## Step 4 — Intercepting the Request & Session Cookie Analysis

Using a proxy tool (e.g., Burp Suite), intercepted a request to the application's health-check feature to study how the authenticated session worked.

**Intercepted Request:**
```http
POST /internal/health HTTP/1.1
Host: 10.49.150.232:5050
Content-Length: 40
Content-Type: application/x-www-form-urlencoded
Cookie: session=eyJyb2xlIjoib3BlcmF0b3IiLCJ1c2VyIjoibmV0b3BzIn0.ajWNNA.I3MlCz9gR-kthnQaEidVs0yc0oI
```

> **Cookie Breakdown:** Flask signs session cookies using `itsdangerous`, producing three dot-separated parts: `<payload>.<timestamp>.<signature>`. Base64-decoding the first segment reveals the session contents:
> ```json
> {"role": "operator", "user": "netops"}
> ```
> This confirms the authenticated session belongs to the `netops` user with the `operator` role — granted purely through the SQL injection bypass in the previous step.

---

## Step 5 — OS Command Injection via Health Check Endpoint

The `/internal/health` endpoint accepts a `target` parameter — almost certainly used server-side to **ping** a host for connectivity monitoring (a classic feature in NOC/monitoring dashboards). Tested it for **OS Command Injection** by appending a URL-encoded newline followed by a second command.

**Payload:**
```
target=127.0.0.1%0acat secret.config
```

> **Payload Breakdown:**
> - `127.0.0.1` — A legitimate target IP, so the first command (likely `ping`) executes normally and doesn't raise suspicion
> - `%0a` — URL-encoded newline character (`\n`)
> - `cat secret.config` — A second, attacker-controlled command appended after the newline
>
> If the backend executes this input via something like `os.system(f"ping -c 1 {target}")` without sanitisation, the newline breaks out of the intended single command and the shell executes `cat secret.config` as a **separate, second command**.

**Server Response — contents of secret.config:**
```ini
[auth]
session_lifetime = 1800
# service account used by the backup agent
# TODO: migrate to secrets manager before Q2 audit

[backup_agent]
run_as   = sysadmin
password = S3cur3Backup$xxc3ss!

[smtp]
host = 127.0.0.1
port = 25
from = noc-alerts@corp.internal
```

> 🔑 **Credentials Found:**
> - **Username:** `sysadmin`
> - **Password:** `S3cur3Backup$xxc3ss!`
>
> The comment `# TODO: migrate to secrets manager before Q2 audit` confirms this is a known piece of technical debt — a real-world pattern where service account credentials are left in plaintext config files "temporarily."

---

## Step 6 — SSH Access as sysadmin & Flag 1

Used the leaked credentials to SSH directly into the target as the `sysadmin` user.

**Command:**
```bash
ssh sysadmin@10.49.150.232
```

Once authenticated, listed the home directory contents and read the first flag.

**Commands:**
```bash
ls
cat user.txt
```

**Output:**
```
backups  user.txt

THM{xxxx_4nd_cMd_1nj3ct10n_l3D_y0u_h3re!}
```

> 🚩 **Flag 1:** `THM{xxxx_4nd_cMd_1nj3ct10n_l3D_y0u_h3re!}`
>
> The flag text itself confirms the two chained vulnerabilities used so far: **SQL Injection** (login bypass) and **Command Injection** (config file exfiltration).

---

## Step 7 — Privilege Escalation: Race Condition in Backup Script

Explored the `backups` directory found in the `sysadmin` home folder and discovered a custom Python script alongside an encrypted credential vault.

**Commands:**
```bash
cd backups
ls
```

**Output:**
```
README.txt  cp.py  infrastructure.kdbx
```

> **Findings:**
> - **`cp.py`** — A custom backup/copy script, likely executed periodically with elevated (root) privileges via a cron job or scheduled backup process
> - **`infrastructure.kdbx`** — An encrypted KeePass password vault, potentially containing further infrastructure credentials
> - **`README.txt`** — Documentation for the backup process

Analysis of `cp.py` revealed a **Time-Of-Check-To-Time-Of-Use (TOCTOU) race condition**: the script checks a file's properties (e.g., path or permissions) and then operates on that file in a separate step. Because there's a timing gap between the *check* and the *use*, the file can be swapped out (e.g., for a symlink pointing elsewhere) in that window — allowing a lower-privileged user to trick the root-executed script into reading or writing an attacker-controlled file.

**Exploit script written:** `copyfail.py` — designed to win this race condition window and leverage the privileged execution context of `cp.py` to spawn a root shell.

**Commands:**
```bash
nano copyfail.py
python3 copyfail.py
```

> **Command Breakdown:**
> - `nano copyfail.py` — Opens the nano text editor to write the exploit script
> - `python3 copyfail.py` — Executes the race-condition exploit

**Result:**
```bash
whoami
```
```
root
```

> ✅ **Root shell obtained** by winning the TOCTOU race condition against the privileged `cp.py` backup process.

---

## Step 8 — Root Access & Final Flag

With a root shell active, confirmed the current working directory and navigated to `/root` to retrieve the final flag.

**Commands:**
```bash
pwd
cd /
ls
cd root
ls
cat root.txt
```

**Output:**
```
/home/sysadmin/backups

root.txt  snap

THM{xxxx_V4ul7_H4s_b33n_cr4ck3d_0peN}
```

> 🚩 **Final Flag (root):** `THM{xxxx_V4ul7_H4s_b33n_cr4ck3d_0peN}`

---

## Flags Summary

| # | Flag                                       | User Context | Method                                          |
|---|---------------------------------------------|---------------|---------------------------------------------------|
| 1 | `THM{xxxx_4nd_cMd_1nj3ct10n_l3D_y0u_h3re!}` | sysadmin      | SQLi login bypass + OS Command Injection → leaked credentials |
| 2 | `THM{xxxx_V4ul7_H4s_b33n_cr4ck3d_0peN}`     | root          | TOCTOU race condition exploit against `cp.py`     |

---

## Vulnerability Summary

| # | Vulnerability                          | Location                          | Impact                                         |
|---|------------------------------------------|------------------------------------|--------------------------------------------------|
| 1 | Hidden internal panel discoverable       | `/internal`                        | Exposed internal-only NOC login portal           |
| 2 | SQL Injection (authentication bypass)    | `/internal` login form             | Bypassed login without valid credentials         |
| 3 | OS Command Injection                     | `/internal/health` (`target` param)| Arbitrary command execution as the web app user  |
| 4 | Plaintext service credentials in config  | `secret.config`                    | Leaked `sysadmin` SSH password                   |
| 5 | TOCTOU race condition in backup script   | `cp.py` (root-executed)            | Local privilege escalation to root               |

---

## Attack Chain

```
Nmap → Port 5050 (Flask app) → Dirb → /internal discovered
→ SQLi login bypass → session as netops (role: operator)
→ Intercept /internal/health request → decode Flask session cookie
→ Command Injection via target=127.0.0.1%0acat secret.config
→ Leaked sysadmin password: S3cur3Backup$Acc3ss!
→ SSH as sysadmin → Flag 1
→ Enumerate ~/backups → cp.py (TOCTOU race condition)
→ Exploit (copyfail.py) → root shell
→ /root/root.txt → Flag 2
```

---

## Key Takeaways

- **Internal/admin panels are still part of the attack surface** — even if not linked publicly, they're discoverable through directory brute-forcing
- **Always use parameterised queries / prepared statements** — string-concatenated SQL queries make authentication bypass trivial
- **Never pass user input directly into shell commands** (e.g., `os.system`, `subprocess` with `shell=True`) — a single newline character was enough to chain an arbitrary second command
- **Configuration files should never store plaintext credentials** — the `# TODO: migrate to secrets manager` comment shows the developers were aware of this risk but hadn't acted on it
- **Privileged scripts (run as root via cron/sudo) must avoid TOCTOU patterns** — any gap between checking a file and acting on it can be exploited via symlink races; use atomic operations or proper locking instead
- **Defense in depth matters** — this box required chaining four separate weaknesses (SQLi → CMDi → leaked creds → race condition) to reach root, showing how individually "minor" issues compound into full compromise

---

*Writeup by: [showrya] | Platform: TryHackMe | Room: Silent Monitor*
