---
layout: writeup
title: Plum - Proving Grounds
date: 2024-06-23 13:32:20 +0300
description: A Writeup of the Plum Box from Proving Grounds
image: # /assets/images/Plum/Plum_5.png
fig-caption: # Add figcaption (optional)
tags: [Linux, TJ Null]
---

Here is a writeup for the Plum lab on [Proving Grounds](https://www.offsec.com/labs/), another box from TJ Null's OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#). I got started as usual by scanning for open ports, but while I did I also checked port 80 by simply pasting the target IP in into the address bar, and voila. It looks like we have some kind of blog running on a web service called pluxml. I run a directory scan and check for exploits.

![Plum_5.png](/assets/images/Plum/Plum_5.png){: .responsive-image}

The directory scan shows a `http://192.168.174.28/core/admin/auth.php?p=/core/admin/` so I check it out and see that it a login screen to the admin portal. 

![Plum_6.png](/assets/images/Plum/Plum_6.png){: .responsive-image}

As usual I check admin:admin just in case, and I'm able to log in. That should help with any authenticated exploits and a number of file upload vulnerabilities. I'll check for those as well as the ability to edit any files. Given that it's a blog, maybe I get put some executable php in there. (Side note, I noticed from `http://$targetIP/readme/CHANGELOG` that we're dealing with PLUXML 5.8.7.)

![Plum_7.png](/assets/images/Plum/Plum_7.png){: .responsive-image}

That part is actually pretty simple. After clicking around for a bit, I find the themes page which allows me to edit some of the pages which show php. I decide to add a php reverse shell to - home.php. 

![Plum_1.png](/assets/images/Plum/Plum_1.png){: .responsive-image}

And we get a shell!

![Plum_2.png](/assets/images/Plum/Plum_2.png){: .responsive-image}

I grab local.txt from `/var/www` and start looking around for privesc opportunities. I can't run sudo, so I download linpeas and lse (linux-suggested-exploits), autoscan tools. Nothing really jumps out except that I could maybe try DirtyPipe or AutoPwn, the machine is listening to port 25 on localhost, and the user has mail for some reason. That's from lse, and I'm not sure what it means. It looks like there is mail at `/var/mail/www-data`. I check it out and find this: 

![Plum_3.png](/assets/images/Plum/Plum_3.png){: .responsive-image}

Maybe it's a clue!

![Plum_4.png](/assets/images/Plum/Plum_4.png){: .responsive-image}

I switch to the root user and grab the flag. Lessons learned - no exploits needed, just use your head on the services you have access to and check out interesting directories flagged by the auto recon tools. I never really check the mail folders on my own, now I know at least that they get flagged. 

