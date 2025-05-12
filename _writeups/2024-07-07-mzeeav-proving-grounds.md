---
layout: writeup
title: MZEEAV - Proving Grounds
date: 2024-07-07 13:32:20 +0300
description: A Writeup of the MZEEAV Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi]
---

Here is a writeup for the MZEEAV lab on [Proving Grounds](https://www.offsec.com/labs/) which I found on another list from LainKusanagi's [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). First things first, I get started with a port scan. This time I'm trying out nmapAutomator, using a `-t Full` option to get a list of open ports and then run some additional scripts on them. That said, I also put the IP in the URL bar just in case there is a web server running on port 80, and in this case, there is.

![MZEEAV_1.png](/assets/images/MZEEAV/MZEEAV_1.png){: .responsive-image}

Folks, it looks like we might be dealing with a file upload vulnerability here, just a guess. I think I need to upload a shell and call it, but I imagine I'm going to need to alter it in some way so that the website actually accepts it. Might need Burp Suite for this one. But first I start with a test.txt file so I can try to figure out where it's being uploaded. Also worth noting I only see ports 22 and 80 open, so we probably will be sticking with port 80. I'll run a directory scan at the same time just in case there's anything hidden. After I upload the test file I'm brought here. Worth noting also that this is a Linux box according to Wappalyzer, so I'm surprised to see .exe files. 

![MZEEAV_2.png](/assets/images/MZEEAV/MZEEAV_2.png){: .responsive-image}

Also a good thing I ran the directory scan. I see that there is a backups folder with a .zip file:

```
200      GET     1213l     7233w   601221c http://192.168.174.33/backups/backup.zip
200      GET       16l       59w      949c http://192.168.174.33/backups/
```

I download it, and it seems to be a backup of the web application: 
![MZAVEE_3.png](/assets/images/MZEEAV/MZEEAV_3.png){: .responsive-image}

Maybe I can look through the upload.php file to get some clues for how to upload a shell or where to look for it if I can't find out from the directory scan. After visiting the http://$IP/upload I can see the corresponding index.html and can also download the files within by visiting those URLs. So that will be the place to go once I can upload a shell. 

![MZAZEE_4.png](/assets/images/MZEEAV/MZEEAV_4.png){: .responsive-image}
That said, when I try to download the file I uploaded (test.txt), I can see it has changed to be called file.tmp. (The file said 'whoami'). So I should expect my shell to be a .tmp file as well. Maybe I can get the directory to execute .tmp files as .php by uploading my own .htaccess file. 

![MZAZEE_5.png](/assets/images/MZEEAV/MZEEAV_5.png){: .responsive-image}

Then it's time to check out the upload.php file. This seems to be the relevant part:

![MZEEAV_6.png](/assets/images/MZEEAV/MZEEAV_6.png){: .responsive-image}

I'm not exactly sure what this means, but it seems to be checking magic bytes to make sure that the file is a PEFILE, which according to Google means Portable Executable, which includes .exe files. Makes sense that the files that were already there were .exe files. I also know that I can execute .fart files as php by uploading a .htaccess file with this content: `AddType application/x-httpd-php .fart`. Maybe I can do the same thing with .exe files. 

From Google, it looks like the magic bytes refers to the initial characters of the file which tell the program executing it what type of file it is. For MZ executables, the magic bytes are `4D 5A` in hex and `MZ` in ISO 8859-1. I'm not totally sure what that means, but I'm going to try to add those to copies of a php executable and test them out. It sounds like I may need to use a .exe file extension as well, after uploading the .htaccess file. 

I tried uploading the php shell titled as .exe, then titled as .exe with `MZ` at the beginning of the file, and then `4D 5A` at the beginning of the file, and only the `MZ` would download, making me think that the other two were basically dropped. I can confirm this from the listing.php page which shows only that one. Finally I upload it again with `MZ` alone on the first line and still titled as .php, and I catch a shell. I guess I didn't need the .htacess file...
![MZEEAV_7.png](/assets/images/MZEEAV/MZEEAV_7.png){: .responsive-image}

I check `sudo -l` and run `find / -type f -perm -u=s 2>dev/null` to check for interesing SUID binaries, and I find `/opt/fileS`. Interesting. I can't read it though, so I decide to run linpeas and lse.sh. I can't really find anything interesting there except that there is a `/opt/fileS` with strange permissions.

![MZAVEE_8.png](/assets/images/MZEEAV/MZEEAV_8.png){: .responsive-image}

I can't read it, and I can't seem to figure out what it does with linpeas, lse, pspy64, and ps. I also took a flyer on compiling DirtyPipe prior to transferring to the target, but I couldn't get it working. After having spent way too long on this, I'm wondering if I actually know how to do whatever technique is required or if this is the lab to learn it. Unfortunately, I'd also just begun a Linux Privesc course through The Cyber Mentor, so maybe it makes sense to finish that and try everything rather than cave and look for a writeup. 

I tried running strace on the file, but it wasn't available. Gcc wasn't available for any exploits. Eventually I tried to run `/opt/fileS --help` and I at least got output. Output I could google. Output which turned out to be documentation for the `find` command, making it looks as though the find command was simply replaced! I used `/opt/fileS` along with the find command for SUID found on gtfobins, and it worked! `./fileS . -exec /bin/sh -p \; -quit` gave me a root shell!


![MZEEAV_9.png](/assets/images/MZEEAV/MZEEAV_9.png){: .responsive-image}

Lessons learned - check `--help` on mysterious binaries I guess. Maybe I should have figured it out from the output of running the command itself, but obviously I didn't. Interesting box. I definitely made it more complicated that necessary for the exploit, and I took way too long on privesc, but I actually feel good about it because I usually would have looked up the answer, and I didn't. Try harder mentality baby!

