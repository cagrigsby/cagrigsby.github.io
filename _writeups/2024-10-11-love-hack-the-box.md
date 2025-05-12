---
layout: writeup
title: Love - HackTheBox
date: 2024-10-11 13:32:20 +0300
description: A Writeup of the Love Box from HackTheBox
image: # /assets/images/Love/Love.png
fig-caption: # Add figcaption (optional)
tags: [LainKusanagi, Windows]
---

Today I'm doing a writeup for a [Hack The Box](https://app.hackthebox.com/profile/2013658) box from LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called OpenAdmin, and it is rated Easy by HackTheBox. As usual, we get started with an nmap scan. I'm using my own [custom script](https://github.com/pentestpop/verybasicenum/blob/main/vbnmap.sh) for this which (gives more detail but) shows these open ports:

```
PORT     STATE SERVICE
80/tcp   open  http
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
443/tcp  open  https
445/tcp  open  microsoft-ds
3306/tcp open  mysql
5000/tcp open  upnp
```

Looks like we may be dealing with a web app here, though of course I notice the 445 (can't see anything without authentication) and 3306 (same) ports open as well. But I check out port 80 in the browser to see if there's anything interesting, and we do see an app called "Voting System." I note in particular that the source code refers to it as "Voting System using PHP."

![Love2.png](/assets/images/Love/Love2.png){: .responsive-image}

I check exploit-db for Voting System, and I see this [unauthenticated RCE](https://www.exploit-db.com/exploits/50088) which looks promising. I read through it and fiddle with a few lines which direct the http requests to `/Online_voting_system/admin/` when the actual lab just says `/admin`, and I think I get it working, but I get this 404 error when requesting the uploaded shell:

![Love3.png](/assets/images/Love/Love3.png){: .responsive-image}

I check through the code again, which is ultimately uploading a shell to `/admin/upload` after my first changes, but when checking the browser, it doesn't seem like this folder exists. I run feroxbuster, and I can't find an upload directory in the output. 

So I go back through my nmap output, and I find a few other ports to check and a domain name. I check ports 443 (Forbidden) and 5000 (also Forbidden) and then add the domain name `staging.love.htb` to the `/etc/hosts` file and check the browser again. Oops. Looks like we've got a VHOST on our hands. 

![Love4.png](/assets/images/Love/Love4.png){: .responsive-image}

It says it's not live so I go to the demo page, and I see they offer to scan a file from a URL. So I load up responder and enter my own url. 

![Love5.png](/assets/images/Love/Love5.png){: .responsive-image}

Doesn't work. I get a `401 - Unauthorized: Access is denied due to invalid credentials` response when running responder, not sure why. It does download a real file, but it simply prints the contents to the page like so (the test.txt file says 'whoami' after I created it):

![Love6.png](/assets/images/Love/Love6.png){: .responsive-image}


I tried calling the files possibly uploaded from the previous exploit and calling the php files of the website itself, but they didn't print. It also didn't try to authenticate when I submitted my own smb share link. I tried a few directory traversal possibilities here, but I couldn't get any of them working. One of them even seemed to pull up the application from `http://10.10.10.239`.

![Love7.png](/assets/images/Love/Love7.png){: .responsive-image}

Eventually I tried localhost ports that hadn't worked before. Port 443 didn't get anything, but submitting `127.0.0.1:5000` returned this:

![Love8.png](/assets/images/Love/Love8.png){: .responsive-image}

Perfect. So I have creds to try for the application, maybe even for the machine. I saw some authenticated exploits in exploit-db at the beginning, but I figure I might as well log in and see if I can find something quick like a file upload reverse shell or something. I click around and find a `Add New Candidate` button with a photo upload section. Bingo. 

![Love9.png](/assets/images/Love/Love9.png){: .responsive-image}

It won't upload unless I select a position, and there are none available, so I have to add one, then go back and upload the [Ivan Sincek](https://github.com/ivan-sincek/php-reverse-shell)reverse shell as the photo. 

![Love10.png](/assets/images/Love/Love10.png){: .responsive-image}

I then right-click the broken image icon to open it in a new tab, and pop a shell on my listener. 

![Love11.png](/assets/images/Love/Love11.png){: .responsive-image}

I'm phoebe. I check the usual paths, run winpeas, and see this:

![Love12.png](/assets/images/Love/Love12.png){: .responsive-image}

After refreshing my memory with the link, I remember it means I can install any `.msi` file as `NT AUTHORITY\SYSTEM`. I create an `.msi` file with `msfvenom -p windows/adduser USER=poppop PASS=P@ssword123! -f msi -o poppop.msi` and transfer it to the machine. After that it's as simple as running it, which adds the user poppop to the Administrators group. Then I can use `evil-winrm -i 10.10.10.239 -u poppop -p 'P@ssword123!'` to get a shell on the machine (My extended results showed port 5985 open).

![Love13.png](/assets/images/Love/Love13.png){: .responsive-image}

And we're done. 

### Lessons Learned
- It took too long for me to remember to re-check the localhost and ports 443 and 5000 in the file upload vulnerability. Easy enough to remember to point it back to the target, but I should have checked the other ports faster. 
- It took too long to add the domain name to the /etc/hosts file to be honest. I run my [vbnmap script](https://github.com/pentestpop/verybasicenum/blob/main/vbnmap.sh)to first run a basic TCP scan before running full port and UDP scans so that I can get started before the whole thing is done, but I gotta remember to go back quickly and check it. Instead I got too far down the "Voting System" path before checking. 
- Gotta remember to use rockyou.txt for cracking as well. 

### Remediation Steps
This is kind of a CTF-y box, it's not like there's any real use case to print the ssh key *and* a hint for the password to a web app. So let's just start with:
- Removing the ssh key and password hint from the web app. 
- Probably the Voting System web app should be patched. Ultimately we didn't wind up using a known exploit, but we could have. 
- Don't allow php files to be uploaded as images to the web app, and don't allow them to be executed. There's probably a few ways of accomplishing that. 
- Maybe the file malware checker should be prevented from accepting a localhost address from being input. It wouldn't really have made a difference without the ssh key being printed, but it's probably not best practice. 
- The phoebe user should not be able to installed every .msi file as elevated. 