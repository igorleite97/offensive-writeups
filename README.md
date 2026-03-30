# Offensive Security Write-Ups

<div align="center">

[![TryHackMe](https://img.shields.io/badge/TryHackMe-UmbraNull-212C42?style=flat-square&logo=tryhackme&logoColor=white)](https://tryhackme.com/p/UmbraNull)
[![HackTheBox](https://img.shields.io/badge/HackTheBox-UmbraNull-9FEF00?style=flat-square&logo=hackthebox&logoColor=black)](https://app.hackthebox.com/profile/019c4c95-6170-73b2-b61c-c5741403d621)
[![GitHub](https://img.shields.io/badge/Portfolio-igorleite97-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/igorleite97)

![Status](https://img.shields.io/badge/Status-Active-success?style=flat-square)
![Rooms](https://img.shields.io/badge/Rooms_Completed-12-blue?style=flat-square)
![Last Updated](https://img.shields.io/github/last-commit/igorleite97/offensive-writeups?style=flat-square&label=Last+Updated)

</div>

---

## About This Repository

Documentação técnica de máquinas vulneráveis exploradas em plataformas de treinamento controladas (TryHackMe e HackTheBox). Cada write-up registra a cadeia de ataque completa — do reconhecimento inicial à escalada de privilégio — com foco no raciocínio por trás de cada decisão técnica, não apenas nos comandos executados.

O repositório funciona como portfólio prático de segurança ofensiva e base de conhecimento pessoal, documentando técnicas aplicadas, ferramentas utilizadas e lições aprendidas em cada engajamento.

---

## Methodology

Cada write-up segue uma estrutura consistente inspirada em relatórios de pentest:

```
Machine Information   →  plataforma, OS, dificuldade
Attack Chain          →  visão geral da cadeia de comprometimento
Enumeration           →  reconhecimento de superfície de ataque
Initial Access        →  vetor de entrada e raciocínio ofensivo
Post-Exploitation     →  coleta de inteligência pós-acesso
Privilege Escalation  →  técnica de elevação de privilégio
Flags                 →  evidências de comprometimento
Lições Aprendidas     →  análise técnica e síntese
Tools Used            →  arsenal aplicado
References            →  fontes e CVEs relevantes
```

---

## Completed Rooms

### TryHackMe

| Room | Difficulty | OS | Key Techniques | Write-up |
|------|------------|-----|----------------|----------|
| [Agent Sudo](tryhackme/agent-sudo/) | Easy | Linux | User-Agent enumeration, steganography (binwalk/steghide), zip2john, CVE-2019-14287 | ✅ |
| [Startup](tryhackme/startup/) | Easy | Linux | FTP anonymous write + webshell RCE, PCAP credential extraction, cron privesc | ✅ |
| [Break Out The Cage](tryhackme/break-out-the-cage/) | Easy | Linux | Base64 + Vigenère cipher, Python command injection via os.system(), cron exploitation | ✅ |
| [Brooklyn Nine Nine](tryhackme/brooklyn-nine-nine/) | Easy | Linux | FTP anonymous, SSH brute-force (Hydra), SUID less (GTFOBins) | ✅ |
| [Kenobi](tryhackme/kenobi/) | Easy | Linux | ProFTPD mod_copy (CVE-2015-3306), NFS misconfiguration, PATH hijacking | ✅ |
| [Mr. Robot CTF](tryhackme/mr-robot-ctf/) | Medium | Linux | — |  |
| [Wonderland](tryhackme/wonderland/) | Medium | Linux | — |  |
| [Attacktive Directory](tryhackme/attacktive-directory/) | Medium | Windows | — |  |
| [Basic Pentesting](tryhackme/basic-pentesting/) | Easy | Linux | SMB enumeration, SSH brute-force, SSH key cracking | ✅ |
| [Simple CTF](tryhackme/simple-ctf/) | Easy | Linux | CMS Made Simple SQLi, VIM GTFOBins | ✅ |
| [Lookup](tryhackme/lookup/) | Easy | Linux | — |  |
| [Basic Malware RE](other/basic-malware-re/) | — | — | — |  |

### HackTheBox

| Box | Difficulty | OS | Status |
|-----|------------|-----|--------|
| Lame | Easy | Linux | Pendente |
| Legacy | Easy | Windows | Pendente |
| Blue | Easy | Windows | Pendente |

---

## Skills Demonstrated

### Reconnaissance & Enumeration
- Port scanning e service fingerprinting (Nmap)
- Directory and content discovery (Gobuster, Feroxbuster)
- SMB/NFS enumeration (enum4linux, smbclient)
- Custom enumeration scripts in Python

### Web Exploitation
- SQL Injection (error-based, time-based blind)
- CMS exploitation (WordPress, CMS Made Simple)
- File upload vulnerabilities and webshell deployment
- Authentication bypass techniques

### Credential Attacks
- Hash cracking (John the Ripper, Hashcat)
- Password brute-force (Hydra)
- Credential extraction from network captures (PCAP analysis)
- SSH key theft and exploitation

### Privilege Escalation
- SUID binary exploitation (GTFOBins)
- Sudo misconfiguration abuse
- PATH hijacking in privileged scripts
- Cron job exploitation (writeable scripts)
- Linux capabilities abuse (cap_setuid)
- CVE-based escalation (CVE-2019-14287, CVE-2015-3306)

### Active Directory
- Kerberos enumeration
- AS-REP Roasting
- DCSync attack

---

## Offensive Toolkit

| Category | Tools |
|----------|-------|
| Scanning | `nmap`, `masscan`, `autorecon` |
| Web Discovery | `gobuster`, `feroxbuster`, `ffuf`, `nikto` |
| SMB/AD | `enum4linux`, `smbclient`, `impacket`, `bloodhound` |
| Exploitation | `metasploit`, `searchsploit`, `burp suite`, `sqlmap` |
| Credential Attacks | `john`, `hashcat`, `hydra` |
| Post-Exploitation | `linpeas`, `winpeas`, `crackmapexec` |
| Steganography | `binwalk`, `steghide`, `stegseek` |
| Scripting | `python3`, `bash` |

---

## Write-Up Structure

```
tryhackme/
└── room-name/
    ├── writeup.md       ← documentação principal
    ├── scripts/         ← exploits e scripts customizados
    └── notes/           ← dados brutos de enumeração
```

---

## Certification Path

| Certification | Issuer | Status |
|---------------|--------|--------|
| eJPT | INE Security | Expected 2026 |
| ISO/IEC 27001:2022 Associate | SkillFront | Completed 2026 |
| Getting Started in Cybersecurity 3.0 | Fortinet | Completed 2026 |
| CRPO | EU Cyber Academy | Completed 2026 |
| OSCP | Offensive Security | Planned |

---

## References

- [GTFOBins](https://gtfobins.github.io) — SUID/sudo binary exploitation reference
- [HackTricks](https://book.hacktricks.xyz) — Privilege escalation and attack techniques
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) — Payload reference
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/) — Web security testing methodology
- [Exploit-DB](https://www.exploit-db.com) — CVE and exploit database

---

## Legal Notice

All activities documented in this repository were performed exclusively in authorized training environments (TryHackMe, HackTheBox) or personal lab setups. No systems were tested without explicit permission from platform owners.

Unauthorized access to computer systems is illegal. This repository exists for educational and professional development purposes only.

---

*Igor Leite (UmbraNull) · [linkedin.com/in/igor-leite-a9b839222](https://www.linkedin.com/in/igor-leite-a9b839222) · [github.com/igorleite97](https://github.com/igorleite97)*
