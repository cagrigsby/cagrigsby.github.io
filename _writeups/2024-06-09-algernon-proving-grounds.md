---
layout: writeup
title: Algernon - Proving Grounds
date: 2024-06-09 13:32:20 +0300
description: A Writeup of the Algernon Box from Proving Grounds
image: /assets/images/Algernon/Logs.png
fig-caption: # Add figcaption (optional)
tags: [Windows, TJ Null, LainKusunagi]
---


Alright, this is a pretty quick writeup of Algernon from [Proving Grounds](https://www.offsec.com/labs/), part of TJ Null's OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#), as well as LainKusanagi's [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). Apparently that is a useful list as well. 

Right off the bar we get started with an nmap scan which revealed these ports:

PORT      STATE SERVICE
21/tcp    open  ftp
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
5040/tcp  open  unknown
9998/tcp  open  distinct32

A little bit of enumeration revealed that the FTP server allowed anonymous login, so I checked that out while I scanned the web page for directories. 

![Algernon FTP](/assets/images/Algernon/ftp_connected.png){: .responsive-image}

We search around in the FTP server to reveal a bunch of logs, including a few that say administrative. That could be a clue.

![Algernon Logs](/assets/images/Algernon/Logs.png){: .responsive-image}

Unfortunately I didn't really find anything in there, and in fact later realized that new administrative logs continued to be written as I checked out the web page. Port 80 returned the default Microsoft IIS landing page and no interesting sub-directories. So that was a dead end, and I decided to check out port 9998. 

![Algernon SmarterMail](/assets/images/Algernon/SmarterMail.png){: .responsive-image}

So that could be something. We can see the software is called SmarterMail, so we'll check the web for any exploits while fuzzing for more sub-directories (which also didn't show anything particularly interesting.) Side note - I also ran a full port scan at this port which returned an unknown open port on 17001, a clue for later. 

![Algernon Exploits](/assets/images/Algernon/smartermail_exploits.png){: .responsive-image}

After checking through a few of these exploits, we ultimately settle on the RCE exploit for Build 6985. At that point, I didn't know the build, but it felt worth checking out. As I looked through the exploit I noticed that the ports and addresses are hardcoded and need to be changed to my port and IP, as well as the target point and IP. 

![Algernon Ports and IPs](/assets/images/Algernon/change_this.png){: .responsive-image}

And we see 17001 as the target port. I think I might have assumed 9998 because that's where the SmarterMail application was hosted, but it helped to realize that the unknown open port I'd already found was also being used for the exploit. After that we ran the exploit, caught the root shell, and checked for proof.txt. 

![Algernon Root](/assets/images/Algernon/caught_root_shell.png){: .responsive-image}

Bingo! Another quick box here, I should probably start writing up some of the more complex ones, but these go nice and quick. 
