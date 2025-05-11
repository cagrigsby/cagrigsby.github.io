---
layout: writeup
title: Timelapse - HackTheBox
date: 2024-08-11 13:32:20 +0300
description: A Writeup of the Timelapse Box from HackTheBox
image: # /assets/images/Timelapse/Timelapse.png
fig-caption: # Add figcaption (optional)
tags: [TJ Null, LainKusanagi, Windows, Active Directory]
---

![Timelapse1.png](/assets/images/Timelapse/Timelapse1.png){: .responsive-image}

Today I'm doing a writeup for a [Hack The Box](https://app.hackthebox.com/profile/2013658) box from both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)and LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called Timelapse, and it is rated Easy by HackTheBox. As usual, we get started with an nmap scan. I'm using my own [custom script](https://github.com/pentestpop/verybasicenum/blob/main/vbnmap.sh) for this which (gives more detail, but the initial scan) shows these open ports:

```
PORT     STATE SERVICE
53/tcp   open  domain
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

So no web port, but it looks like we are able to list SMB shares:

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share 
Shares          Disk      
SYSVOL          Disk      Logon server share 
```

I access shares and download everything with `prompt off`, `recurse on`, `mget *`. 

![Timelapse2.png](/assets/images/Timelapse/Timelapse2.png){: .responsive-image}

Looks like we have a zip file (password protected) and some documents. The documents appear to be Microsoft documentation, or they are at least stylized that way. It's possible some of the guidance on password complexity within them might be helpful, but nothing else stood out immediately so I decide to try and crack the password of the zip file. I run `zip2john winrm_backup.zip > winrm_backup.john` and then it cracks quickly as `supremelegacy`. Unzipping presents a new file called `legacyy_dev_auth.pfx` which is also password protected with a different password. I can run the same thing again, but this time with `pfx2john legacyy_dev_auth.pfx > legacyy_dev_auth.john`. Eventually it cracks to `thuglegacy`. Inside it looks like we have a key and a certificate. 

![Timelapse3.png](/assets/images/Timelapse/Timelapse3.png){: .responsive-image}

I'm not exactly sure what to do with these, so I google around a little bit, and I find this [linkedin post](https://www.linkedin.com/posts/kjohnson422_capturetheflag-oscp-htb-activity-7110979686423240704-T9tZ). It suggests that we can extract them and use them with `evil-winrm`. The commands are as follows:

1. Extracting the key file -> `openssl pkcs12 -in file.pfx -nocerts -out file.key-enc`
	1. This will ask for the password (`thuglegacy`), and it will ask you to set a new PEM pass phrase which can be whatever, but you will need it again. 
2. Extracting the certificate -> `openssl pkcs12 -in file.pfx -clcerts -nokeys -out file.crt` 
	1. This will ask for the password. 
3. Decrypting the key file -> `openssl rsa -in file.key-enc -out file.key` 
	1. This will request the PEM pass phrase you set already. 

I get to the end of the post and realize it seems to be for this box in particular. Oh well. Now we have legacyy.key and legacyy.crt, and we can use evil-winrm: `evil-winrm -i 10.10.11.152 -k legacyy.key -c legacyy.crt -S`. Note that the `-S` flag is for enabling ssl. This will allow you to use the key and certificate instead of authenticating with a user. If you leave it out, the command will fail and tell you that the user flag is required. So we get a shell. 

![Timelapse4.png](/assets/images/Timelapse/Timelapse4.png){: .responsive-image}

At this point I run a custom powershell script called [vbenum.ps1](https://github.com/pentestpop/verybasicenum/blob/main/vbenum.ps1) to run the enumeration commands I would run prior to something more extensive like winpeas, and a few things stick out, namely the history file:

```
==========================
PowerShell ConsoleHost History:
==========================
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -
SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
exit
```

It looks like we may have creds for `svc_deploy`:`E3R$Q62^12p7PLlC%KWaxuaV`. I can't use these credentials to log in, but I can see that the svc_deploy user is the the `LAPS_Readers` group, LAPS standing for "Local Administrator Password Solution". So that could definitely be something. Unfortunately when I google "'LAPS_Readers' Group", many of the results are for this machine, and I don't want to copy writeups (at least not more than I already accidentally have).

At this point I check my notes to see what I have for LAPS, and there's a few options for this. Basically the LAPS password can be grabbed using the `nxc ldap` module or there is also a tool called [pyLAPS](https://github.com/p0dalirius/pyLAPS). The commands are either: 
1. `nxc ldap timelapse.htb -u 'svc_deploy' -p 'E3R$Q62^12p7PLlC%KWaxuaV' --kdcHost timelapse.htb -M laps`
2. OR: `python3 pyLAPS.py --action get -u 'svc_deploy' -d 'timelapse.htb' -p 'E3R$Q62^12p7PLlC%KWaxuaV' --dc-ip 10.10.11.152`

In either case, they return a password:
![Timelapse5.png](/assets/images/Timelapse/Timelapse5.png){: .responsive-image}

![Timelapse6.png](/assets/images/Timelapse/Timelapse6.png){: .responsive-image}

So we have a password of: `4[y5h.I&6h01yogLwe3A,%C9`. Not exactly sure what to do with it, but I see on [hacktricks](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/laps) that I should be able to use it with Administrator for `xfreerdp` or `impacket-psexec`. I immediately tried `evil-winrm`, but that didn't work. RDP isn't open, so that won't work, but hopefully psexec will. I try it a few times, and it says it's starting the service, but it keeps dying so I try to reset the lab. FYI - if you do this, there will be a new LAPS password, the one above no longer worked. 

I still can't get psexec or evil-winrm working, so I go for `impacket-smbexec`, and I'm able to get a semi-interactive shell. 

![Timelapse7.png](/assets/images/Timelapse/Timelapse7.png){: .responsive-image}

This is enough to grab the root.txt flag (on `C:\Users\TRX\Desktop\root.txt`). I found this somewhat unsatisfying so I used the Admin shell to add a new user (`net user pop Ev!lpass /add`), and then add them to the administrators group (`net localgroup administrators pop /add`). I then tried to use winrm to log in with pop, and it didn't work. I know they were created and added because I can see them in the legacyy shell. 

![Timelapse8.png](/assets/images/Timelapse/Timelapse8.png){: .responsive-image}

Still couldn't log in. Then I added pop to the Remote Management Users group, and still couldn't use psexec or evil-winrm. So I felt like I got most of the way there and looked up if other people had the same problem, and they didn't because they all used `-S` to enable ssl for evil-winrm. In fact, I could have even logged in with the `svc_deploy` user had I used that flag. I assumed it was needed simply to use the key and certificate to log in, but it's actually because of the nature of the box. Evil-winrm uses port 5985 or 5986 to connect, but in this case, only port 5986 is open, and I didn't notice this. Usually they both are. So I could have used the LAPS key and `Administrator` user to log in without creating the pop user. Ironically, this might have made the whole thing take longer because I may not have focused on what I could do remotely with the `svc_deploy` user, which in this case was dump the LAPS password. 

Also I think impacket-psexec didn't work because of anti-virus. When I tried to upload reverse shell executables that didn't work either, and that is how psexec works. 

### Lessons Learned
- I got a referesher on using `zip2john` and `pfx2john`. 
- I used a key and cert with `evil-winrm`. 
- I used both nxc and pyLAPS to dump the LAPS password.
- I learned that the LAPS passwords are re-generated rather than set once. 
- I learned the evil-winrm over port 5986 requires the -S flag. 

### Remediation Steps
- Remove the `winrm-backup.zip` from the publicly accessible share as it is crackable and contains the key and certificate used to access the machine over winrm. 
- Consider disabling powershell history as plain text passwords should not be available there. 
- Consider disabling the svc_deploy user's ability to read the LAPS password as it can be done remotely. It's unclear what the use case is for it to be enabled. 