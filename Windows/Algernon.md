# Algernon — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Easy
**OS:** Windows

---

## TL;DR

SmarterMail on port 9998 → anonymous FTP reveals admin username in logs → SmarterMail Build 6985 RCE exploit → NT AUTHORITY\SYSTEM.

---

## Enumeration

```bash
nmap -sV -p- 192.168.195.65
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | Microsoft ftpd (anonymous login!) |
| 80 | HTTP | IIS 10.0 |
| 135 | MSRPC | Microsoft Windows RPC |
| 139/445 | SMB | |
| 9998 | HTTP | IIS 10.0 (SmarterMail) |
| 17001 | Remoting | MS .NET Remoting |

**Anonymous FTP** reveals directories: `ImapRetrieval`, `Logs`, `PopRetrieval`, `Spool`. The log files contain the username `admin`.

**SmarterMail** is running on port 9998.

---

## Exploitation — SmarterMail RCE (EDB-49216)

```bash
searchsploit smartermail
```

The key exploit: **SmarterMail Build 6985 - Remote Code Execution** ([EDB-49216](https://www.exploit-db.com/exploits/49216))

Modify the IP and port in the exploit, then run:

```bash
python3 49216.py
```

No output appears, but the listener catches a shell — as **NT AUTHORITY\SYSTEM**. 🎉

---

## Key Takeaways

- **Anonymous FTP** can reveal usernames, directory structures, and configuration files
- **SmarterMail** has a critical RCE vulnerability that gives instant SYSTEM access
- The exploit may not print output — always have your listener ready before running

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
