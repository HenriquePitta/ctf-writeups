# CTF Writeups

Coleção de writeups de máquinas e salas de CTF (Hack The Box, TryHackMe) — técnicas de exploração, escalação de privilégios, e notas de metodologia.

## Sobre

Cibersegurança / pentest, com foco em infraestrutura, aplicações web e automação. CET em Cibersegurança (ATEC). Experiência prática com Linux (RHEL, Ubuntu, Kali), VMware, e ferramentas de scanning/compliance (Nessus, OpenSCAP, SCC).

## Índice de Writeups

### Hack The Box

| Máquina | Dificuldade | Vetor de Acesso Inicial | Escalação | Data |
|---|---|---|---|---|
| [Reactor](writeups/htb/reactor.md) | — | RCE não autenticado (CVE-2025-55182 — React Server Components) | Node.js Inspector exposto (localhost:9229) | 2026-08-15 |

### TryHackMe

| Sala | Tópico | Data |
|---|---|---|
| _(por adicionar)_ | | |

## Metodologia Geral

A abordagem seguida na maioria dos writeups:

1. **Reconhecimento** — `nmap`, enumeração de serviços e versões
2. **Enumeração web** — análise de bundles JS/frameworks, `gobuster`, procura de rotas escondidas
3. **Identificação de vulnerabilidades** — correlação de versões de software com CVEs conhecidos
4. **Exploração** — acesso inicial via exploit público ou técnica manual
5. **Pós-exploração** — enumeração local, procura de credenciais, pivoting
6. **Escalação de privilégios** — análise de SUID, cron jobs, serviços root, grupos especiais (lxd, docker, etc.)

## Estrutura do Repositório

```
.
├── README.md
└── writeups/
    ├── htb/
    │   └── reactor.md
    ├── thm/
    └── template.md
```

## Disclaimer

Todos os writeups referem-se a máquinas e ambientes de CTF autorizados e projetados para fins educativos (Hack The Box, TryHackMe). Nenhuma técnica aqui descrita deve ser usada contra sistemas sem autorização explícita.
