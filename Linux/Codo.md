# Codo — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Easy
**OS:** Linux

---

## TL;DR

CodoForum v5.1 with default credentials → manual PHP reverse shell upload via logo upload → config file leaks root password → `su root`.

---

## Enumeration

Full port scan:

```bash
sudo nmap -sC -p- -n -Pn -sV --min-rate=9362 192.168.203.23 -oN nmap.txt
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.2p1 Ubuntu |
| 80 | HTTP | Apache httpd 2.4.41 |

The web server is running **CodoForum** (we can see "CODOLOGIC" in the title). Gobuster reveals an `/admin` panel:

```bash
gobuster dir -u http://192.168.203.23/ -w `/usr/share/wordlists/dirb/common.txt`
```

Key findings:
- `/admin` — Admin panel (301)
- `/cache` — Cache directory (301)
- `/sites` — Sites directory (301)

---

## Exploitation — Default Creds + PHP Shell Upload

Tried the default credentials on the admin panel: `admin:admin` — and they work. The CodoForum version is **v5.1.105**.

There are known RCE exploits for CodoForum v5.1:
- [EDB-50978 — CodoForum v5.1 RCE](https://www.exploit-db.com/exploits/50978)
- [CVE-2022-31854](https://github.com/Vikaran101/CVE-2022-31854)

Both automated exploits failed, so we go manual:

1. Log into the admin panel at `http://192.168.203.23/admin/`
2. Navigate to **Global Settings**
3. Find the **"Upload logo for your forum"** option
4. Upload a [PHP reverse shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) (remember to edit the IP and port in the shell)
5. Trigger the payload by navigating to:

```
http://192.168.203.23/sites/default/assets/img/attachments/php-reverse-shell.php
```

With our nc listener running, we catch a shell.

---

## Privilege Escalation — Config File Password Reuse

After some enumeration on the box, we find the CodoForum database config:

```bash
cat /var/www/html/sites/default/config.php
```

This reveals the password: `FatPanda123`

Let's try it for root:

```bash
su root
# Password: FatPanda123
```

It works — **root**. 🎉

---

## Key Takeaways

- **Default credentials** strike again — `admin:admin` is the first thing you should always try
- When automated exploits fail, **manual exploitation** is the way — understanding what the exploit does lets you replicate it by hand
- **CMS config files** often contain database passwords that are reused for system accounts — always check `config.php`, `settings.py`, `.env`, etc.
- Logo/avatar upload features are commonly exploitable for file upload attacks

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
