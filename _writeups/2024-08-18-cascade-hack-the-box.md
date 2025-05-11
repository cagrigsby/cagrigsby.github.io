---
layout: writeup
title: Cascade - HackTheBox
date: 2024-08-18 13:32:20 +0300
description: A Writeup of the Cascade Box from HackTheBox
image: # /assets/images/Cascade/Cascade.png
fig-caption: # Add figcaption (optional)
tags: [TJ Null, LainKusanagi, Windows, Active Directory]
---


Today I'm doing a writeup for a [Hack The Box](https://app.hackthebox.com/profile/2013658) box from both TJ Null’s OSCP [lab prep list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#)and LainKusanagi’s [list of OSCP like machines](https://www.reddit.com/r/oscp/comments/1c8pzyz/lainkusanagi_list_of_oscp_like_machines/). It is called Cascade, and it is rated Medium by HackTheBox. As usual, we get started with an nmap scan. I'm using my own [custom script](https://github.com/pentestpop/verybasicenum/blob/main/vbnmap.sh) for this which (gives more detail but) shows these open ports:

```
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
49154/tcp open  unknown
49155/tcp open  unknown
49157/tcp open  unknown
49158/tcp open  unknown
49165/tcp open  unknown
```

It looks like we have a domain controller with no web ports. I check the detailed output (or `nxc smb 10.10.10.182 -u '' -p ''`) to see that the domain name is `cascade.local` which I add to my `/etc/hosts` file. I am able to grab a list of users from `enum4linux -a cascade.local`, which after using the cut command a few times, looks like this:

```
CascGuest
arksvc
s.smith
r.thompson
util
j.wakefield
s.hickson
j.goodhand
a.turnbull
e.crowe
b.hanson
d.burman
BackupSvc
j.allen
i.croft
```

I also notice that the description for the CascGuest user is: "Built-in account for guest access to the computer/domain." And there are a few interesting users and groups that stick out from the enum4linux output as well:

```
Group: AD Recycle Bin' (RID: 1119) has member: CASCADE\arksvc
Group: Audit Share' (RID: 1137) has member: CASCADE\s.smith
Group: IT' (RID: 1113) has member: CASCADE\arksvc
Group: IT' (RID: 1113) has member: CASCADE\s.smith
Group: IT' (RID: 1113) has member: CASCADE\r.thompson
...
Group: Remote Management Users' (RID: 1126) has member: CASCADE\arksvc
Group: Remote Management Users' (RID: 1126) has member: CASCADE\s.smith
```

I try a few different things with impacket and ultimately wind up getting more information through ldapsearch, first by trying to search all object classes and then narrowing to users with `ldapsearch -x -H ldap://10.10.10.182 -D 'CN=CascGuest,DC=cascade,DC=local' -W -b 'DC=cascade,DC=local' 'objectClass=user'`. There is still a ton of noise, but we do find what appears to be a password at the end of the r.thompson user. 

![Cascade2.png](/assets/images/Cascade/Cascade2.png){: .responsive-image}

I realize it's actually base64 encoded after trying to figure out what kind of hash it is for a bit. I use CyberChef to decode it and get `rY4n5eva`. With nxc, I confirm that the creds are valid as `r.thompson:rY4n5eva`. So I check smbclient again to list the shares, and I'm able to (not possible without valid creds):
```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
Audit$          Disk      
C$              Disk      Default share
Data            Disk      
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share 
print$          Disk      Printer Drivers
SYSVOL          Disk      Logon server share 
```

Off the bat the `Data` share seems the most promising so I check that out. We are able to access a folder call `IT` inside this share, which makes sense because `r.thompson` is a part of the `IT` group according to enum4linux. Inside we have a file called `Meeting_Notes_June_2018.html` which gives us some interesting info about an account:

![Screenshot 2024-10-21 at 11.22.39 PM.png](/assets/images/Cascade/Screenshot 2024-10-21 at 11.22.39 PM.png){: .responsive-image}

We're not sure what the normal admin password is, but we do have a potentially high-privilege account. It's worth also noting that when I add this `TempAdmin` user to the users.txt file, I can not actually confirm its existence with kerbrute's userenum function. So maybe it is a local account.

Continuing through the SMB share, we also see a file called `VNC Install.reg` which seems like it could have a password. 
![Cascade3.png](/assets/images/Cascade/Cascade3.png){: .responsive-image}

Writeups for this box came up when I tried to google it, which isn't uncommon, but it could be a pretty good if kinda spoilery sign. Doing some more googling around the file name, I find this github ["repo"](https://github.com/billchaison/VNCDecrypt)which just contains a simple script to decode it. The script looks like this: `echo -n d7a514d8c556aade | xxd -r -p | openssl enc -des-cbc --nopad --nosalt -K e84ad660c4721ae0 -iv 0000000000000000 -d -provider legacy -provider default | hexdump -Cv`, but I swap out the `d7a514d8c556aade` with `6bcf2a4b6e5aca0f` from the highlighted portion above, and it works: 

![Cascade4.png](/assets/images/Cascade/Cascade4.png){: .responsive-image}

We get a password of `sT333ve2`. We can guess this password goes to `s.smith` as it is in their folder, but we check against all of the usernames. We run `nxc winrm 10.10.10.182 -u users.txt -p sT333ve2 --continue-on-success` and confirm that these are the correct credentials and that we are able to access the machine via winrm and grab user.txt.

![Cascade5.png](/assets/images/Cascade/Cascade5.png){: .responsive-image}

At this point I spend a lot of time trying different things with Rubeus, mimikatz, winpeas, adPEAS, and bloodhound. None of them amount to much, except I do notice that we're working with Windows Server 2008, which is old enough that there's going to be some exploits for it. 

![Cascade6.png](/assets/images/Cascade/Cascade6.png){: .responsive-image}

I don't really like to look for OS exploits in labs like these, often they are not the point. So I don't spend much time with them here. I do notice that our user `s.smith` is in the Audit Share and IT groups, but I don't know what they do. When we run `net user s.smith`, we see a Logon script called `MapAuditDrive.vbs`, and searching for it with `Get-ChildItem` we find it in the `C:\Windows\SYSVOL\domain\scripts` directory looking like this:

![Cascade7.png](/assets/images/Cascade/Cascade7.png){: .responsive-image}

I notice that it specifically calls out a share called `Audit$`, a share which I was not able to enumerate with the `r.thompson` user. When I enter the share with smbclient and the `s.smith` user, I see these files and download them for inspection. 

![Cascade8.png](/assets/images/Cascade/Cascade8.png){: .responsive-image}

The Audit.db file in particular is interesting and contains a table called Ldap which seems to show a password, or at least a password hash, for the `ArkSvc` user. 

![Cascade9.png](/assets/images/Cascade/Cascade9.png){: .responsive-image}

It looks like base64, but I can't seem to decode it, and nth and crackstation don't know what to do with it either. 

To be honest I got stuck here, and I needed to look for a nudge. I kinda feel ok about it though because I learned to use a new tool that I didn't know was possible. All of the writeups I found required you to reverse engineer the CascAudit.exe binary, and they all suggested using [dnsSpy](https://github.com/dnSpy/dnSpy). This tool is not available for ARM machines which I am using. What is available to use though is [ILSpy](https://github.com/icsharpcode/ILSpy/releases)This is an open-source .NET assembly browser and decompiler. I had learned to do some of this while taking the PEN-200 course but skimmed over it thinking that it wasn't possible for my machine, and I would have to re-learn it when I get a new one. Good new though, I got a chance to do it now. For reference, you simply download and unzip the latest release [here](https://github.com/icsharpcode/ILSpy/releases) and then simply run `./ILSpy` and open the binary. From there we check out the main function - `Main(): void`.

![Cascade10.png](/assets/images/Cascade/Cascade10.png){: .responsive-image}

One thing that sticks out is that there is what appears could be a password, but is actually a key that the actual password is encrypted with. The binary is using a function called `Crypto.DecryptString` to decrypt the Pwd found in the database with this key. If I'm able to find that function, I may be able to do the same thing. We search through the code and find it:

![Cascade11.png](/assets/images/Cascade/Cascade11.png){: .responsive-image}

We can see from the highlighted line that it is taking two arguments, the EncryptedString (which will be`BQO5l5Kj9MdErXx6Q6AGOw==` for us) and the key (`c4scadek3y654321`). Unfortunately, I can't simply rerun this code in an [online compiler](https://www.programiz.com/csharp-programming/online-compiler/)with the parameters replaced as it either gets errors or does not print to the console. 

I fiddled with this for a while, wanting to avoid looking up the answers from here and also avoid using ChatGPT. That said, I don't have any familiarity with C# and couldn't get anything to work. One option from [0xdf](https://0xdf.gitlab.io/2020/07/25/htb-cascade.html) was to simply run the code from a Windows host with a breakpoint, but again, I am on an ARM machine, and I dont have this option. I looked up a few other writeups, but they all presented their full code either with their own [C#](https://medium.com/@mdfrigillana/hack-the-box-walkthrough-cascade-aec1ac3bb07a) or a [python library called pyaes](https://medium.com/@Poiint/htb-cascade-write-up-8e8cf4934e99). Unfortunately, in either case, I would have needed ChatGPT to get me over the line. Given that I am doing this primarily to study for OSCP, I want this to be a no-go, but on the other hand, this is outside the scope of what is required for the OSCP, so I'm simply going to move on after a few hours. Here is my code:

```
using System;
using System.IO;
using System.Security.Cryptography;
using System.Text;

class Program
{
    static void Main(string[] args)
    {
        // Provide the variables for EncryptedString and Key
        string encryptedString = "BQO5l5Kj9MdErXx6Q6AGOw==";
        string key = "c4scadek3y654321";

        // Call the DecryptString method and print the result
        string decryptedString = DecryptString(encryptedString, key);
        Console.WriteLine($"Decrypted Password: {decryptedString}");
    }

    static string DecryptString(string EncryptedString, string Key)
    {

        byte[] array = Convert.FromBase64String(EncryptedString);
        Aes aes = Aes.Create();
        aes.KeySize = 128;
        aes.BlockSize = 128;
        aes.IV = Encoding.UTF8.GetBytes("1tdyjCbY1Ix49842");
        aes.Mode = CipherMode.CBC;
        aes.Key = Encoding.UTF8.GetBytes(Key);

        using MemoryStream stream = new MemoryStream(array);
        using CryptoStream cryptoStream = new CryptoStream(stream, aes.CreateDecryptor(), CryptoStreamMode.Read);
        byte[] array2 = new byte[checked(array.Length - 1 + 1)];
        cryptoStream.Read(array2, 0, array2.Length);

        string result = Encoding.UTF8.GetString(array2);
        return result;
    }
}
```

It returns `Decrypted Password: w3lc0meFr31nd`. So now we have the `ArkSvc`:`w3lc0meFr31nd`. Moving on. 

I use `nxc` to confirm we have winrm access with these credentials and then use `evil-winrm` to get a shell. I don't see anything immediately interesting with privileges, but when I run `net user arksvc` I see that we are a part of the 'AD Recycle Bin' group, so that could be interesting. There is a command from [hacktricks](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/privileged-groups-and-token-privileges) to try running, but it doesn't work. 

At this point I tried a few different things, and even caved and asked ChatGPT how to recover these deleted objects. Nothing worked. So I reverted the machine and the hacktricks command worked: `Get-ADObject -filter 'isDeleted -eq $true' -includeDeletedObjects -Properties *`. And we get this line that sticks out pretty clearly given that we know the TempAdmin password was the same as the normal Administrator password. 

![Cascade12.png](/assets/images/Cascade/Cascade12.png){: .responsive-image}

I decode it with base64 again and get `baCT3r1aN00dles`. Then I simply use evil-winrm to get a shell as the Administrator. 

![Cascade13.png](/assets/images/Cascade/Cascade13.png){: .responsive-image}

And I grab the root.txt file and we're done!

### Lessons Learned
- I learned how to use [ILSpy](https://github.com/icsharpcode/ILSpy/releases), that could be helpful in the future. It sounds like the more efficient way to get to the next step of this box would have been to run the binary with a breakpoint and check local variables, but that wasn't possible for me at the moment. Still good to know though. 
- I was reminded to check the user (`net user $user`), as I often don't do that. In this case it was crucial to find the logon script. Realistically I may have been able to figure out the next step by checking the available smb shares which I hadn't yet done with the `s.smith` user. 
- I learned how to decrypt passwords stored in VNC files. Hadn't done that. 
- I was reminded to check carefully through the output of ldapsearch. To be honest I may have gotten stuck here if I hadn't suspected I needed to look through all of the users. Now when you start up a HTB lab, it defaults to "Guided Mode" and showed me the first question of what is the username of Ryan Thompson. I'm wondering if I hadn't seen though would I have known to look there. I did take the output of my ldapsearch queries and grep for password, username, pass, and a few other things, but the variable was `cascadeLegacyPwd`, so I never saw it. There was a ton of output to look through, so hopefully I can remember to take the time to do that in the future. 

### Remediation Steps
- Remove old passwords from Active Directory Objects, and don't store them there in the future, even base64 encoded. 
- Remove them from the `VNC Install.reg` file as well. They shouldn't be available to anyone even encrypted, especially when they can be decrypted easily through publicly available means. 
- The Audit share had both a database file (no password) which stored an encrypted and encoded password, and the program which made it possible to decrypt it (and decode it). It's unclear what the use case could be for something like this, but the database containing sensitive data should be protected, and the executable which decrypts it should not be so available for users to reverse engineer. Because the use case is unclear, it's hard to know the remediation steps, but there are a few options:
	- Alter the binary such that the key is not available to view when reverse engineering - maybe it could be randomly generated.
	- That binary decrypts a password from a database - make the password much stronger. 
	- The `ArkSvc` user that the password comes from needs to have privileges revoked according to the principle of least privilege. This ostensible service account should not be able to log on through WinRM, it should just be allowed to run the commands it was created for. 
	- A stronger encryption algorithm could be used such that it can't be decrypted so easily when the key is known. 
	- Ultimately there are a lot of ways to mitigate the risk here, but it's just unclear which is the best not knowing the circumstances. It's possible the Binary should be run at all, and there should be another method to accomplish the same task.