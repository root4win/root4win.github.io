---
title: "AD Enumeration Cheat Sheet"
date: 2026-06-17
tags: [oscp, active-directory, pentest, cheat-sheet]
---

# Active Directory Enumeration Cheat Sheet (No Credentials)

Quick reference for initial AD enumeration.

---

# 1. PASSIVE RECON

## Traffic Analysis
- tcpdump / Wireshark  
  → ARP, mDNS, DHCP, NetBIOS traffic

## Name Poisoning Detection
- Responder (listen mode only)  
  → LLMNR / NBT-NS leaks (hostnames, users)

---

# 2. ACTIVE DISCOVERY

## Host Discovery
- fping -asgq <range>
- arp-scan <range>

## Service Scan
- nmap -sS -T4 -Pn -iL hosts.txt

---

## Domain Controller Identification

Look for:

- 88 → Kerberos
- 389 / 636 → LDAP
- 445 → SMB
- 53 → DNS

Rule:
88 + 389 + 445 = Domain Controller

---

# 3. LOW-IMPACT ENUMERATION

## SMB / LDAP checks
- enum4linux-ng <ip>
- smbclient -L //<ip>

## Nmap scripts
- smb-os-discovery
- smb-enum-shares
- ldap-rootdse

## LDAP anonymous test
- ldapsearch -x -H ldap://<ip>

---

# 4. USER ENUMERATION

## Kerberos user discovery
- kerbrute userenum -d <domain> users.txt --dc <ip>

## RID Brute Force
- SMB SID enumeration (if allowed)

---

# 5. INITIAL FOOTHOLD

## NTLM Capture
- Responder / Inveigh → NTLMv2 hashes

## Offline cracking
- hashcat / john

---

## Password Attacks
- nxc smb <ip> -u users.txt -p passwords.txt
- kerbrute password spray

---

## Legacy Exploits
- MS17-010 (EternalBlue)
- Legacy services (FTP / IIS / Tomcat)
- Web vulnerabilities

---

## External Intelligence
- leaked credentials (paste sites, dumps)
- password reuse
- username format guessing

---

# 6. WITH CREDENTIALS

## AD Enumeration
- BloodHound / SharpHound → attack paths
- PowerView → AD enumeration
- ldapdomaindump → LDAP extraction

---

# 7. POST-CREDS ATTACKS

- Kerberoasting
- AS-REP roasting
- ACL abuse (GenericAll / WriteDACL)
- DCSync
- Pass-the-Hash / Pass-the-Ticket

---

# QUICK RULES

- 88 open → likely Domain Controller
- SMB null session → immediate enum opportunity
- LLMNR/NBT-NS → always check first
- LDAP anonymous → high value
- Kerbrute → username validation only
