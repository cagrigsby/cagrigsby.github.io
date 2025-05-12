---
layout: writeup
title: Hunit - Proving Grounds
date: 2024-07-18 13:32:20 +0300
description: A Writeup of the Hunit Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi]
---

Here's another writeup for Apex, an Intermediate Proving Grounds box I saw on the [LainKusanagi list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). I take note that the community has rated this Very Hard. As usual, I get started with an nmap scan with the -v flag so I can see results as they appear: `sudo nmap -p- -sC -A -v -T5 192.168.172.125`. We notice quickly that there is an open port on 8080 and head there to check it out. 

![Hunit_1.png](/assets/images/Hunit/Hunit_1.png){: .responsive-image}

It looks like a page for posting poetry. I read them all and take note that they are not all haikus, which is...not really interesting I guess. I move on to port 18030 and see this:

![Hunit_2.png](/assets/images/Hunit/Hunit_2.png){: .responsive-image}

Looks like a game. 

At this point I did have to look up how to proceed. I needed to go to the `/api/user?` directory. I'd found the api directory, but you need to use `/api/` not `/api`. Noted. 

![Hunit_3.png](/assets/images/Hunit/Hunit_3.png){: .responsive-image}

We can use one of these to ssh into the target, in particular the one that looks like different than the others, being dademola:ExplainSlowQuest110. This brings us to `/home/dademola` where we can grab local.txt. 

We can't access the web server over port 80, but we can use scp since we know the ssh password. `scp -P 43022 linpeas.sh dademola@192.168.172.125:/home/dademola`. We can also use the web server over port 43022 for example. 

I notice there is a `/home/git` folder, as well as a an `avahi` user in `/etc/passwd`, so we'll keep an eye out for ways to escalate to those users, moreso than the other users from the `/api/users` web directory. Interesting: 

![Hunit_4.png](/assets/images/Hunit/Hunit_4.png){: .responsive-image}

I can get something called a git shell when I ssh into this git user, but I can't seem to execute any commands. I also see that there is a folder called `/home/git/git-shell-commands`, but it's empty. There is also a `/git-server` folder, but I'm not sure what it's for, and I can't seem to use git diff on any of it. I git (get it?) the sense that I need to use the git shell for something, but I'm not sure how it works. The git documentation states that 'a "git> " prompt is presented at which one can enter any of the commands from the git-shell-commands directory, or exit to close the connection.' But there's nothing in that directory. 

I tried adding this, but permission was denied: 
```
[dademola@hunit git]$ cat >git-shell-commands/shell <<\EOF
> #!/bin/bash
> EOF  
```

I'm kind of at a loss because I can't seem to do anything at all with the git shell, and I also can't add anything. 

After a couple of hours I check the writeups again, and they say there is a cronjob running. I apparently couldn't see this because I can't see all of the linpeas output as I don't have another scrollback configured. That's frustrating. I checked the lse output which doesn't show this cronjob, but I didn't both transferring the linpeas.txt file back to my machine because it didn't have nc, so I would have to start openssh. Pure laziness. 

![Hunit_5.png](/assets/images/Hunit/Hunit_5.png){: .responsive-image}

We can't read these files because they are in the root directory. But we can if we clone them I guess? According to writeups we can `git clone file:///git-server/` from the home directory, and then we cd into `/home/git-server` and see this: 

![Hunit_6.png](/assets/images/Hunit/Hunit_6.png){: .responsive-image}

I don't get why. We can also run `git-log` from inside the directory and see the commits, but not from `/git-server`. Not sure why. 

What's happening is that we are able to use the git commands, so we use them to clone what we already know is an existing project. Then we add a new shell to the backups.sh, a reverse shell. Then we try to commit and push to the origin, so that the root will update it. Like so:

1. Download our malicious backups.sh to the machine with scp
2. `git add backups.sh`
3. `git commit -m "whatever"
	1. This won't work, it will make you add a username and email like so:
	2. `git config --global user.name "$user$`
	3. `git config --global user.email ""$email`
		1. You can omit global to only set the identity in this directory
4. then `git commit -m "whatever"` again (it will work)
5. `git push -u origin` - if this doesn't work, you need a new user, in this case the git ssh user

At that point the writeups had to google how to `git clone` using an ssh key and come up with this command:
1.  `GIT_SSH_COMMAND='ssh -i private_key_file -o IdentitiesOnly=yes' git clone user@host:repo.git`
	1. So: `GIT_SSH_COMMAND='ssh -i git.ssh -p 43022 -o IdentitiesOnly=yes' git clone git@192.168.172.125:/git-server/`
	2. Then update the backups.sh file with our malicious script and `git add backups.sh` and `git commit -m "whatever2"` again. 
2. Then push: `GIT_SSH_COMMAND='ssh -i ../git.ssh -p 43022' git push -u origin`
3. After a while, the backups.sh script runs, and we get a root shell. 

![Hunit_7.png](/assets/images/Hunit/Hunit_7.png){: .responsive-image}

And we're done. Lessons learned - I couldn't read linpeas in my terminal, and I assumed it didn't contain materially different information from my lse script. That was wrong, and it cost me a lot of time and a hint. And it was just lazy, I could have used my default terminal rather than terminator, and I still can maybe change the terminator scrollback even more than I rleady have. I was worried to do that because it can supposedly cause performance issues, but I really haven't been all that close to that yet, and this is the second time this has happened. I also got stuck trying to use the git-shell because I didn't realize I could `git clone`, `git add`, `git ccmmit`, and `git push origin` over ssh using the git-shell's key. I don't feel nearly as bad about this because I haven't run into it before. This lab *was* the learning opportunity, and I have taken it. The scrollback thing was embarrassing though. 
