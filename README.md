<h1 align="center">
  <br>
  🛡️ Proving Grounds Writeups 🛡️
  <br>
</h1>

<h4 align="center">A comprehensive collection of detailed walkthroughs for OffSec's Proving Grounds Practice.</h4>

<p align="center">
  <a href="#about">About</a> •
  <a href="#contents">Contents</a> •
  <a href="#highlights">Highlights</a> •
  <a href="#disclaimer">Disclaimer</a>
</p>

---

## 📖 About

This repository serves as a personal knowledge base and showcase of my work working through **OffSec's Proving Grounds (PG) Practice** labs. It contains step-by-step, highly detailed writeups demonstrating enumeration, exploitation, and privilege escalation techniques.

Whether you are preparing for the OSCP or just looking to sharpen your penetration testing skills, these writeups document practical methodologies, custom exploit adaptations, and lateral movement strategies in both Linux and Windows environments (including Active Directory).

---

## 📂 Contents

The writeups are cleanly organized by operating system and environment type:

### 🐧 Linux
A deep dive into Linux privilege escalation, exploiting outdated CMS platforms, taking advantage of misconfigured cron jobs, cracking weak SSH keys, and escaping restricted shells.

### 🪟 Windows
Exploiting IIS web servers, manipulating malicious Service Principal Names (SPNs), breaking through weak SMB shares, and leveraging classic Windows privilege escalation techniques like `SeImpersonatePrivilege` (PrintSpoofer/GodPotato/JuicyPotato) and `AlwaysInstallElevated`.

### 🏢 Active Directory
Advanced Windows environments focusing on domain enumeration (BloodHound/enum4linux), Kerberoasting, AS-REP Roasting, Resource-Based Constrained Delegation (RBCD) attacks, Group Managed Service Account (gMSA) abuse, and manipulating Group Policy Objects (GPOs) to achieve Domain Admin.

---

## 🎯 Highlights & Methodologies

Across these boxes, some key techniques and vulnerabilities covered include:

- **Web Exploitation:** Local/Remote File Inclusion (LFI/RFI), SQL Injections, SSRF to NTLM theft, CMS vulnerabilities (WordPress, Subrion, ManageEngine).
- **Network Attacks:** SMB relaying, anonymous FTP abuse, LDAP anonymous binding data leaks.
- **Privilege Escalation (Linux):** SUID/SGID binaries, `sudo` misconfigurations (tar wildcards), writable `/etc/passwd`.
- **Privilege Escalation (Windows):** Token impersonation (`SeImpersonate`, `SeRestore`), unquoted service paths, unencrypted `.ini`/`.xml` configurations, LAPS password extraction.
- **Active Directory:** Kerberos ticket manipulation, SharpGPOAbuse, offline NTDS.dit extraction.

---

## ⚠️ Disclaimer

All information and writeups in this repository are for **educational purposes only**. The techniques demonstrated here should only be used on systems you own or have explicit, documented permission to test (such as the OffSec Proving Grounds environment). Do not use these techniques for malicious purposes.

---

<p align="center">
  <i>"The only truly secure system is one that is powered off, cast in a block of concrete and sealed in a lead-lined room with armed guards."</i><br>— Eugene H. Spafford
</p>
