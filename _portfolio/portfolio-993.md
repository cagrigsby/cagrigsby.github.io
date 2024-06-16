---
title: "Ruby Dome - Proving Grounds"
excerpt: "A Writeup of the Ruby Dome Box from Proving Grounds<br/><img src='/images/RubyDome/RubyDome.png'>"
collection: portfolio
---

Back with another [Proving Grounds](https://www.offsec.com/labs/) box from TJ Null's OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#). This one is called "RubyDome" which could give us a little preview on a certain coding language which might be involved.  

This time I kicked things off with [nmapAutomator](https://github.com/21y4d/nmapAutomator) from 21y4d. I like this tool; it runs pretty fast and it's pretty simple to extend from port scanning to vuln scanning. There may be better tools out there, but I'll be using this one for a bit for sure. 

    PORT     STATE SERVICE VERSION
    22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey: 
    |   256 b9:bc:8f:01:3f:85:5d:f9:5c:d9:fb:b6:15:a0:1e:74 (ECDSA)
    |_  256 53:d9:7f:3d:22:8a:fd:57:98:fe:6b:1a:4c:ac:79:67 (ED25519)
    3000/tcp open  http    WEBrick httpd 1.7.0 (Ruby 3.0.2 (2021-07-07))
    |_http-server-header: WEBrick/1.7.0 (Ruby/3.0.2/2021-07-07)
    |_http-title: RubyDome HTML to PDF
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

<br>

Looks like we got a web page on port 3000, I'll check that out while I keep using run more scans in the background. 
<br>
![RubyDome Proof](/images/RubyDome/RubyDome.png){: .center-aligned width="600px"}
<br>
Ok, looks like a pretty basic tool which converts HTML pages to PDFs for download. At this point I checked for exploits for WEBrick/1.7.0 and Ruby/3.0.2, and I found a few, but I couldn't get them working, even to a point where it would make a lot of sense to take a screenshot. This must be the valuable time spent failing on these boxes, because I spent hours looking for exploits for different tools in this lab that went nowhere. At some point while I did this, I figured it also made sense to upload a sample html page and see what happened. So I created a simple one and uploaded it from a python server. I checked Burp Suite while I uploaded it, finding nothing super interesting. Then I used exiftool to analyze the resulting pdf and see if I could learn anything. 

<br>
![RubyDome Proof](/images/RubyDome/exiftool.png){: .center-aligned width="600px"}
<br>
Hey that could be something. I found an exploit for wkhtmltopdf on exploit-db, and I spent some time working on that, but I got nowhere there either. I noticed after you point the webpage to an HTML page and click "Convert to PDF" as expected, you get moved to a new page which looks like this:
<br>
![RubyDome Proof](/images/RubyDome/pdfkit.png){: .center-aligned width="600px"}
<br>
So now we have PDFKit, maybe there's an exploit for that. And there [was(https://www.exploit-db.com/exploits/51293)]! It took me a little bit to figure this one out too. Basically the issue here is that if you only use the "-s" flag for reverse shell mode, the exploit actually spits out what it calls a payload to upload to the online tool. I tried that for a while, but I couldn't get it working. Maybe it would have worked, but I should have put $IP:3000/pdf instead of just $IP:3000/. Regardless, I did get it working with the "-w" which allows you to input the target URL so the tool will send the payload itself. It also requires a "-p" flag for parameter, and it took me a while to figure out the parameter in that case was URL. Oh well. Eventually we got it figured out and recieved the shell. 
<br>
![RubyDome Proof](/images/RubyDome/shell_caught.png){: .center-aligned width="600px"}
<br>
After that, I looked around for a while, got the local.txt flag, and eventually uploaded linpeas for a boost. Below are the lines which stuck out to me:
<br>
![RubyDome Proof](/images/RubyDome/shell_caught.png){: .center-aligned width="600px"}
<br>
Ok - so we need to use Ruby - like RubyDome - got it. I just needed to overwrite /home/andrew/app/app.rb with something useful and run it with sudo. Looking back, I should have just execute bash from there, but I apparently decided to waste my own time trying several Ruby reverse shells. It took me three before I got one working, but it did. I used [this one](https://gist.github.com/gr33n7007h/c8cba38c5a4a59905f62233b36882325) from gr33n7007h. 

![RubyDome Proof](/images/RubyDome/proof.png){: .center-aligned width="600px"}

Bingo. This lab was definitely a test in trying harder, not because the particular lab was so difficult, but because I kept going down the wrong path. Hopefully in the future I can continue to get better at zeroing in on the information I actually need to succeed. I will say I learned one more thing from this lab though. I showed the linpeas output before which says:

    User andrew may run the following commands on rubydome: (ALL) NOPASSWD: /usr/bin/ruby /home/andrew/app/app.rb
<br>
I didn't realize that you have to give the full path for the app if you want to run sudo without the password. I was in the /home/andrew/app directory, and I couldn't get the privesc working for that reason. Now I know for next time. 

![RubyDome Proof](/images/RubyDome/valuable.png){: .center-aligned width="600px"}

Thanks for reading!