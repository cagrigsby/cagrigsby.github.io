---
layout: writeup
title: Authby - Proving Grounds
date: 2024-09-13 13:32:20 +0300
description: A Writeup of the Authby Box from Proving Grounds
image: # /assets/images/Authby/Authby.png
fig-caption: # Add figcaption (optional)
tags: [Windows, TJ Null, LainKusanagi, MS11-046]
---

Today I'm doing a writeup for another [Proving Grounds](https://www.offsec.com/labs/) box from TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)as well as LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called Authby, and it is rated Intermediate by Proving Grounds and Hard by the community. As usual, we get started with an nmap scan: 

```
21/tcp   open  ftp                zFTPServer 6.0 build 2011-10-17
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| total 9680

...

242/tcp  open  http               Apache httpd 2.2.21 ((Win32) PHP/5.3.8)
| http-auth: 
| HTTP/1.1 401 Authorization Required\x0D
|_  Basic realm=Qui e nuce nuculeum esse volt, frangit nucem!
|_http-title: 401 Authorization Required
|_http-server-header: Apache/2.2.21 (Win32) PHP/5.3.8
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
3145/tcp open  zftp-admin         zFTPServer admin
3389/tcp open  ssl/ms-wbt-server?
| rdp-ntlm-info: 
|   Target_Name: LIVDA
|   NetBIOS_Domain_Name: LIVDA
|   NetBIOS_Computer_Name: LIVDA
|   DNS_Domain_Name: LIVDA
|   DNS_Computer_Name: LIVDA
|   Product_Version: 6.0.6001
|_  System_Time: 2024-09-02T21:03:38+00:00
|_ssl-date: 2024-09-02T21:03:42+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=LIVDA
| Issuer: commonName=LIVDA
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2024-08-01T06:37:13
| Not valid after:  2025-01-31T06:37:13
| MD5:   b0a3:8bda:dc42:e14c:5f17:f1ea:0576:376c
|_SHA-1: 7c1b:5cdf:fb7b:828b:225a:15a5:6f70:0c0c:0001:821e
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

```

First thing that pops up when I ran an nmap scan with the `-v` flag was port 21, so I tried to log in to ftp with `anonymous:anonymous`, and it worked:
![Authby1.png](/assets/images/Authby/Authby1.png){: .responsive-image}

I can't see anything in the first couple directories, but I do see this in accounts: 
![Authby2.png](/assets/images/Authby/Authby2.png){: .responsive-image}

I can't retrieve anything in this FTP server, but I can see accounts here which might be helpful. At this point I go check out the other open ports, namely 242 and 3145, but I can't actually do anything with them because 242 requires Auth and 3145 hangs. I tried to log back into the the FTP server using `admin:admin` because admin was listed in the accounts, and it worked, showing this:

![Authby3.png](/assets/images/Authby/Authby3.png){: .responsive-image}

Because I see the `index.php` file, I'm wondering this is is a root directory, and I could potentially put another file there and execute it from the browser. But I also download these files, and check them out. The .htaccess shows:
```
AuthName "Qui e nuce nuculeum esse volt, frangit nucem!"
AuthType Basic
AuthUserFile c:\\wamp\www\.htpasswd
<Limit GET POST PUT>
Require valid-user
</Limit>   
```
And the .htpasswd file says: `offsec:$apr1$oRfRsc/K$UpYpplHDlaemqseM39Ugg0`. So it looks like we have a password or a hash anyway (it's a hash). I use `nth` (NameThatHash) to determine that it is a MD5 hash and run hashcat to crack it. The credentials are `offsec:elite`. That won't let me log into the FTP server again, but it's fine because I can log into the basic Auth on port 242. That takes me to this page which fits with the description from nmap (and index.php): ![Authby4.png](/assets/images/Authby/Authby4.png){: .responsive-image}
I am also able to put a php reverse shell in to the ftp server and access it over this port in the browser as expected.

![Authby5.png](/assets/images/Authby/Authby5.png){: .responsive-image}

And I get a shell. I further notice after running `whoami /priv` that we have `SeImpersonatePrivilege`. I spend some time trying a few different Potato attacks, but can't get any of them to actually do anything. When I run PrintSpoofer (64-bit), I at least get this error: `This version of C:\wamp\www\PrintSpoofer.exe is not compatible with the version of Windows you're running. Check your computer's system information to see whether you need a x86 (32-bit) or x64 (64-bit) version of the program, and then contact the software publisher.` Most of them give no output at all. When I run `systeminfo`, I do get `System Type: X86-based PC`, but it doesn't help me with any of the Potato attacks to exploit the SeImpersonatePrivilege. Weirdly, I also can't seem to run winpeas either. It would make me think I can't execute anything, but the PrintSpoofer did give me some output. 

At that point I actually tested the msfvenom stageless windows shell on its own just to see if I could use it, and I couldn't, getting the same error as I did for PrintSpoofer above. I also realized the machine did not have nc.exe on it. So I uploaded that, and I was able to use it over port 21. This proved that I was able to execute .exe's, and I wasn't hindered by a firewall. 

At this point, I'm a little confused about what is oging on because I've tried several versions of Winpeas, and none of them have done anything. So I check the Windows version and find that it's pretty old. 

![Authby6.png](/assets/images/Authby/Authby6.png){: .responsive-image}

When we search for "windows Version 6.0.6001 exploit" in google, we notice a result from [exploit-db](https://www.exploit-db.com/exploits/40564)which clarifies that the vulnerability is `MS11-046`. I actually transferred this exploit from my computer to the target using the FTP service, but it didn't execute immediately. I googled around and found a .exe rather than a .c file [here in a github repo](https://github.com/SecWiki/windows-kernel-exploits/blob/master/MS11-046/ms11-046.exe), so I downloaded that, and it worked. 

![Authby7.png](/assets/images/Authby/Authby7.png){: .responsive-image}

Done. 

### Lessons Learned
Sign into FTP with each user you can because there might be different information available, and make sure to check the OS version. I understated how much time I wasted by failing to do this, because I simply didn't expect such an old version for a lab like this. I'm specifically studying for the OSCP here, and I can be pretty sure that there won't be a Windows 6 machine with a privesc vector like this, but that's no excuse to spend hours not checking the OS version. In real life, where I ultimately want to be pentesting, it may be much more possible. 

### Remediation Steps
- Add stronger passwords to the FTP server. You shouldn't allow anonymous login if it can be avoided, and certainly shouldn't put anything sensitive behind `admin:admin`. 
- In this case I was able to crack the offsec password based on what I found in the FTP server. That password should be stronger, and it shouldn't be accessible in the FTP server, even behind stronger creds. Granted I did have to crack it, and that would have been harder with a better password, but we should assume passwords can be cracked if they can be found. 
- Disallow uploads to the FTP server if possible, granted it was for an admin user, so it may not be. Certainly you should not be able to upload executable files to a webroot directory that is accessible from the internet. 
- Update the Windows version. 