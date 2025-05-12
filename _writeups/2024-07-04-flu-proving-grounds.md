---
layout: writeup
title: Flu - Proving Grounds
date: 2024-07-04 13:32:20 +0300
description: A Writeup of the Flu Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi, Confluence]
---

Here is a writeup for the Flu lab on [Proving Grounds](https://www.offsec.com/labs/). This one actually isn't on TJ Null's OSCP prep list; it's on another list from LainKusanagi's [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). Still on Proving Grounds, but I think I may have ran through TJ Null's list. Maybe I will go back over them and write up some of the machine's I've completed but not bothered to put on here. As usual, I got started with an nmap scan which finds port 8090 open, so I go to my browser and check out a Confluence page. 

![Flu_1.png](/assets/images/Flu/Flu_1.png){: .responsive-image}

Note the `Powered by Atlassian Confluence 7.13.6` in the bottom. After searching around for an exploit, we find a few Confluence exploits, but zero in on one for version 7.13.6 specifically. Initially we actually find one linked in a blog that apparently disappeared, but by searching the CVE in particular, we find this exploit here: https://github.com/nxtexploit/CVE-2022-26134. We can clone the repository, read the exploit, test the command with `id` and eventually using the busybox shell to get a reverse shell. 

![Flu_2.png](/assets/images/Flu/Flu_2.png){: .responsive-image}

Nice! Got the shell. I look around a bit for the local flag initially before realizing that there might be a directory for our user in `/home` and grab the local txt there. At that point, I checked for SUID binaries to exploit and ran `sudo -l` just in case. Neither of those worked out, so I went to the temp directory and downloaded some auto exploit scripts to try and grab any low hanging fruit. I've been enjoying `lse.sh` ([linux smart enumeration](https://github.com/diego-treitos/linux-smart-enumeration)) lately, with the `-l1` flag to go a level deeper. When I ran that I see that root is running a `/opt/log-backup.sh` script. 

![Flu_3.png](/assets/images/Flu/Flu_3.png){: .responsive-image}

I go check it out, and it looks like I can edit it. So I append the busybox shell.

![Flu_4.png](/assets/images/Flu/Flu_4.png){: .responsive-image}

And boom. Caught the reverse shell for the root user. 

![Flu_5.png](/assets/images/Flu/Flu_5.png){: .responsive-image}

Lessons learned: In my opinion, this was a pretty simple box, even though it's rated as Intermediate by both Offsec and the Proving Grounds community. I did have to look around a little bit for a good exploit given that my initial search results showed one that had been deleted, but after finding it everything went pretty smoothly. Bonus points for the busybox shell and lse.sh, both of which have been great lately. 

