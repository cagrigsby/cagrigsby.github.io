---
layout: writeup
title: Pelican - Proving Grounds
date: 2024-06-20 13:32:20 +0300
description: A Writeup of the Pelican Box from Proving Grounds
image: # /assets/images/Pelican/exhibitor.png
fig-caption: # Add figcaption (optional)
tags: [Linux, TJ Null, LainKusunagi]
---

Alrighty, let's get started with another [Proving Grounds](https://www.offsec.com/labs/) box from TJ Null's OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#). It's actually on LainKusanagi's [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/) as well. This one is called "Pelican," and I had fun with it. It's rated as Intermediate, and I wonder a bit if that's because there's some much information that's ultimately not very necessary to actually solve the machine. I kicked things off with an nmapAutomator scan with this results:

    PORT      STATE SERVICE
    22/tcp    open  ssh
    139/tcp   open  netbios-ssn
    445/tcp   open  microsoft-ds
    631/tcp   open  ipp
    2181/tcp  open  eforward
    2222/tcp  open  EtherNetIP-1
    8080/tcp  open  http-proxy
    8081/tcp  open  blackice-icecap
    37753/tcp open  unknown

I actually ran a Full scan with nmapAutomator which gave the output from some scripts as well, but here's the shortened version. Usuaally for a box like this, the creator doesn't tend to include a bunch of services that aren't actually a part of the exploit, so I figured I was looking at something more complicated than I was. Obivously there's no port 80 open here, so I checked out port 8080 to get started while I ran some background scans. Port 8080 throws a 404 code (Not Found), so I checked out 8081 which actaully auto redirects back to port 8080: 

![Pelican Exhibitor](/assets/images/Pelican/exhibitor.png){: .responsive-image}

It looks like we are dealing with a service called "Exhibitor," so I searched for exploits on that, as well as a few of the other services returned in the more detailed namp scans, including CUPS v 2.2.10, blackice-icecap, and Jetty 1.0. Fortunately I found something quickly for [Exhibitor](https://www.exploit-db.com/exploits/48654). The gist of this script is that you can navigate to the config editor in the platform and place a netcat script to catch a reverse shell on your machine. After that you just hit commit, and you're off.  

![Pelican Exploit](/assets/images/Pelican/runit.png){: .responsive-image}

This actually worked first try for me which feels rare at this point, so that was nice. I caught a shell for user charles and grabbed the local.txt flag. I checked for ssh keys, finding nothing, and then went through the normal process of enumerating the machine. Very early on I checked for SUID bins and ran 'sudo -l' to see if I could run anything without a password. 

![Pelican Gcore](/assets/images/Pelican/gcore.png){: .responsive-image}

Nice. That's usually a great sign for a box like this. That said, I didn't actually know what gcore does, so I copy/pasted some commands from [GTFObins](https://gtfobins.github.io/) which of course did nothing, before actually Googling around to try and learn [something](https://linuxhint.com/gcore-linux-command/). Essentially the gcore command dumps the contents of memory as they relate to running whatever process you select. It also requires superuser privileges, which is why I would be needing sudo to run it. I didn't exactly understand what that would mean, but I did gather that you use it to copy the contents of memory for that process to a file in the working directory, and then check for useful strings within it. So to get started, it's best to look for a process that might have something useful, like credentials. So I checked out some processes and found something juicy: 

![Pelican Processes](/assets/images/Pelican/ps_root.png){: .responsive-image}

That looks like something that could have passwords in it for sure. I ran "sudo gcore 484" and then ran strings on the resulting file, gcore.484. 

![Pelican Password](/assets/images/Pelican/passwd.png){: .responsive-image}

I noticed these strings appearing within the output, looking like a very good sign. So I tried to sign into root:

![Pelican Proof](/assets/images/Pelican/proof.png){: .responsive-image}

And bingo. Clearly I intially forgot to blur the password to blur the password the first time, but what's a lab without a little mistake. Overall, like I said, a ton of information from scans that ultimately wasn't needed at all. Sometimes it's just best to follow the most likely path for a little bit before trying to research every little thing, at least on this machine. Cheers!