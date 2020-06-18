---
layout: post
title:  "View2akill: Vulnhub Walkthrough"
date:   2019-11-11 15:07:19
categories: [tutorial]
comments: true
---
This is a boot2root machine hosted on [VulnHub][1] that was created by [creosote][2] and is part of a series called `View2aKill`. Its graded as intermediate. As this is purely for educational purposes I'll throw in spoilers now. To exploit this machine we'll need to chain two different exploits, there will be some port forwarding and we'll need to exploit a second machine. At the end we'll also include some mitigation strategies. Without much further ado here we go. Word from the author.
```
Mission: Millionaire psychopath Max Zorin is a mastermind behind a scheme to destroy Silicon
Valley in order to gain control over the international microchip market. Get root and stop
this madman from achieving his goal!

Difficulty: Intermediate
Flag is /root/flag/flag.sh
Use in VMware. DHCP enabled.
Learning Objectives: Web Application Security, Scripting, Linux enumeration and more.
```
<!--more-->
## Information Gathering
Do an nmap scan:


Bruteforce directory attack..
```
gobuster dir -w /usr/share/wordlists/dirb/common.txt -l -t 30 -e -k -x .html,.php -u http://192.168.109.145:80 -o recon/gobuster_192.168.109.145_80.txt

....
http://192.168.109.145:80/.htpasswd (Status: 403) [Size: 280]
http://192.168.109.145:80/.htpasswd.html (Status: 403) [Size: 280]
http://192.168.109.145:80/.htpasswd.php (Status: 403) [Size: 280]
http://192.168.109.145:80/.htaccess (Status: 403) [Size: 280]
http://192.168.109.145:80/.htaccess.html (Status: 403) [Size: 280]
http://192.168.109.145:80/.htaccess.php (Status: 403) [Size: 280]
http://192.168.109.145:80/.hta (Status: 403) [Size: 280]
http://192.168.109.145:80/.hta.html (Status: 403) [Size: 280]
http://192.168.109.145:80/.hta.php (Status: 403) [Size: 280]
http://192.168.109.145:80/dev (Status: 301) [Size: 316]
http://192.168.109.145:80/index.html (Status: 200) [Size: 194]
http://192.168.109.145:80/index.html (Status: 200) [Size: 194]
http://192.168.109.145:80/joomla (Status: 301) [Size: 319]
http://192.168.109.145:80/pics (Status: 301) [Size: 317]
http://192.168.109.145:80/robots.txt (Status: 200) [Size: 83]
http://192.168.109.145:80/server-status (Status: 403) [Size: 280]
==============================================================
```
Interesting links:
http://192.168.109.145:80/dev
http://192.168.109.145:80/joomla
http://192.168.109.145:80/pics

```

## Low Privilege shell

## Privilege Escalation
[1]: https://www.vulnhub.com/entry/view2akill-1,387/
[2]: https://twitter.com/@_creosote
[3]: https://www.vulnhub.com/
[4]: https://github.com/21y4d/nmapAutomator
[5]: https://twitter.com/@4nqr34z
