---
layout: writeup
title: Nukem - Proving Grounds
date: 2024-08-10 13:32:20 +0300
description: A Writeup of the Nukem Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi, MySQL]
---

Here's a writeup for Nukem, an Intermediate Proving Grounds box I saw on the [LainKusanagi list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). I take note that the community has rated this Hard. Let's kick things off with a port scan using nmap: `sudo nmap -p- -T4 -v 192.168.195.105`. I also notice there is a website open on port 80 as I usually throw that in the URL bar to check just in case, so I go ahead and scan that with feroxbuster: `feroxbuster -u http://192.168.195.105/ --thorough -r`. 

Here we have the nmap results, as well as the homepage:

```
PORT      STATE  SERVICE
22/tcp    open   ssh
80/tcp    open   http
3306/tcp  open   mysql
5000/tcp  closed upnp
13000/tcp open   unknown
36445/tcp open   unknown
```

![Nukem1.png](/assets/images/Nukem/Nukem1.png){: .responsive-image}

I notice a few unusual open ports, so I run another nmap scan to get more details: `sudo nmap -sC -A -p22,80,3306,5000,13000,36445 -T4 192.168.195.105`, and the site says it's a Wordpress site, so I'll kick off `wpscan` as well. As I check around the website, I find that there is a student registration page which seems to register my user for the Wordpress site, though it doesn't appear there is much to do with my permissions. I also see that port 13000 appears to be a web server, but I just see a login page, so I make a note to run a directory scan and move on for now. 

I take a look at the long output from the wpscan, and I noticed that there seem to be some vulnerabilities with a plugin called `Simple File List`. 

![Nukem2.png](/assets/images/Nukem/Nukem2.png){: .responsive-image}

I get started on that to see if I can find anything quickly. I notice there are a couple exploits on exploit-db, and I try an RCE one which doesn't work for me before trying the [Arbitrary File Upload](https://www.exploit-db.com/exploits/48979)exploit next. After editing the code, to reflect my own $IP and port, I run it, and I get a shell. 

![Nukem3.png](/assets/images/Nukem/Nukem3.png){: .responsive-image}

![Nukem4.png](/assets/images/Nukem/Nukem4.png){: .responsive-image}

At that point I look briefly around the machine and grab the `local.txt` flag in `/home/commander`. So we know there is at least one non-root user. Commander also has a directory called `python_rest_flask` in their home directory, maybe that will come into play later. At that point I move to temp to run linpeas/lse and a few other basic low-hanging fruit commands. At that point I noticed some interesting SUID binaries, dosbox in particular. 

![Nukem5.png](/assets/images/Nukem/Nukem5.png){: .responsive-image}

There is also a service running on `127.0.0.1:5901`, and I can see this line in linpeas:
`root         554  0.0  1.5  44804 31680 ?        Ss   21:17   0:00 /usr/bin/python /home/commander/python_rest_flask/server.py`. It shows that root is running this python server, so there could be a privesc path there. 

I start with `/usr/bin/dosbox`, but I can't get anything working, and further research suggests we may need access to the DOSBox graphical window, so I drop that for the moment. I also try to access the MySQL server, but I cannot from either host. I do eventually see this in the linpeas output:

![Nukem6.png](/assets/images/Nukem/Nukem6.png){: .responsive-image}

From there I can login to the mysql server and access the users table, which ultimately leads nowhere as I already have access to these accounts. 

![Nukem7.png](/assets/images/Nukem/Nukem7.png){: .responsive-image}

But I can also try the same password for commander just by switching to that user:

![Nukem8.png](/assets/images/Nukem/Nukem8.png){: .responsive-image}

At this point I got stuck for a while looking around the machine and trying to use `dosbox` and failing, initially because I thought I needed GUI access thanks to a comment I read, and then because I tried to use it to read authorized files but couldn't get it working.  And I looked up the next step (SHAME). I had

Apparently the correct path the entire time was to use dosbox to *write* to a file. The first require line from gtfobins for dosbox is `LFILE='\path\to\file_to_write'`. I misunderstood the whole time and thought I was writing to a temporary file. In fact the next line is `./dosbox -c 'mount c /' -c "echo DATA >c:$LFILE" -c exit`. This is made slightly more confusing by the `c:$LFILE` because it looks like Windows when that is not the case. I begin by using `openssl passwd party321` to create a password and use dosbox to append it to the /`/etc/passwd` file. It works to create a new user, but the new password doesn't show up, and I can't `su` into it. 

![Nukem9.png](/assets/images/Nukem/Nukem9.png){: .responsive-image}

But then we can still add extra permissions to the `/etc/sudoers` file for our commander user. `LFILE='/etc/sudoers` and `/usr/bin/dosbox -c 'mount c /' -c "echo commander ALL=(ALL) NOPASSWD: ALL >> c:$LFILE" -c exit` lets us use sudo on anything, so we use `sudo su` and get the proof.txt. 

![Nukem10.png](/assets/images/Nukem/Nukem10.png){: .responsive-image}

Lessons learned - when you ge the opportunity to write with permissions, you can try both `/etc/passwd/` and `/etc/sudoers`, among other things like authorized_keys perhaps. I guess it also helps to remember that some things are just there to steal you attention. The website on port 13000 didn't lead anywhere, and we didn't actually need to log into the MySQL server as it was just there so we could read the config file for commander's password. Also I'm not sure if the `/home/commander/python_rest_flask` was used for this box at all, but I tried adding a reverse python shell to it, knowing that root had executed it before, but that did nothing. 

All in all, a pretty frustrating box. I found the intended path very early on, but I just didn't understand how to exploit it. I guess now I'll know for next time. 

Bonus: After doing reading a little bit more after, I saw [this writeup](https://medium.com/@vivek-kumar/offensive-security-proving-grounds-walk-through-nukem-3fe58fcf64ec)and  found that escaping $'s would have allowed me to properly add my new password to the `/etc/passwd` file. At least I learned something even when I failed. 
