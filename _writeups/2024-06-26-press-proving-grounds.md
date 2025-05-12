---
layout: writeup
title: Press - Proving Grounds
date: 2024-06-26 13:32:20 +0300
description: A Writeup of the Press Box from Proving Grounds
image: # /assets/images/Press/Press_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, TJ Null, FlatPress, apt]
---

Here is a writeup for the Press lab on [Proving Grounds](https://www.offsec.com/labs/), another box from TJ Null's OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#). As usual we kick things off with an nmap scan using the `-v` flag so we can see some results before they are finished. We notice port 80 is open, so we check it out. 
![Press_1.png](/assets/images/Press/Press_1.png){: .responsive-image}

I actually don't see much on this site. The Contact page doesn't seem to do anything, and the Product Details tab links only to one product, which is also where all the products listed in the Shop tab link to. Fortunately we have the scan running and we see port 8089 running as well, so we check that out. 

![Press_2.png](/assets/images/Press/Press_2.png){: .responsive-image}

This looks like a simple blog CMS, and it also says it's powered by FlatPress. I run a directory scan on it and take a quick peek into the source to see if I can find a version. 

![Press_3.png](/assets/images/Press/Press_3.png){: .responsive-image}

Looks like we may have version 1.2.1. That could help us if we find any public exploits. I also see the admin page, and I figured I could try to guess my way in quickly. I try `admin:admin` which doesn't work, but `admin:password` does. Nice! Though that probably just means that the real work of the box hasn't started yet. For a web service like this, I figure I can maybe find some authenticated exploits or even just edit a page to execute php. 

![Press_4.png](/assets/images/Press/Press_4.png){: .responsive-image}

I tried an RCE exploit from exploit-db, but I'm not able to get it working, and I notice another which requires a specific plugin which is not enabled. I notice there is a tab called "Uploader" so maybe I can find something there. Intially I can't upload a .php file, but I can upload a .txt file. Maybe I can find a way to upload the .php file anyway...

I tried to change the MIME type and change the file name a few different ways, but I couldn't get it working. Then I went back to this exploit (https://github.com/flatpressblog/flatpress/issues/152) I'd previously found on github and read it more carefully. Initially it seemed to say that you could just upload a .php file to `http://192.168.174.29:8089/admin.php?p=uploader&action=default`, but it actually make it clear that you need to add `GIF89a;` to the very top of the file. Oops. I add that to an Ivan Sincek reverse php shell from revshells.com and upload it. 

![Press_5.png](/assets/images/Press/Press_5.png){: .responsive-image}

And we get our shell. Cool. I check the other files in the web directory and head over to the /tmp directory to start downloading my favorite enumeration scripts., but before I do, I run `sudo -l` to see if I can run anything as sudo and I see this: 

![Press_6.png](/assets/images/Press/Press_6.png){: .responsive-image}

Apt-get it is. I check gtfobins, and I see something to try. First run `sudo apt-get changelog apt` then hit return to accept, then `!/bin/sh` and you have a root shell. 

![Press_7.png](/assets/images/Press/Press_7.png){: .responsive-image}

I grab the proof.txt file and that's all folks. Lessons learned - read public exploits carefully, they may have something specific in them that isn't readily apparent when you just look at the title. All in all, didn't take too long either way, but it could have been faster. 

