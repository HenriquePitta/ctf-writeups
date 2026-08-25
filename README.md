# CTF Writeups

A collection of writeups from CTF machines and rooms (Hack The Box, TryHackMe) — exploitation techniques, privilege escalation, and methodology notes.

## About

Cybersecurity Engineer with 3+ years of experience in defense and critical infrastructure environments, transitioning into Penetration Testing with a focus on infrastructure/Active Directory and web application security. Strong foundation in vulnerability management, malware analysis, and system hardening (DISA STIG) provides a defender's perspective on attack surfaces and exploitation paths. Hands-on experience with Windows PKI/AD Certificate Services, Palo Alto firewall management, and VMware infrastructure. 

## Writeup Index

### Hack The Box

| Machine | Difficulty | Initial Access Vector | Privilege Escalation | Date |
|---|---|---|---|---|
| [Reactor](writeups/htb/reactor.md) | Easy | Unauthenticated RCE (CVE-2025-55182 — React Server Components) | Exposed Node.js Inspector (localhost:9229) | 2026-08-15 |
| [Cohort](writeups/htb/cohort.md) | Medium | SSRF (decimal IP bypass) → marimo pre-auth RCE (CVE-2026-39987) | PackageKit TOCTOU LPE (CVE-2026-41651) | 2026-08-25 |
| [DanglingTree](writeups/htb/danglingtree.md) | Hard | WAC Broken Access Control + SmarterMail CVE-2026-23760 | AD CS ESC4 (ForceChangePassword chain) | 2026-08-19 |

### TryHackMe

<div align="center">

[![TryHackMe](https://tryhackme-badges.s3.amazonaws.com/henriquepitta.png)](https://tryhackme.com/p/henriquepitta)

![](https://img.shields.io/badge/Top%203%25-105%20rooms%20completed-red?style=flat-square&logo=tryhackme)

</div>

| Room | Topic | Date |
|---|---|---|
| *(writeups to be added)* | | |

## General Methodology

The approach followed in most writeups:

1. **Reconnaissance** — `nmap`, service and version enumeration
2. **Web enumeration** — JS bundle/framework analysis, `gobuster`, hidden route discovery
3. **Vulnerability identification** — correlating exact software versions with known CVEs
4. **Exploitation** — initial access via public exploit or manual technique
5. **Post-exploitation** — local enumeration, credential hunting, pivoting
6. **Privilege escalation** — SUID analysis, cron jobs, root-owned services, special groups (lxd, docker, etc.)

## Repository Structure

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

All writeups refer to authorized CTF machines and environments designed for educational purposes (Hack The Box, TryHackMe). No technique described here should be used against systems without explicit authorization.
