---
layout: writeup
title: Marketing - Proving Grounds
date: 2024-07-15 13:32:20 +0300
description: A Writeup of the Marketing Box from Proving Grounds
image: # /assets/images/Flu/Flu_1.png
fig-caption: # Add figcaption (optional)
tags: [Linux, LainKusunagi]
---

Here's a writeup for Marketing, an Intermediate Proving Grounds box I saw on the [LainKusanagi list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). I take note that the community has rated this Very Hard. It was one of the harder boxes I've done to prepare for the OSCP to be honest, so that tracks. It did require writeups for me to complete, but I don't think I looked them up too early. It was simply above my level of understanding. 

As usual, I get started with an nmap scan with the -v flag so I can see results as they appear: `sudo nmap -p- -v -T4 192.168.216.225`. The first interesting thing (besides port 22 - ssh and often used for administration of labs) is that port 80 is open (only those two ports in fact), so I go to check it out in the browser and start a directory scan with feroxbuster just in case. 

![Marketing1.png](/assets/images/Marketing/Marketing1.png){: .responsive-image}

I can't really find anything interesting on the main page, but feroxbuster shows a `$IP/old/` directory, and that sounds like it could be interesting, so I go check that out. Also I notice that if I click anything click anything, the URL bar shows this: `http://192.168.216.225/#[object Object]`. I don't really know what that means, but it didn't happen in the main page, so that's at least a clue we're on the right track. It shows the same when going to `$IP/index_old.html#[object%20Object]`. [This](https://stackoverflow.com/questions/64907280/html-website-url-contains-object-object-when-navigating) stackoverflow post says that it's trying to assign an object where only strings are valid. I spend some time trying to figure out how to pursue this path, but eventually give up and check a [writeup](https://medium.com/@ardian.danny/oscp-practice-series-32-proving-grounds-marketing-bf040837eeff). 

It had nothing to do with the `[object Object]`. There was a site mentioned in the source code of the /old site called `customers-survey.marketing.pg`. I couldn't visit it until I added it to the `/etc/hosts` file. I thought that meant that I can't go to the original site, but it means that if I type `http://$IP` I get the original site, but the `customers-survey.marketing.pg` goes to a different site. Lesson learned, I didn't know that. It goes to this page: 

![Marketing2.png](/assets/images/Marketing/Marketing2.png){: .responsive-image}

Apparently it is an open source survey tool, and it shows 14 exploits on exploit-db. While checking [this](https://www.exploit-db.com/exploits/48297) traversal vuln, I get taken to a login page. The remaining exploits from the last 10 years seem to be mostly authenticated, so hopefully I can find my way in here. I try a few combinations, and `admin:password` works. Bingo. 

I tried the [most recent exploit](https://www.exploit-db.com/exploits/50573)without thinking much, which unsurprisingly didn't work. I go into the app to enumerate and determine we are on `Version 5.3.24`, later than any of the listed exploits on exploit-db. I looked around the app for a while and googled the service+version+exploit which led me to a [github page](https://github.com/Y1LD1R1M-1337/Limesurvey-RCE) for the exploit. At that point I realized that the exploit from exploit-db was the automated version, but there were steps which involved uploading and activating a malicious plugin with a reverse shell which could then be called. The python itself contained code that needed to be changed. That's normal, but it in fact required other other files, so the code from exploit db would never have worked on its own. Should have read through it more carefully. Regardless, I cloned and used the exploit, and I got a shell. 

![Marketing3.png](/assets/images/Marketing/Marketing3.png){: .responsive-image}

I notice there is a mysql database open and looking around for credentials.  After looking around I noticed this in `/var/www/LimeSurvey/applications/config/config.php`.

![Marketing5.png](/assets/images/Marketing/Marketing5.png){: .responsive-image}

We can use those credentials to log into a MySQL db. Once I log in, I see the database "limesurvey" and decide to enumerate that, but I don't see anything immediately interesting, and the tables I check are empty, so I decide to sort the tables by their size so at least I can tell which ones to check. I use this command:
```
SELECT table_name AS "Table",
ROUND(((data_length + index_length) / 1024 / 1024), 2) AS "Size (MB)"
FROM information_schema.TABLES
WHERE table_schema = "limesurvey"
ORDER BY (data_length + index_length) DESC;
```

And I see these non-zero tables:
```
+---------------------------------------+-----------+
| Table                                 | Size (MB) |
+---------------------------------------+-----------+
| lime_question_themes                  |      0.03 |
| lime_surveymenu_entries               |      0.02 |
| lime_templates                        |      0.02 |
| lime_template_configuration           |      0.01 |
| lime_surveymenu                       |      0.01 |
| lime_users                            |      0.01 |
| lime_permissions                      |      0.01 |
| lime_notifications                    |      0.01 |
| lime_surveys_groups                   |      0.01 |

```

Ultimately I find nothing interesting in these except the password hash for the admin user, which we already know is "password". Maybe I can use the password for something else. I try to `su` into root, m.sander, and t.miller, and t.miller works. When I run `sudo -l` on t.miller, I see this: 
```
User t.miller may run the following commands on marketing:
    (m.sander) /usr/bin/sync.sh
```


I check out this file and apparently it updates `/home/m.sander/personal/notes.txt` if there is any difference between that and the target file. Apparently it can only be run on target files m.sander can read though, and we don't know what those are. Not sure what to do with that. 

I got a hint. Shame on me. 

I guess I should have noticed in linpeas that we belong to the mlocate group and this shows up in the output. Apparently the mlocate group allows us to read the database of indexed files, located at: `/var/lib/mlocate/mlocate.db`.

![Marketing6.png](/assets/images/Marketing/Marketing6.png){: .responsive-image}

When I tried to read this file, it showed a bunch of files with no formatting and crashed my terminal. Apparently we were supposed to find a file called `/home/m.sander/personal/creds-for-2022.txt`, create a symlink for it, and then run sync.sh on that. The writeup I saw had to look that up, and neither that [writeup](https://medium.com/@ardian.danny/oscp-practice-series-32-proving-grounds-marketing-bf040837eeff) author nor I can even figure out how we could know that. You can't cat the mlocate.db and grep for the file (or run strings) even if you know it's there, it didn't find anything. Not sure how to do this correctly. 

Eventually I try just running the commands I found in that writeup exactly, and they don't work. I try iterating around them, like removing the symlink step for example, but nothing works. This was the point at which I surrendered and looked up the official walkthrough from Proving Grounds, which didn't exactly clarify anything. Problems:

1. For one, they give this command to find the files inside `/home/m.sander/personal`: 
`cat /var/lib/mlocate/mlocate.db | strings | grep personal -A 3 -B 3`. Why is this a problem? The `strings` command doesn't exist on the machine. "But you can transfer it to your machine and run strings on it!" you say. Sure, but you wouldn't transfer it to your own `/var/lib/mlocate/` folder would you? That would be stupid, so why the full path? There's also no indication that the file was transferred in the writeup, though I'm not going to screenshot it in case I get in trouble. 

1. Ok, suppose you know to do that, and to be fair, I will next time. They then give these two commands to use to run `sync.sh` command and read the files. 
	1. `ln -sf /home/m.sander/personal/creds-for-2022.txt abc` - This creates the sym link.
	2. `sudo -u m.sander /usr/bin/sync.sh abc` - This runs the script against the sym link. 
	It still takes me a while to get the correct output. You can't create the sym link in `/home/m.sander` (shown below) or `/tmp` (not shown). I also got a forbidden error, and a no changes output at different times. Unfortunately I don't have screenshots of that because I reverted the box multiple times. Regardless, I do eventually get the correct output which gives me new creds to try. 

![Marketing7.png](/assets/images/Marketing/Marketing7.png){: .responsive-image}

I'm able to `su m.sander` using `EzPwz2022_12345678#!`. Right when I log in, I see `To run a command as administrator (user "root"), use "sudo <command>"`, so I run `sudo su`, and get a root shell. 
	
![Marketing8.png](/assets/images/Marketing/Marketing8.png){: .responsive-image}

Ultimately this was a really frustrating box. I missed the `customers-survey.marketing.pg` URL in the source code of the web page, and I didn't understand that adding that with $IP to the `/etc/hosts` file would allow us to go to a different page than simply visiting $IP in the browser. This helped me to better understand name-based virtual hosting - the server running two websites over the same port and returning the correct site based on the `Host` header sent in the HTTP request. I didn't get that. I also didn't understand notice the mlocate group or know what it could do. I didn't *really* understand the sync.sh script was looking only for files in the m.sander directory, though it makes sense now. I didn't understand that I could create a sym link using a file I can't read, but only in specific directories apparently? I still don't really understand that part, and writeups and ChatGPT haven't helped. So lessons learned: 

1. Name-based virtual hosting means multiple sites can run over the same port on a web server
2. The mlocate group allows us to read the database of indexed files, located at: `/var/lib/mlocate/mlocate.db`.
3. Go over the source code of discovered websites more carefully. 
4. You can create sym links of files you can't read. 

Rough box. I might put this writeup online just to show that learning is a process, but this was not my best work. 


