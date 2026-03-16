# Offensive Security Write-Ups

```
███████╗██╗  ██╗██████╗ ██╗      ██████╗ ██╗████████╗
██╔════╝╚██╗██╔╝██╔══██╗██║     ██╔═══██╗██║╚══██╔══╝
█████╗   ╚███╔╝ ██████╔╝██║     ██║   ██║██║   ██║   
██╔══╝   ██╔██╗ ██╔═══╝ ██║     ██║   ██║██║   ██║   
███████╗██╔╝ ██╗██║     ███████╗╚██████╔╝██║   ██║   
╚══════╝╚═╝  ╚═╝╚═╝     ╚══════╝ ╚═════╝ ╚═╝   ╚═╝   
```

<div align="center">

**Technical documentation of offensive security laboratories in controlled environments**

[![TryHackMe](https://img.shields.io/badge/-TryHackMe-212C42?style=for-the-badge&logo=tryhackme&logoColor=white)](https://tryhackme.com/p/UmbraNull)
[![HackTheBox](https://img.shields.io/badge/-HackTheBox-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black)](https://app.hackthebox.com/profile/019c4c95-6170-73b2-b61c-c5741403d621)

![Status](https://img.shields.io/badge/Status-Active-success?style=flat-square)
![Rooms](https://img.shields.io/badge/Rooms_Completed-11-blue?style=flat-square)
![Streak](https://img.shields.io/badge/Streak-15_days-orange?style=flat-square)

</div>

---

## 🎯 Repository Purpose

This repository documents **exploitation processes** of vulnerable machines, including:

- **Methodology:** Step-by-step attack chains from reconnaissance to root
- **Tools:** Offensive security toolkit and techniques applied
- **Vulnerabilities:** CVEs exploited and attack vectors identified
- **Remediation:** Security recommendations for each vulnerability

**Target Audience:** Security professionals, pentesters, and ethical hackers seeking practical exploitation documentation.

---

## 💭 Philosophy

```
"Frustration is not failure — it's the signal you're leaving your comfort zone
and confronting what you don't know yet. Every challenge is a teacher."

— UmbraNull
```

This repository reflects a journey from **ignorance to mastery**, one vulnerability at a time.

Each write-up documents:
- ✅ What worked (successes)
- ❌ What didn't (failures and lessons learned)
- 🧠 What was learned (new techniques and concepts)

**Growth mindset over ego.**

---

## 🏆 Skills Demonstrated

### Reconnaissance & Enumeration
```
• Port scanning (Nmap, Masscan)
• Service fingerprinting and version detection
• Directory fuzzing (Gobuster, Feroxbuster, ffuf)
• SMB/NFS enumeration (enum4linux, smbclient)
• CMS identification and version analysis
```

### Web Exploitation
```
• SQL Injection (Error-based, Time-based Blind)
• CMS exploitation (WordPress, Joomla, CMS Made Simple)
• LFI/RFI (Local/Remote File Inclusion)
• File upload vulnerabilities
• Authentication bypass
```

### Credential Harvesting
```
• Hash extraction and analysis
• Password cracking (John the Ripper, Hashcat)
• Salt-based hash cracking (MD5, SHA, NTLM)
• SSH key theft and exploitation
• Credential stuffing and reuse
```

### Privilege Escalation
```
• SUID binary exploitation
• Sudo misconfiguration abuse (GTFOBins)
• PATH manipulation
• Kernel exploits
• Cron job hijacking
• Capabilities abuse
```

### Linux Post-Exploitation
```
• Lateral movement techniques
• Persistence mechanisms
• Data exfiltration
• Log cleaning and anti-forensics
• Rootkit deployment
```

---

## 📂 Documented Rooms

### TryHackMe

<table>
<tr>
<td width="33%">

#### Easy Rooms
- [x] [**Basic Pentesting**](tryhackme/basic-pentesting/)  
  *SMB, SSH bruteforce, SSH key cracking*
  
- [x] [**Kenobi**](tryhackme/kenobi/)  
  *ProFTPD mod_copy, NFS, SUID*
  
- [x] [**Simple CTF**](tryhackme/simple-ctf/)  
  *CMS Made Simple, SQLi, VIM GTFOBins*

- [ ] **Brooklyn Nine Nine**  
  *Steganography, SUDO abuse*
  
- [ ] **Lazyadmin**  
  *Web exploitation, Reverse shells*

</td>
<td width="33%">

#### Medium Rooms
- [x] [**Mr Robot CTF**](tryhackme/mr-robot-ctf/)  
  *WordPress, Hash cracking, SUID*
  
- [x] [**Wonderland**](tryhackme/wonderland/)  
  *Rabbit hole, PATH, Capabilities*
  
- [ ] **Startup**  
  *File upload, Wireshark, Cron*

</td>
<td width="33%">

#### Hard Rooms
- [x] [**Attacktive Directory**](tryhackme/attacktive-directory/)  
  *Kerberos, AS-REP Roasting, DCSync*
  
- [ ] **Active Directory Exploitation**  
  *BloodHound, Pass-the-Hash*

</td>
</tr>
</table>

### HackTheBox

- [ ] **Lame** (Linux Easy)
- [ ] **Legacy** (Windows Easy)
- [ ] **Blue** (Windows Easy)

### Other

- [x] [**Basic Malware RE**](other/basic-malware-re/)  
  *Reverse engineering, Static analysis*

---

## 📖 Write-Up Structure

Each write-up follows a **professional penetration testing report** format:

```
📁 room-name/
├─ README.md (Main write-up)
├─ images/ (Screenshots)
├─ scripts/ (Custom exploits/tools)
└─ notes/ (Raw enumeration data)
```

### Standard Sections

1. **Machine Information** - Basic details (OS, difficulty, IP)
2. **Learning Objectives** - Skills practiced
3. **Attack Surface Analysis** - Ports, services, vulnerabilities
4. **Exploitation Walkthrough** - Step-by-step with commands
5. **Key Learnings** - Techniques explained in depth
6. **Remediation** - How to fix vulnerabilities
7. **Tools Used** - Arsenal and command reference
8. **Timeline** - Time tracking

---

## 🛠️ Offensive Toolkit

### Reconnaissance
```bash
nmap, masscan, autorecon
gobuster, feroxbuster, ffuf, dirsearch
enum4linux, smbclient, smbmap
whatweb, wappalyzer, nikto
```

### Exploitation
```bash
metasploit, searchsploit
burp suite, sqlmap
john, hashcat, hydra
```

### Post-Exploitation
```bash
linpeas, winpeas
bloodhound, crackmapexec
mimikatz, impacket
```

### Custom Scripts
```python
# Python automation for repetitive tasks
# Exploit development and modification
# Custom enumeration scripts
```

---

## 📊 Progress Tracking

### Current Stats (March 2026)

```
╔══════════════════════════════════════╗
║  OFFENSIVE SECURITY JOURNEY          ║
╠══════════════════════════════════════╣
║                                      ║
║  Platforms:                          ║
║  ├─ TryHackMe:    11 rooms           ║
║  ├─ HackTheBox:    0 boxes           ║
║  └─ Total:        11 completed       ║
║                                      ║
║  Streak:          15 days 🔥         ║
║  Rank (THM):      Top 3%             ║
║                                      ║
║  Skills Acquired:                    ║
║  ├─ SMB Exploitation                 ║
║  ├─ SQL Injection                    ║
║  ├─ Hash Cracking (salted)           ║
║  ├─ SUID Privilege Escalation        ║
║  ├─ GTFOBins Techniques              ║
║  ├─ Active Directory Attacks         ║
║  └─ CMS Exploitation                 ║
║                                      ║
║  Next Milestone:                     ║
║  └─ eJPT Certification (Q1 2026)     ║
║                                      ║
╚══════════════════════════════════════╝
```

### Learning Path

```
Phase 1: Foundation (CURRENT)
├─ ✅ Basic Pentesting
├─ ✅ Kenobi
├─ ✅ Simple CTF
└─ 🔄 Brooklyn Nine Nine

Phase 2: Intermediate
├─ 📋 Lazyadmin
├─ 📋 Startup
└─ 📋 Network exploitation rooms

Phase 3: Advanced
├─ 📋 Buffer overflow techniques
├─ 📋 Active Directory deep dive
└─ 📋 Advanced persistence

Certification Goal:
└─ eJPT (March 2026)
```

---

## 🎓 Certifications & Training

### In Progress
- **eJPT** (eLearnSecurity Junior Penetration Tester) - Target: March 2026
- **Fortinet Certified Fundamentals** - In progress

### Planned
- **OSCP** (Offensive Security Certified Professional)
- **eCPPT** (eLearnSecurity Certified Professional Penetration Tester)
- **CRTO** (Certified Red Team Operator)

### Education
- **CST Information Security** - UNIP (2025-2027)
- **Ethical Hacking & Defense** - Udemy (2021)

---

## 🔗 Connect & Collaborate

<div align="center">

[![GitHub](https://img.shields.io/badge/GitHub-100000?style=for-the-badge&logo=github&logoColor=white)](https://github.com/igorleite97)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/igor-leite-a9b839222)
[![TryHackMe](https://img.shields.io/badge/-TryHackMe-212C42?style=for-the-badge&logo=tryhackme&logoColor=white)](https://tryhackme.com/p/UmbraNull)
[![HackTheBox](https://img.shields.io/badge/-HackTheBox-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black)](https://app.hackthebox.com/profile/019c4c95-6170-73b2-b61c-c5741403d621)

</div>

**Looking to collaborate on:**
- CTF team formations
- Vulnerability research
- Tool development
- Knowledge sharing

---

## 📜 Disclaimer

```
⚠️  LEGAL NOTICE

All activities documented in this repository were performed in:
• Authorized training platforms (TryHackMe, HackTheBox)
• Personal lab environments
• With explicit permission from platform owners

NEVER perform security testing on systems without explicit authorization.
Unauthorized access is illegal and punishable by law.

This repository is for EDUCATIONAL PURPOSES ONLY.
```

---

## 🌟 Contributing

Found value in these write-ups? Consider:

- ⭐ **Starring** this repository
- 🍴 **Forking** for your own documentation
- 💬 **Sharing** with the security community
- 📝 **Providing feedback** via issues

**Collaboration welcome!** If you have:
- Alternative attack paths
- Tool suggestions
- Corrections or improvements

Open an issue or PR!

---

## 📚 Resources & References

### Recommended Learning Paths
- [TryHackMe Offensive Pentesting Path](https://tryhackme.com/path/outline/pentesting)
- [HackTheBox Starting Point](https://app.hackthebox.com/starting-point)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)

### Essential Reading
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [GTFOBins](https://gtfobins.github.io/)
- [HackTricks](https://book.hacktricks.xyz/)
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)

### Tools & Frameworks
- [Metasploit Framework](https://www.metasploit.com/)
- [Burp Suite](https://portswigger.net/burp)
- [Kali Linux](https://www.kali.org/)

---

## 📈 Repository Stats

![GitHub last commit](https://img.shields.io/github/last-commit/igorleite97/offensive-writeups?style=flat-square)
![GitHub repo size](https://img.shields.io/github/repo-size/igorleite97/offensive-writeups?style=flat-square)
![GitHub stars](https://img.shields.io/github/stars/igorleite97/offensive-writeups?style=social)

---

<div align="center">

```
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│  "Every exploit begins with curiosity.                        │
│   Every compromise starts with enumeration.                   │
│   Every root shell is earned through persistence."            │
│                                                               │
│  — UmbraNull                                                  │
│                                                               │
│  Status: [ACTIVE] • Learning • Documenting • Sharing          │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

**Last Updated:** March 16, 2026  
**Maintained by:** [Igor Leite (UmbraNull)](https://github.com/igorleite97)

---

⭐ **If these write-ups helped you, star the repo and share the knowledge!**

**Happy Hacking!** 🎩🔐

</div>
