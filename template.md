# [Machine Name] — TryHackMe Write-up

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-blue)
![OS](https://img.shields.io/badge/OS-Linux-lightgrey)
![Status](https://img.shields.io/badge/Status-Completed-success)

---

## Machine Information

| Attribute          | Details              |
|--------------------|----------------------|
| **Name**           | [Machine Name]       |
| **Platform**       | TryHackMe            |
| **Difficulty**     | Easy / Medium / Hard |
| **OS**             | Linux / Windows      |
| **Date Completed** | [Month Year]         |

---

## Attack Chain

```
[PHASE]     →  [tool/technique]  →  [result]
[PHASE]     →  [tool/technique]  →  [result]
```

---

## Enumeration

### Nmap

```bash
nmap -sC -sV -p- -T4 <IP> -oN nmap.txt
```

| Port   | Service | Version | Note |
|--------|---------|---------|------|
| xx/tcp | —       | —       | —    |

[Parágrafo curto em português explicando o raciocínio sobre os serviços encontrados.]

### [Other Enumeration — ex: Gobuster, SMB, NFS]

```bash
[command]
```

[Resultado relevante e o que ele significa para o ataque.]

---

## Initial Access

[Parágrafo em português descrevendo o vetor escolhido e por quê.]

```bash
[commands]
```

> **Note:** [Explicação técnica de um mecanismo relevante — ex: por que /dev/tcp funciona, por que binary mode no FTP, etc.]

---

## Post-Exploitation

[Parágrafo em português descrevendo o que foi encontrado e qual é a próxima superfície de ataque.]

```bash
[commands]
```

---

## Privilege Escalation

[Parágrafo em português descrevendo a vulnerabilidade identificada.]

```bash
[commands]
```

---

## Flags

| File         | Path       | Flag                  |
|--------------|------------|-----------------------|
| `user.txt`   | `/home/x/` | `[flag]`              |
| `root.txt`   | `/root/`   | `[flag]`              |

---

## Lições Aprendidas

[Parágrafo em português com as principais conclusões técnicas desta room. Foco no "por quê" das vulnerabilidades existirem, não apenas no "como" explorar. Máximo 3-4 pontos.]

```bash
# Comando de referência futura relevante
[command]
```

---

## Tools Used

| Tool       | Purpose                     |
|------------|-----------------------------|
| `nmap`     | Port scanning               |
| `[tool]`   | [purpose]                   |

---

## References

- [Resource Name](URL) — [one-line description]

---

*UmbraNull · [github.com/igorleite97](https://github.com/igorleite97) · offensive-writeups*
