---
layout: writeup
title: LaVita - Proving Grounds
date: 2024-09-24 13:32:20 +0300
description: A Writeup of the LaVita Box from Proving Grounds
image: # /assets/images/LaVita/LaVita1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, TJ Null, LainKusanagi, Laravel, pspy64]
---

Today I'm doing a writeup for a [Proving Grounds](https://www.offsec.com/labs/) box from both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)and LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called LaVita, and it is rated Intermediate by Proving Grounds and Hard by the community. As usual, we get started with an nmap (`sudo nmap -A -sC -v -p- --open 192.168.227.38 -oN nmap`) which (gives more detail but) shows these open ports:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
```

So we really mostly have port 80. Port 22 could be of use, but it's often for administrative purposes for the lab creators, and at minimum we will likely need credentials before we can use it. So we check out port 80 in the browser and find:

![LaVita1.png](/assets/images/LaVita/LaVita1.png){: .responsive-image}

It looks like we have some names that could potentially be useful, but I run a directory scan (`gobuster dir -u http://192.168.227.38/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -t 5 -x .php, .txt -o results.txt -k `) while I click around the website. It finds the `/login` and the `/register` directories, and I register a user, which logs me in automatically, taking me to this page:

![LaVita2.png](/assets/images/LaVita/LaVita2.png){: .responsive-image}

Maybe we can upload a reverse shell using this image upload folder. I try to upload a php file, which does not do anything, and then a jpg file which also does nothing. I tried to find the image file (orville.jpg) by visiting `http://192.168.227.38/orville.jpg` which shows the name and version of the service:

![LaVita3.png](/assets/images/LaVita/LaVita3.png){: .responsive-image}

Searching Laravel in exploit-db (and searchsploit) reveals [this RCE](https://www.exploit-db.com/exploits/49424) for version 8.4.2. Close enough maybe. I copy it to my working directory and check it out. It looks like the usage requires the server, path to logs, and a command. I'll have to try with the suggested log path here:

![LaVita4.png](/assets/images/LaVita/LaVita4.png){: .responsive-image}

I run `python3 49424.py http://192.168.227.38 /var/www/html/laravel/storage/logs/laravel.log 'whoami'`, but it just hangs. I also tried [this exploit](https://github.com/zhzyker/CVE-2021-3129), but it doesn't show a result. At this point I figure I should have checked that I know what Laravel is so I can know whether I can just point these exploits at the server or whether it needs to be a specific directory. For reference I had been trying these with debug mode enabled (see the second screenshot of this post), so I tried without, but that seemed to cause additional errors such that I figured it needs to be enabled. 

[Hacktricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/laravel) says "If Laravel is in debugging mode you will be able to access the code and sensitive data. For example http://127.0.0.1:8000/profiles:". So I check for the `/profiles` directory, and I get a 404. While reviewing the source code for `/home` I see that `/upload-image.php` is responsible for uploading images in that directory. I check that to see if I can find anything, and I see this page:

![LaVita5.png](/assets/images/LaVita/LaVita5.png){: .responsive-image}

There's a little bit more information, but one thing that sticks out to me is the top of the page which says `/var/wwww/html/lavita`. I find this interesting because I used `/var/www/html/laravel/storage/logs/laravel.log` to reference the log page, and it makes me wonder if I should simply be using lavita instead of laravel. I try a few options here like swapping the two words or adding lavita before laravel, but nothing works. 

At this point I go back to the [other exploit](https://github.com/zhzyker/CVE-2021-3129) and notice that I'd run it incorrectly, so I try it again while it uses 'id' as the default command. It actually runs 11 RCE attempts and prints them all. About half of them return the output of the `id` command like so: 

![LaVita6.png](/assets/images/LaVita/LaVita6.png){: .responsive-image}

Because I know Laravel/RCE2 works, I change the exploit code to run something else. I change `id` out for `whoami` here:

![LaVita7.png](/assets/images/LaVita/LaVita7.png){: .responsive-image}

And I get a response: 

![LaVita8.png](/assets/images/LaVita/LaVita8.png){: .responsive-image}

Ok so if nothing else, I do have RCE on this machine. From there it was actually kinda tricky to get a shell on the machine. I tried using nc and busybox, but any shell I got died immediately. I also tried downloading reverse shells, but I couldn't get them to execute for whatever reason, either in the browser by downloading it to the webroot or by using the exploit itself. I was able to show that the storage logs are in `/var/www/html/lavita/storage/logs/laravel.log` for the first exploit, but I still can't get it working. 

At this point I go through all the exploits in the second exploit and try them all with the busybox shell, but none of them work. I look around the machine for some other kind of clue like an ssh key or something in `/etc/passwd`, but I still can't find anything. I thought maybe it would make sense to try another exploit at this point, so I found [this one](https://github.com/joshuavanderpoll/CVE-2021-3129). And it works - it's interactive so it will ask for a host and what commands to send, and like the previous exploit, it has multiple exploit which allows it to move to subsequent exploits when one doesn't work. 

That said, I moved back to the previous exploit because I realized I was trying to start a cmd shell for a linux machine rather than a bash shell.

![LaVita9.png](/assets/images/LaVita/LaVita9.png){: .responsive-image}

Which gets me a shell:

![LaVita10.png](/assets/images/LaVita/LaVita10.png){: .responsive-image}

As usual, there's a lot to try. I don't find myself having particularly interesting privileges or SUID binaries, and I don't see much in linpeas, except the password to the DB in lavita which would contain the creds we created. 

![LaVita11.png](/assets/images/LaVita/LaVita11.png){: .responsive-image}

It's something to keep an eye on, but I also run `pspy64` to get more information about running processes, and I find this after letting it run for a while:

![LaVita12.png](/assets/images/LaVita/LaVita12.png){: .responsive-image}

It looks like there is a scheduled task running. We can copy a malicious file to `/var/www/html/lavita/artisan`. In my case I copy a reverse shell (ivan sincek from revshells) there and open up a listener on port 80: 

![LaVita13.png](/assets/images/LaVita/LaVita13.png){: .responsive-image}

With the skunk user, I run `sudo -l` and find that I can run `/usr/bin/composer` with sudo without a password but only with `--working-dir\=/var/www/html/lavita *`. 

![LaVita14.png](/assets/images/LaVita/LaVita14.png){: .responsive-image}

We can check out this binary on [gtfobins](https://gtfobins.github.io/gtfobins/composer/) and see:

![LaVita15.png](/assets/images/LaVita/LaVita15.png){: .responsive-image}

Our skunk user does not have write access to the `/var/www/html/lavita` directory, but does have access to run composer with sudo. So we need to write with our www-data user and execute with skunk.

From www-data (**from the /var/www/html/lavita directory**): `echo '{"scripts":{"x":"/bin/sh -i 0<&3 1>&3 2>&3"}}' > composer.json`

From skunk: `sudo /usr/bin/composer --working-dir=/var/www/html/lavita run-script x`

![LaVita16.png](/assets/images/LaVita/LaVita16.png){: .responsive-image}
