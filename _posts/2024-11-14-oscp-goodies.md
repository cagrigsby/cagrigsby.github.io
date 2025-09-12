---
layout: post
title: OSCP+ Guide, Tips, and Resources
#date: 2024-11-14 13:32:20 +0300
description: Everything I Wish I'd Known Outside the Course Material
image: /assets/images/bloodhound_header.jpg
fig-caption: # Add figcaption (optional)
tags: [personal, professional]
---

## Table of Contents

- [Table of Contents](#table-of-contents)
- [Purpose](#purpose)
- [Who Am I](#who-am-i)
- [How Did I Prepare](#how-did-i-prepare)
- [My Setup](#my-setup)
- [General Advice](#general-advice)
	- [Do The Suggested Labs](#do-the-suggested-labs)
	- [Have a Process For Taking Notes](#have-a-process-for-taking-notes)
	- [Use AI](#use-ai)
	- [Automate As Much As You Can By Writing Your Own Scripts](#automate-as-much-as-you-can-by-writing-your-own-scripts)
		- [Checklists](#checklists)
		- [Shameless Script Kiddie Behavior](#shameless-script-kiddie-behavior)
- [More Specific Tips](#more-specific-tips)
	- [adPEAS](#adpeas)
	- [Bloodhound Abuse](#bloodhound-abuse)
	- [Maintain Your Wordlists](#maintain-your-wordlists)
	- [No Nano, ~~No~~ Fewer Problems](#no-nano-no-fewer-problems)
	- [Other .zshrc Options](#other-zshrc-options)
	- [Prevent Hanging](#prevent-hanging)
	- [Speedier, Thorough Enumeration](#speedier-thorough-enumeration)
	- [SublimeText or Equivalent](#sublimetext-or-equivalent)
	- [XFreeRDP](#xfreerdp)
	- [This, Not That](#this-not-that)
	- [Soup Up Your VM](#soup-up-your-vm)
- [FAQ](#faq)
	- [How many lab machines is enough?](#how-many-lab-machines-is-enough)
	- [Is the course material enough to pass the exam?](#is-the-course-material-enough-to-pass-the-exam)
	- [Oh cool, which labs?](#oh-cool-which-labs)
	- [What other material would you recommend?](#what-other-material-would-you-recommend)
	- [Is the OSCP Worth It?](#is-the-oscp-worth-it)
- [Valuable Resources](#valuable-resources)
	- [Kali Package Manager](#kali-package-manager)
	- [GitHub](#github)
	- [Websites/Gitbooks](#websitesgitbooks)
	- [YouTube](#youtube)
- [Closing Thoughts](#closing-thoughts)

## Purpose
While preparing to take the OSCP, I frequently browsed the [OSCP subreddit](http://old.reddit.com/r/oscp) checking guides and advice posts, and I saw [this one](https://eins.li/posts/oscp-secret-sauce/) in particular which helped a ton despite the content being mostly just a few helpful commands. I don't want to just copy their stuff, but you should check it out. The busybox shell and Mimikatz one-liner it references were vital for me. I figured I could share a few other suggestions that I found useful outside the PEN-200 material. I want to give (or rather underscore) some general tips on how to prepare, make note of a few specific suggestions I wish I'd known, and then I'll dig into some FAQs. 

Please note that this post is written for people who are attempting or want to attempt the OSCP, with a special focus on those with a little less experience. **I am trying to share some of the things I learned outside the PEN-200 course that made the experience easier on me.** Some of them *should* be too simple for you. 

For a more complete technical/methodological guide, I like [this one](https://medium.com/@redefiningreality/your-bs-less-guide-to-acing-oscp-4eccaf497410). For my personal notes, they are available [here](link) in repo form, and [here](https://pentestpop.github.io/oscp_notes/) on this very site.  

## Who Am I
I may have a more detailed blog post about this, but the gist is that I became interested in cybersecurity during the Covid-19 pandemic and from 0 in late 2020, I completed a bunch of online courses in IT generally, cybersecurity specifically, and also programming, in particular for Python, SQL, and bash. I spent tons and tons of time on [TryHackMe](https://tryhackme.com). I picked up a few certifications including Sec+, CySA+, and the ethical Junior Penetration Tester, or [eJPT](https://security.ine.com/certifications/ejpt-certification/) from INE security. I mention the last one because while it's not as prestigious, it can be seen as jumping off point for OSCP. I would say it's a good place to start, especially to figure out whether you have a passion for this, but it's not essential. If you can do that, you're ready to move on, but of course there are a ton of other signals that it's time. For work, I spent a little over three years in security compliance, both outside as a consultant and inside in GRC. 

All this to say that I'm approaching this guide as someone who did not spend a ton of time IT or engineering, without much formal training. If I did it, you can too, but you're gonna need to love it. 

## How Did I Prepare
It's difficult to know how much this is going to help given that we all start from a different place, but for some perspective, here's what I did:
- Full PEN-200 course with all exercises completed
- These Challenge Labs:
	- Secura
	- Medtech
	- Relia
	- OSCP A
	- OSCP B
	- OSCP C
- [TCM Academy](https://academy.tcm-sec.com/):
	- Full course: Linux Privilege Escalation for Beginners
	- Full course: Windows Privilege Escalation for Beginners
	- From Practical Ethical Hacking: The Complete Course
		- Active Directory Overview
		- Attacking Active Directory: Initial Attack Vectors
		- Attacking Active Directory: Post-Compromise Enumeration
		- Active Directory Case Studies
- From [Hack The Box Academy](https://academy.hackthebox.com)
	- [Active Directory Enumeration & Attacks](https://academy.hackthebox.com/module/details/143):
- From [TryHackMe](https://tryhackme.com/):
	- Over 200 rooms, so too many to list here. My public profile is [here](https://tryhackme.com/r/p/grica421), and it lists them. A lot of this was pretty basic stuff I did when I was first starting, but everything helps. That's how to get to top 1% for years lol. 
- From [Hack The Box Labs](https://app.hackthebox.com)and [Proving Grounds](https://www.offsec.com/labs/individual/): 
	- Again it's a lot, 90+ machines combined. I have 50+ writeups on this website, and I highly recommend doing writeups while you're going through your own labs, to practice taking better notes if nothing else. 
	- The majority of these were from [TJ Null’s PEN-200 list of labs](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#) and [LainKusanagi’s list of OSCP like machines](https://docs.google.com/spreadsheets/d/18weuz_Eeynr6sXFQ87Cd5F0slOj9Z6rt/edit?gid=487240997#gid=487240997). I did every one that overlapped between them at least once. 
- So much YouTube, so much Google, so much ChatGPT. 
- Then I went back and did the Challenge Labs again, in part to test myself, but also in part to see if I could identify anything noteworthy about the practice exams that might help me in the real thing, more on that later. 

## My Setup
- M1 Macbook Pro
- VMWare Fusion (free) with Kali on the external monitor.
- Two Obisdian notes on built-in monitor - with my notes repo to continuously update on one side and my current working note (like writeup) on the other. 

I really only mention this because I considered alternatives like:
- Getting a used AMD64 machine because some of the material references tools that don't work well on ARM (reverse engineering and buffer overflows)
- Paying for a hypervisor like Parallels
- Taking notes on my Kali VM (smh my damn head)

**Do not do any of that.** ARM is fine for the exam and the free version of VMWare is fine. Do not take notes on a VM, unless you already pay for a cloud feature or something. Do not become enamored with the simplicity of CherryTree. Do not fiddle with Flameshot hot keys. Do not put yourself in any kind of position that you need to transfer files between your VM and host. Either you know this already or you haven't spent enough time screwing around yet. **In fact, if you are a prospective employer, I was joking about taking notes locally on a VM. I never did that, obviously. I am very smart, and I would never make such an embarrassing and foolish mitsake.**

![](/assets/images/OSCP_Tricks/joking.png){: .responsive-image}

## General Advice
For the most part, you probably know this stuff already, but you really should take it to heart. These sections also contain a few different resources that have helped me along the way. 

### Do The Suggested Labs
Duh. TJ Null list [here](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview), LainKusanagi list [here](https://docs.google.com/spreadsheets/d/18weuz_Eeynr6sXFQ87Cd5F0slOj9Z6rt/edit?gid=487240997#gid=487240997). A reddit user named [/u/obeyeater](https://www.reddit.com/r/oscp/comments/1gee7m4/from_zero_to_90_points/) compiled them both into a study tracker [here](https://docs.google.com/spreadsheets/d/1nzEN0G6GzneWCfs6qte6Qqv-i8cV_j6po-tFlZAOx1k/edit?gid=488959887#gid=488959887). I wish I'd found it sooner to be honest. Make a copy, update it, check your progress. 

How many do you need to do? This is impossible to know, and as much as you may want a discrete number, anyone who gives you one is pandering. Your background is different from mine; your test will be different from mine. I would say do writeups as you go and take note of when you need to look for hints. If it's on something not covered by the Exam material, fine. If you're needing to search for writeups because you forgot to do something simple or just never got the syntax right, you're probably not ready. 

You need to be able to knock out Proving Grounds Easy machines with no help for sure. If you need help on a Medium box, it better be because you had something to learn outside the course material. 

### Have a Process For Taking Notes
Notice that I did not say something generic like "take good notes." You need a process. Maybe this is a no brainer, but I never said this was a post for people with brains, and it's so, so important. I think sometimes I failed at this because I figured I understood something well enough to not need notes, but a couple mistakes with that is too many. Look at all the different OSCP gitbooks on the internet, those are from people who prioritized taking notes. Look at them, think seriously about the best way for you to emulate them, and then do it. Copy mine if you like, again they're [here](https://github.com/pentestpop/OSCP_Vault). You can download the repo and open it as a vault in Obsidian. Here's a taste: 

![](/assets/images/OSCP_Tricks/tasty.png){: .responsive-image}

I just keep this open in a window at all times and add to it when I learn something new or want to remember some syntax. 

### Use AI
I know you can't do it on the exam, but you can learn a lot from asking ChatGPT to break down a command or *why* something is failing. Google is great, Stack Overflow is great, reddit is great, the Offsec discord is great, but AI tools can be awesome for specific questions or questions people on other forums deem unworthy to answer for one reason or another. Yes, ChatGPT can hallucinate, but you'll probably learn from that blowing up in your face too. The tools are incredibly useful, and it's a waste not to use especially knowing that the other team is. 

Btw, as of November 2024, [Claude](https://claude.ai/) is better for code, but the free tier is more limited by message, so sadly you might want to skip the "Perfect, that works, thank you" messages. I tend to be polite to chat bots, but hey I'm from the South. 

### Automate As Much As You Can By Writing Your Own Scripts
Automate by forcing your robot friends to write your own scripts!  Yeah, you'll learn more trying to write every line yourself, but you'll learn nothing if you give up because it's too confusing. I have a few custom scripts on my [GitHub](https://github.com/pentestpop/PopScripts), so you can get more details from the README there, but basically if I know some task is going to require multiple commands, or long commands I might mistype, I (or [someone](https://miro.medium.com/v2/resize:fit:4800/format:webp/1*GmDjVtGv2I7A8-X5vC4tAQ.jpeg)) create a script which runs the commands and echos anything I might need to paste into another window. Examples:
- `liggy` starts Ligolo, asks for what subnet to pivot into, and then prints commands to run into the shell on the target.
- `ivan` prompts the user for a IP and Port before generating a new copy of Ivan Sincek's [PHP-Reverse-Shell](https://github.com/ivan-sincek/php-reverse-shell/blob/master/src/reverse/php_reverse_shell.php) in the working directory.
- `served` prints certutil, iwr, and wget commands to be run on a remote target to download files from your machine using the `tun0` IP. So `served -f file.txt` returns `iwr -uri http://tun0/file.txt -o file.txt` and so on. 
Feel free to copy these or fork them or whatever. But probably you can find some other examples. Losing focus is death by a thousand cuts, make everything as easy on yourself as possible.

#### Checklists
For the OSCP specifically, there's kind of a limited number of pathways you can take to exploit a machine. And it's good advice to **Enumerate Deeply, Exploit Simply**. Check the practice tests they give you, the exploits simply are not that complicated. If you're struggling, you probably missed something relatively simple. One way to make sure you catch all the low hanging fruit is to run scripts which give you all the output. Once again, I have some of this on my [GitHub](https://github.com/pentestpop/verybasicenum). I noticed that I forgot to run a UDP scan a few times, so I made a script for nmap called `vbnmap` (very basic nmap). It doesn't have all the bells and whistles of [AutoRecon](https://github.com/Tib3rius/AutoRecon) for example, but it's fast, it automatically creates an output file, and you won't miss anything. It runs: 
- A simple TCP scan to the terminal so you can get started on the commonly used ports
- A simple full port (`-p-`) TCP scan
- A more detailed TCP scans with only the ports from the full TCP scan
- A UDP scan
- And it creates an output file with all this information. Simple. 

I also have [verybasicenum](https://github.com/pentestpop/verybasicenum) scripts for `.bat`, `.ps1`, and `.sh` that print out the lowest hanging fruit like the users, history files, the common directories where there might be an unusual binary like `/opt` for Linux or `C:\Program Files` for Windows. If you see something that doesn't belong, there's a pretty good chance it's involved. For example, the `.ps1` script runs:
```
whoami
whoami /priv
whoami /groups
net user $env:USERNAME
net user /domain
systeminfo
ipconfig
Get-ChildItem C:\
Get-ItemProperty 'HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*' | Select-Object DisplayName
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*'
Get-Process
Get-Content "$HOME\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
```

The point is to take the commands you know you need to run every time, put them in one script, and run it every time so you know you won't forget to do something simple like check the history file. 

#### Shameless Script Kiddie Behavior
You can of course take this one step further and automate a bunch of commands with a combination of different tools. I'll just drop [this here](https://github.com/pentestpop/verystupidenum), and then we'll never speak of it again. 

## More Specific Tips
That was more genreral advice you almost certainly have heard but may have yet to internalize. Here are more specific items I have picked up along the journey. 

### adPEAS
Maybe you're familiar already, but I feel like I don't see [adPEAS](https://github.com/61106960/adPEAS) talked about nearly as much as Linpeas and Winpeas, maybe because it's from a different creator? Per the README.md:

```
adPEAS is a Powershell tool to automate Active Directory enumeration. In fact, adPEAS is like a wrapper for different other cool projects like

- PowerView
- PoshADCS
- BloodHound Community Edition
- and some own written lines of code
```

It addition to listing out useful information (like ASREPRoastable and Kerberoastable accounts and other credential exposure), it also outputs `.json` files to be used for Bloodhound. 

*I do want to take a second to note that at times the formatting can be weird. It's possible that you may need to re-run Sharphound/bloodhound-python or Rubeus. Most of the time it's fine, but if you're patient, you may be better off treating this as simply as a tool that checks for low-hanging fruit.*

### Bloodhound Abuse
This is only briefly touched on in the course material, but I think only in the caption of an image. **Bloodhound actually includes explicit direction on how to abuse certain permissions and relationships.** See this output from HackTheBox's Support lab (no spoilers in this one):

![](/assets/images/OSCP_Tricks/bloodhound1.png){: .responsive-image}

The Administrator has DCSync permissions over Support.HTB. If we right-click on that edge (right where the cursors is above), we get this view:

![](/assets/images/OSCP_Tricks/bloodhound2.png){: .responsive-image}

Click Help, and we get this view:

![](/assets/images/OSCP_Tricks/bloodhound3.png){: .responsive-image}

It explains the relationship, and if we click Windows Abuse:

![](/assets/images/OSCP_Tricks/bloodhound4.png){: .responsive-image}

We get the exact command we would need to perform a DC Sync attack. In this example it doesn't really make a difference because we would already have Administrator access, but it can seriously help. It just happened to be quick to find this example in my files. What you can do, is check the `Node Info` tab on the the users you do have access, and the click on the different `Outbound Object Control` buttons to see what rights that user has over other nodes. In this case the Administrator user has rights over 94 nodes because it is in the "Domain Admins" group, hence `Group Delegated Object Control`.

![](/assets/images/OSCP_Tricks/bloodhound5.png){: .responsive-image}

Anyway, I probably wouldn't have known this without doing a bunch of labs and reading writeups, and it can be super helpful. You should be able to find this info anyway, but it's nice to have in one place. 

### Maintain Your Wordlists
Wordlists are a thoroughly discussed topic for the OSCP, but I wanted to call out a few things. For the most part `rockyou.txt` is fine for passwords, especially with hashcat rules. Definitely in some cases there are opportunities to create custom worldlists with `cewl`, but for most labs I think `rockyou.txt` is fine. For rules I like `hashcat -m $mode $hashFile /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force`.

But you might want to be more careful with directory brute forcing. In some labs there are cases where you should try directory wordlists specific to the technology in use, like certain CMS's for example. I also needed help on a few labs because I couldn't find the hidden directory. If you do enough of the TJ Null and LainKusanagi lists, you may realize that `.git` is missing on a lot of well-used directory wordlists. For example, it is not on:
-  `/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt` - the default wordlist for `feroxbuster` is
- `/usr/share/wordlists/dirb/big.txt`
- `/usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-small.txt`
- `/usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt`

If you find yourself failing a lab because you didn't have the correct extension for a directory, try using a specific wordlist, and even consider adding that word to you most used wordlists. If you haven't run into this, you may just not have done enough labs yet. 

### No Nano, ~~No~~ Fewer Problems
Use `cat` to create new files (including to copy/paste):
```
cat <<'EOT'> $file.name
> text
> text
> EOT
```
- (typing `EOT` ends the file)

### Other .zshrc Options
While I'm at it, I also add these:

`alias grep='grep --color=auto'`
- Colors the search term when you run `grep`

`alias gitupdate='find /opt -maxdepth 1 -type d -exec bash -c "cd \"{}\"; git pull;" \;'`
- When I clone directories I know I'll be using a lot, I clone them into my `/opt` folder. This helps me to update them when I run `sudo gitupdate`. 

`function mkcd() { mkdir -p "$1" && cd "$1"; }`
- Shout out to whoever put this wherever I saw it. It just means that when you run `mkcd directoryname`, you both create the directory and `cd` into it. Does it save a ton of time? Not really. But it's nice. 
`catch() { rlwrap nc -lvnp "${1:-443}"; }`
- Starts a listener on whatever port you tell it so `rcatch 8080` runs `rlwrap nc -lvnp 8080` with 443 by default. To be honest, I rarely used this because of autocomplete working it's magic every time I started a command with `rl`, but it is kinda cool. 

I've seen people using other custom aliases to do things like start a python server, for example you could include: `function serve() {python3 -m http.server "$1" ; }` so that when you run `serve 80`, a server starts on port 80. Mix it up, make it work for you. 

### Prevent Hanging
You can waste a lot of time restarting shells over and over again because you tried to run the a command on a less-than-stable shell. One way to solve this is to use a nested shell, meaning catch one reverse shell, and then use it to start another. Another way is to use proper commands which start a process and let you get back to what you were doing:

To prevent hanging while running Windows commands:
- `cmd.exe /c $command
- `cmd.exe /c start $command`
- i.e. `cmd.exe /c .\winpeas.exe > winpeas.txt`

To prevent hanging while running Linux commands:
- `$command &`
- i.e. `./linpeas.sh > linpeas.txt &`

### Speedier, Thorough Enumeration
This seems a little too simple for this post, but at some point I didn't know it, and then I learned it, and then life was better. And I definitely learned some of it later than I should. Here are a few commands and their output in a sample directory `example`:

`ls`:

![](/assets/images/OSCP_Tricks/enumeration1.png){: .responsive-image}

`ls -A`:

![](/assets/images/OSCP_Tricks/enumeration2.png){: .responsive-image}

`ls -lA`:

![](/assets/images/OSCP_Tricks/enumeration3.png){: .responsive-image}

`tree`:

![](/assets/images/OSCP_Tricks/enumeration4.png){: .responsive-image}

`tree -a`:

![](/assets/images/OSCP_Tricks/enumeration5.png){: .responsive-image}

`find .`:

![](/assets/images/OSCP_Tricks/enumeration6.png){: .responsive-image}

I highly recommend you consider these commands and how to use them. The `tree` command is even available on Windows (try `tree /a /f`). For the longest time I was `cd`ing into a directory, running `ls` and then `cd`ing into another directory. It was ridiculous. I'm sure most of you aren't doing that but for the few who don't know yet, here ya go buddy. 

I like to add this to my `~/.zshrc` file:
`alias ls='ls -A -F --group-directories-first --sort=extension --color=always'`.  Obviously it won't help on remote hosts, but it's nice on my own machine. For me this just returns the directories first in blue, includes the `.secret` files, and sorts by extension. On your own machine, do what thou wilt. 

 For SMB you can copy a full smb share by running `mget *` inside it, if it's easier to check out locally. Just run `recurse on` and `prompt off` first, so it grabs everything and doesn't confirm each time to download it. 
- In a similiar vein, you can show every file in an SMB share by running: `smbclient //<IP>/<share_name> -c 'recurse;ls'`

### SublimeText or Equivalent
Obviously use whatever equivalent you prefer, but if you have the monitor space, it's pretty great to keep a Sublime window open in your working directory. I've grown to love working from the command line, but it's nice to have everything in one window, and it updates live as you add new files. To show you what I mean, here are my working directory for the HackTheBox machine Return:

![](/assets/images/OSCP_Tricks/Return.png){: .responsive-image}

Here I can see a git repo I downloaded, everything I download from an SMB share, a file with creds I'd found, nmap results, and winpeas output all quickly in one place. If you like [autorecon](https://github.com/Tib3rius/AutoRecon), you know it can take a while to run, but you can view the results as it goes. Simple, but I'd been studying for months before I started doing this. I usually have this, a terminal window, and a web window open, and that's about all I need unless I need to check into BurpSuite Wireshark, or BloodHound for whatever reason. 

### XFreeRDP
`xfreerdp` has a dynamic resolution option. They never mention it in the course material, and it makes the experience clunky as hell. `/dynamic-resolution` allows you to re-size the window which by default you cannot do. I like to run `xfreerdp /u:$user /p:$password /v:$target /drive:/$directoryToShare,$nameToShare /dynamic-resolution`.

### This, Not That
Rapid fire:
- Maybe you know already, but [the Ivan Sincek PHP reverse shell](https://github.com/ivan-sincek/php-reverse-shell) is the best PHP reverse shell in my opinion. I never once had it fail when another worked. 
- I prefer [LSE](https://github.com/diego-treitos/linux-smart-enumeration) to Linpeas. Here is part of the output from the Monitored box on HackTheBox. See how it only gives the output it thinks is notable? If you don't find anything here, run linpeas sure. But I start with `./lse.sh -l1`. 

![](/assets/images/OSCP_Tricks/lse.png){: .responsive-image}

- `ldapdomaindump` over `ldapsearch` and `bloodhound-python`. It does both, it outputs to a chosen directory `.grep` files, `.json` files, and `.html` files for easy viewing in browser.  
	- `ldapdomaindump -u $domain.com\\$user -p '$Password' $domain.com -o $outputDirectory`
- `rlwrap -cAr nc -lvnp $port` is more stable than `nc -lvnp $port`
- Proving Grounds over HackTheBox for OSCP. It's the same folks who make the exam. Also if you don't pay extra for HTB VIP, multiple users can use the same target machine at the same time, and there were instances where it was confusing what was meant to be part of the machine and what was from another user. This ruined some boxes for me. 
- `ligolo` over everything else. For the OSCP, you don't need any additional functionality that one of the other options provides. It was good on every box from the course and every one of the practice labs I did. And it has port forwarding, here is a [guide](https://medium.com/@Thigh_GoD/ligolo-ng-finally-adds-local-port-forwarding-5bf9b19609f9). **Learn this and use it.**
- Use `CTRL + Shift + L` instead of `clear` to move the command line to top of the screen so you can see the results better. This is better than clearing the whole screen in case you need to scroll up. 

### Soup Up Your VM
Intellectually I know it to be a silly thing to spend time on. But the thing is, studying for this exam takes a ton of time. It will take 100's of hours, even 1000's of hours. A lot of that time is going to be discouraging. So remember to have some enjoy the ride. Feel like a HACKERMAN (or HACKERWOMAN or HACKERPERSON or HACKERBABY) and don't take yourself too seriously. **If you aren't having fun, You're Not Gonna Make It.** So you might as well have fun. 

![](/assets/images/OSCP_Tricks/hackerbaby.webp){: .responsive-image}

Believe it or not, that ^ is an AI-generated image. HACKERBABY is a menace at Roblox hacking. I also use [Terminator](https://gnome-terminator.org/) terminal emulator which has a ton of themes to choose from. There are some very simple instructions [here](https://github.com/EliverLara/terminator-themes). I prefer using bright colors so I can recognize my own commands quickly when scrolling. You can also set up a default grid for every time you open it. Mine looks like this: 

![](/assets/images/OSCP_Tricks/grid.png){: .responsive-image}

I use [coolers.co](https://coolors.co/201e1f-ff4000-faaa8d-feefdd-50b2c0)to generate color schemes and use them when I can. I have a custom background I made with a free Photoshop clone called [Photopea](https://www.photopea.com/). 

Try different Desktop environments. Personally I use XFCE because I couldn't quite get GNOME to do everything I wanted. Customize your keyboard shortcuts. I like a tiling manager on my host machine called [Rectangle](https://rectangleapp.com/), but Kali has a lot of that functionality built-in.  The commands and steps are going to depend on your Desktop Environment. Fortunately, we have Google and our [Robot Friends](#use-ai) to help.

## FAQ
### How many lab machines is enough?
I don't know. No one knows. As many as you can do, I guess. As I said [above](#do-the-suggested-labs), one useful metric to use is to do Proving Grounds Practice boxes (since they are created and maintained by Offsec), and do the Easy boxes without help and the Medium boxes without needing help on course material. 

I want to reiterate, no one can really answer this because no one knows your background and no one knows what kind of exam you get. Maybe you could even speed through the course once and still pass the exam if you so happen to retain only the exact most useful information for your specific exam like some kind of Slumdog Millionaire situation. Maybe don't plan on that. 

### Is the course material enough to pass the exam?
This is a better question to me, and I think the answer is technically yes. The benefit of doing a bunch of outside labs and courses is to get reps in, build your notes, and perhaps most importantly to gain exposure to different kinds of problems. I would say there were specific labs from outside the course material that included the same technologies that I encountered on the exam itself, even if not necessarily the same solution. If I didn't have that exposure, maybe the outcome would have been different. 

### Oh cool, which labs?
Nice try officer.

![](/assets/images/OSCP_Tricks/nope.webp){: .responsive-image}

### What other material would you recommend?
I guess just see the How Did I Prepare Section. I can tell you with certainty that that was enough to pass my exam. I'll clarify a bit here - there's a ton over overlapping information among those resources and the exam material itself. Part of how I learned, like actually learned, was to see it more than once in different contexts. Maybe you don't need that, maybe you do. So I can't really say whether TCM's Windows Privsec course is a must-do addition or not. I can just say that it helped me to revisit a lot of the material I'd already learned because I had built up my greater context by the second time I saw it. **I suggest you do a course, then spend a few weeks doing labs, then do another course, then do some labs, then revisit a course, then do some labs, and so on**. It's just a good way to hammer everything home. 

### Is the OSCP Worth It?
I don't know. It's definitely the most challenging and most fun certification I've done. Probably the most interesting exam I've had in my life actually. 

## Valuable Resources
### Kali Package Manager
I have a script that I run on fresh Kali spinups, and these are the tools I tend to install on top of vanilla Kali. All of these can be installed with `sudo apt install $name`:
- [autorecon](https://github.com/Tib3rius/AutoRecon) - network reconnaissance tool which performs automated enumeration of services, explicitly written by Tib3rius for CTFs and other penetration testing environments
- [bloodhound](https://www.kali.org/tools/bloodhound/) - for visualizing Active Directory information  
- [bloodhound.py](https://www.kali.org/tools/bloodhound.py/) - for collecting `.json` files for bloodhound to ingest remotely
- [burpsuite](https://www.kali.org/tools/burpsuite/) - for performing security testing of web applications , through editing HTTP requests for example
- [enum4linux](https://www.kali.org/tools/enum4linux/) - a tool for enumerating information from Windows and Samba systems remotely
- [gccgo-go](https://go.dev/doc/install/gccgo) - a compiler for Go 
- [gobuster](https://www.kali.org/tools/gobuster/) - brute-force directories and files in websites, Virtual Host names, and subdomains 
- [golang-go](https://go.dev/) - the Go programming language 
- [hekatomb](https://www.kali.org/tools/hekatomb/)- a Python script that connects to an LDAP directory to retrieve all computers and users’ information in order to decrypt DPAPI blobs 
- [kerberoast](https://www.kali.org/tools/kerberoast/) - for kerberoasting, though you can probably just use a combination of other tools, impacket in particular 
- [krb5-user](https://packages.debian.org/bullseye/krb5-user) - this package contains the basic programs to authenticate to MIT Kerberos
- [libreoffice](https://www.libreoffice.org/) - FOSS office suite 
- [neo4j](https://neo4j.com/)- graph and database management, you need it to run BloodHound 
- [netexec](https://www.kali.org/tools/netexec/) - includes nxc, the updated version of crackmapexec
- [name-that-hash](https://www.kali.org/tools/name-that-hash/)- or nth, a hash identifier through either file or text 
- [onesixtyone](https://www.kali.org/tools/onesixtyone/) - a simple SNMP scanner, particularly useful for identifying community strings
- [peass](https://www.kali.org/tools/peass-ng/) - well-known privilege escalation scripts for Windows and Linux (and MacOS)
- [pspy](https://www.kali.org/tools/pspy/)- a command line tool designed to snoop on processes without need for root permissions. You'll want to run a binary on target machines.  
- [python3-ldapdomaindump](https://www.kali.org/tools/python-ldapdomaindump/) - Active Directory information dumper via LDAP
- [python3-pip](https://www.kali.org/tools/python-pip/#python3-pip) - Python3 package installer
- [python3-venv](https://docs.python.org/3/library/venv.html) - Python3 package for creating virtual environments, in case you need to briefly use some dependencies that might conflict with your installed libraries 
- [remmina](https://remmina.org/) - an RDP client, alternative to xfreerdp when it didn't want to work for whatever reason
- [rlwrap](https://github.com/hanslub42/rlwrap) - a 'readline wrapper', a small utility that uses the [GNU Readline](https://tiswww.case.edu/php/chet/readline/rltop.html) library to allow the editing of keyboard input for any command, making certain shells more stable/easier to use. 
- [smbmap](https://www.kali.org/tools/smbmap/) - allows users to enumerate samba share drives across an entire domain
- [sublime-text](https://www.sublimetext.com/)- text editor 
- [terminator](https://gnome-terminator.org/) - a simple to use terminal emulator 
- [wpscan](https://www.kali.org/tools/wpscan/) - scan a target WordPress URL and enumerate any plugins that are installed
- [wsgidav](https://www.kali.org/tools/wsgidav/)- a generic and extendable WebDAV server

### GitHub
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)- a ton of useful payloads.
- [SharpCollection](https://github.com/Flangvik/SharpCollection) - a ton of useful binaries for Windows privesc like Rubeus, Sharphound, Snaffler, and SweetPotato.
- [adPEAS](https://github.com/61106960/adPEAS) - Winpeas/Linpeas for Active Directory.
- [Penelope](https://github.com/brightio/penelope)- This is a reverse shell listener with some extended functionality like automatically upgrading shells to Python pty shells and additional commands which allow you to upload and download files directly from the shell. 
- [ConPtyShell](https://github.com/antonioCoco/ConPtyShell)- a stable reverse shell for Windows.
- [LSE](https://github.com/diego-treitos/linux-smart-enumeration) - similar functionality to linpeas, but I personally prefer the output to linpeas. Sometimes I run both, but I always run `lse.sh -l1` first. 
- [Windows Exploit Suggester](https://github.com/AonCyberLabs/Windows-Exploit-Suggester)- it suggest exploits for windows. 
- [Ivan Sincek Reverse Shell](https://github.com/ivan-sincek/php-reverse-shell) - My favorite PHP reverse shell, it should be on your machine. 
- [git-dumper](https://github.com/arthaud/git-dumper) - Dumps git repos from the web, especially useful for those which aren't easily cloned with `git`. 
- [verybasicenum](https://github.com/pentestpop/verybasicenum) - My personal custom enumeration scripts. Simpler and faster than winpeas/linpeas though much less detail. I like to run them first, then the more detailed scripts after.
- [Kerbrute](https://github.com/ropnop/kerbrute)- for brute forcing Kerberos.
- [ILSpy](https://github.com/icsharpcode/ILSpy/releases) - For reverse engineering on AMD64 binaries on ARM machines. Very helpful. 

### Websites/Gitbooks
These are pretty much all of my relevant bookmarks on my Kali machine. You will probably be using most of these, if not all.

- [Crackstation](https://crackstation.net/) - Throw your password hashes in here and see what comes out. Often nothing, but it also often identifies the hash type which is nice.
- [CyberChef](https://gchq.github.io/CyberChef/)- One stop shop for transforming data to and from base64, URL encoding, etc.
- [exploit-db](https://www.exploit-db.com/) - You know this one.
- [GTFOBins](https://gtfobins.github.io/) - You know GTFOBins from the course material, it rocks.
- [HackTricks](https://book.hacktricks.xyz/) - Very much the gold standard. Get used to putting `$searchTerm hacktricks` into google.
- [NTLM.PW](https://ntlm.pw/) - Cracks some NTLM hashes, maybe redundant with crackstation.
- [PayloadsAllTheThings](https://swisskyrepo.github.io/PayloadsAllTheThings/) in gitbook form.
- [RevShells](https://www.revshells.com/) - You probably already know about RevShells, but it allows you to input your IP and listening port and plugs them into a bunch of reverse shells automatically.
- [wadcoms](https://wadcoms.github.io/)- This is a great little tool. It is an interactive cheat sheet, containing a curated list of offensive security tools and their respective commands, to be used against Windows/AD environments. You can select what you have such as a user but no password or an NTLM hash, and then it filters what commands are available. I have worked on a tool to further this work, so we'll see about that in the future. 

Other cheatsheets/gitbooks I have bookmarked:
- [Cheatsheat.haax.fr](https://cheatsheet.haax.fr/)
- [The Hacker Recipes](https://www.thehacker.recipes)
- [S1ckB0y1337 AD Cheat Sheet](https://github.com/S1ckB0y1337/Active-Directory-Exploitation-Cheat-Sheet)
- [Siren Linux Privesc](https://sirensecurity.io/blog/linux-privilege-escalation-resources/)
- [Siren Windows Privesc](https://sirensecurity.io/blog/windows-privilege-escalation-resources/)

To be honest, when you get started you tend to hoard this kind of stuff, but you need to be making your own. I'm not sure I visited any of these in the last month of my studying. That is why I recommend downloading my [notes vault](https://github.com/pentestpop/OSCP_Vault) or someone else's. I used [QuirkyKirkHax](https://github.com/QuirkyKirkHax/OSCP-Resources) CherryTree notes, copied them into an Obsidian Vault, and re-organized them as needed. 

### YouTube
[IppSec](https://www.youtube.com/channel/UCa6eh7gCkpPo5XXUDfygQQA)- He does a ton of lab walkthroughs which are super helpful, espcially if you want to knock some labs off of the TJ Null or LainKusanagi list but can't or won't use your machine for some reason. His website, [ippSec.rocks](https://ippsec.rocks/?#) is awesome too because it has a search tool that links to the specific timestamp and video where he uses it. So for example if you want to know more about dfunc-bypasser, you can search it and get a link to the videos where he uses it. 

![](/assets/images/OSCP_Tricks/ippsec.png){: .responsive-image}

[OffSec](https://www.youtube.com/@OffSecTraining) - A lot of good walkthroughs on these. They tend to be slower I guess, but they also tend to focus more on techniques for the course material. 

[Tyler Ramsbey](https://www.youtube.com/@TylerRamsbey) - This guy is solid too. Less walkthroughs, more general content. He does make some YouTubey faces in his thumbnails, but I like that he clearly reuses them, almost as if he can't bring himself to keep taking those photos.  

There's a ton out there for Cybersecurity in general. The Cyber Mentor, John Hammond, The XSS Rat are good in general, but personally I haven't found their content to be especially helpful when studying for this exam in particular. 

## Closing Thoughts
It's just fun. Do it or don't, I just hope this information helps someone cause I sure had a ton of help from strangers on the internet. 

