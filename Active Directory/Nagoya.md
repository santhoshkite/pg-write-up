# Nagoya — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Intermediate
**OS:** Windows (Active Directory)

---

## TL;DR

Enumerate `/Team` on IIS web server to build username list → Password spray to discover multiple valid AD credentials → Connect to SMB → Retrieve `ResetPassword.exe` binary → Reverse engineer in DNSpy to extract hardcoded `svc_helpdesk` credentials.

---

## Enumeration

```bash
nmap -sV -p- 192.168.244.21
```

**Open Ports:**
- **53, 88, 135, 139, 445**: Standard AD services
- **389, 636, 3268, 3269**: LDAP
- **3389**: RDP
- **5985**: WinRM
- **80**: HTTP (Microsoft IIS/10.0)

Domain: `nagoya-industries.com`
Hostname: `NAGOYA`

---

## Initial Discovery — OSINT and Credential Stuffing

Checking the HTTP service on port 80, we find the "Nagoya Industries" website.
Navigating to `http://192.168.244.21/Team`, we are presented with a list of employee names. 

Standard Active Directory environments often use predictable username formats. We compile the employee names into a `firstname.lastname` format to create a custom username list.

Using a tool like `Kerbrute` or `CrackMapExec`, we perform a light brute-force/password spray against the domain. We get lucky and discover several valid credentials, likely due to weak organizational password policies:
- `Craig.Carr:Spring2023`
- `Fiona.Clark:Summer2023`
- `Andrea.Hayes:Nagoya2023`
- `svc_mssql:Service1`

---

## Exploitation — Reversing Custom Binaries

With valid credentials, we can aggressively enumerate the SMB shares. 
```bash
crackmapexec smb 192.168.244.21 -u 'Craig.Carr' -p 'Spring2023' --shares
```

Deep in one of the readable SMB shares, we discover a custom executable named `ResetPassword.exe`. 
Custom IT or Helpdesk scripts placed on network shares are notorious for containing hardcoded credentials or insecure logic.

We download `ResetPassword.exe` and transfer it to a Windows analysis VM (or use tools natively on Kali) and load it into **dnSpy** (a .NET debugger and assembly editor).

Decompiling the Main function reveals hardcoded credentials for a service account:
`svc_helpdesk : U299iYRmikYTHDbPbxPoYYfa2j4x4cdg`

These credentials allow us to escalate our access and move laterally through the domain! *(Note: The exact path to Domain Admin varies based on the environment's BloodHound graph, but `svc_helpdesk` typically holds sweeping administrative reset rights).*

---

## Key Takeaways

- **Company 'About Us' Pages:** Always scrape public directories for employee names. They are essential for building tailored bruteforce and spraying lists.
- **SMB Scripts/Binaries:** If you find a `.bat`, `.ps1`, or `.exe` on a file share intended for "IT Use", immediately investigate it for hardcoded domain credentials. `dnSpy` is unparalleled for analyzing .NET binaries.

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
