---
layout: post
title: Kioptrix Level 1 (#1) Writeup
excerpt: Kioptrix Level 1 is a simple boot-to-root VulnHub box that is vulnerable to a remote code execution vulnerability impacting its Samba service. This box is a great beginner test to learn basic port enumeration and exploitation.
tags: VulnHub
comments: true
social-share: false
---

In preperation for my OSCP exam, I tackled a series of VulnHub boxes, Kioptrix Level 1 being the first. In retrospect, this box is _really_, _really_ old. Like _2010_ old, so it's real-life applicability is questionable. 

It is a simple boot-to-root box that is vulnerable to a remote code execution vulnerability impacting its Samba service.

It's good baseline prep for anyone studying for their OSCP/PWK, learning Network pentesting, and trying to get some practice enumerating a box.

I'll assume you've already got the box running and are able to reach it. Let's get going.

# Enumeration

### NMAP Scans

```bash
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-26 23:04 EDT
Nmap scan report for 10.0.0.35
Host is up (0.00013s latency).
Not shown: 65529 closed ports
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 2.9p2 (protocol 1.99)
| ssh-hostkey: 
|   1024 b8:74:6c:db:fd:8b:e6:66:e9:2a:2b:df:5e:6f:64:86 (RSA1)
|   1024 8f:8e:5b:81:ed:21:ab:c1:80:e1:57:a3:3c:85:c4:71 (DSA)
|_  1024 ed:4e:a9:4a:06:14:ff:15:14:ce:da:3a:80:db:e2:81 (RSA)
|_sshv1: Server supports SSHv1
80/tcp   open  http        Apache httpd 1.3.20 ((Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-title: Test Page for the Apache Web Server on Red Hat Linux
111/tcp  open  rpcbind     2 (RPC #100000)
139/tcp  open  netbios-ssn Samba smbd (workgroup: MYGROUP)
443/tcp  open  ssl/https   Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-title: 400 Bad Request
|_ssl-date: 2020-07-27T07:05:40+00:00; +3h59m59s from scanner time.
| sslv2: 
|   SSLv2 supported
|   ciphers: 
|     SSL2_RC4_64_WITH_MD5
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|     SSL2_RC2_128_CBC_WITH_MD5
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|     SSL2_RC4_128_WITH_MD5
|_    SSL2_DES_64_CBC_WITH_MD5
1024/tcp open  status      1 (RPC #100024)
MAC Address: 08:00:27:A0:57:DB (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 2.4.X
OS CPE: cpe:/o:linux:linux_kernel:2.4
OS details: Linux 2.4.9 - 2.4.18 (likely embedded)
Network Distance: 1 hop

Host script results:
|_clock-skew: 3h59m58s
|_nbstat: NetBIOS name: KIOPTRIX, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
|_smb2-time: Protocol negotiation failed (SMB2)

TRACEROUTE
HOP RTT     ADDRESS
1   0.13 ms 10.0.0.35

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 80.55 seconds
```

Cool, we see quite a few interesting possibilities here. Let's start from bottom to top manually enumerating each open port.


### Enumerating SSH

We can use Netcat to quickly grab the service banner on port 22. You'll notice that this appeared on NMAP scans too, but manually fingerprinting the service isn't _always_ a waste of time.

```bash
ein@~/VulnHub/Kioptrix1:$ nc 10.0.0.35 22
SSH-1.99-OpenSSH_2.9p2
^C
```

Which reveals that the SSH verson is `SSH-1.99-OpenSSH_2.9p2`. We will keep that in mind and move on to HTTP.


### Enumerating HTTP

Browsing to the port 80 via a web browser simply displays the default Apache landing page.

![Browsing to HTTP](/assets/img/VulnHub/Kioptrix1_1.PNG)

Let's fingerprint the port using netcat and try to find out what version of Apache is running: 
```bash
ein@~/VulnHub/Kioptrix1:$ nc 10.0.0.35 80
GET / HTTP/1.0

HTTP/1.1 200 OK
Date: Mon, 27 Jul 2020 07:12:21 GMT
Server: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
Last-Modified: Thu, 06 Sep 2001 03:12:46 GMT
ETag: "8805-b4a-3b96e9ae"
Accept-Ranges: bytes
Content-Length: 2890
Connection: close
Content-Type: text/html
```

Cool, so we can see that `Apache/1.3.20` is running as well as `mod_ssl/2.8.4` and `OpenSSL/0.9.6b`. Let's remember these version numbers for later.

### Enumerating SMB 

We'll skip up to port 139. I'll go ahead and use `smbclient` to see if there are any shares on the machine:

```bash
smbclient -m SMB2 -N -L //10.0.0.35/
Server does not support EXTENDED_SECURITY  but 'client use spnego = yes' and 'client ntlmv2 auth = yes' is set
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        IPC$            IPC       IPC Service (Samba Server)
        ADMIN$          IPC       IPC Service (Samba Server)
Reconnecting with SMB1 for workgroup listing.
Server does not support EXTENDED_SECURITY  but 'client use spnego = yes' and 'client ntlmv2 auth = yes' is set
Anonymous login successful

        Server               Comment
        ---------            -------
        KIOPTRIX             Samba Server

        Workgroup            Master
        ---------            -------
        MYGROUP              KIOPTRIX

```

Okay, looks like the `IPC$` and `ADMIN$` shares are hosted via SMB. Let's see if we can connect to these shares:

```bash
ein@~/VulnHub/Kioptrix1:$ smbclient -m SMB2 -N //10.0.0.35/IPC$
Server does not support EXTENDED_SECURITY  but 'client use spnego = yes' and 'client ntlmv2 auth = yes' is set
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
NT_STATUS_NETWORK_ACCESS_DENIED listing \*
smb: \> exit
ein@~/VulnHub/Kioptrix1:$ smbclient -m SMB2 -N //10.0.0.35/ADMIN$
Server does not support EXTENDED_SECURITY  but 'client use spnego = yes' and 'client ntlmv2 auth = yes' is set
Anonymous login successful
tree connect failed: NT_STATUS_WRONG_PASSWORD
```

No luck, let's grab the SMB version before moving on. I'll quickly use the `auxiliary/scanner/smb/smb_version` Metasploit module to perform this:

```bash
msf5 auxiliary(scanner/smb/smb_version) > run

[*] 10.0.0.35:139         - Host could not be identified: Unix (Samba 2.2.1a)
[*] 10.0.0.35:445         - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
msf5 auxiliary(scanner/smb/smb_version) > 
```

### Enumerating HTTPS

Looking at port 443 with a web browser reveals the same default landing page.

![Browsing to HTTP](/assets/img/VulnHub/Kioptrix1_2.PNG)

At this point, it feels like a good time to go ahead and try to directory brute-force the web server.
Let's kick this off on both port 80 and 443 while we hit some other services. We'll use `ffuf` for this:

#### HTTPS:

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  -u https://10.0.0.35/FUZZ
```

#### HTTP:

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  -u http://10.0.0.35/FUZZ
```

Let's take a look at the results:
```bash

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.1.0-git
________________________________________________

 :: Method           : GET
 :: URL              : http://10.0.0.35/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
________________________________________________

#                       [Status: 200, Size: 2890, Words: 453, Lines: 87]
#                       [Status: 200, Size: 2890, Words: 453, Lines: 87]
# Suite 300, San Francisco, California, 94105, USA. [Status: 200, Size: 2890, Words: 453, Lines: 87]
# or send a letter to Creative Commons, 171 Second Street,  [Status: 200, Size: 2890, Words: 453, Lines: 87]
# Priority ordered case sensative list, where entries were found  [Status: 200, Size: 2890, Words: 453, Lines: 87]
#                       [Status: 200, Size: 2890, Words: 453, Lines: 87]
# This work is licensed under the Creative Commons  [Status: 200, Size: 2890, Words: 453, Lines: 87]
# Attribution-Share Alike 3.0 License. To view a copy of this  [Status: 200, Size: 2890, Words: 453, Lines: 87]
# on atleast 2 different hosts [Status: 200, Size: 2890, Words: 453, Lines: 87]
# directory-list-2.3-medium.txt [Status: 200, Size: 2890, Words: 453, Lines: 87]
# license, visit http://creativecommons.org/licenses/by-sa/3.0/  [Status: 200, Size: 2890, Words: 453, Lines: 87]
#                       [Status: 200, Size: 2890, Words: 453, Lines: 87]
# Copyright 2007 James Fisher [Status: 200, Size: 2890, Words: 453, Lines: 87]
                        [Status: 200, Size: 2890, Words: 453, Lines: 87]
manual                  [Status: 301, Size: 294, Words: 19, Lines: 10]
usage                   [Status: 301, Size: 293, Words: 19, Lines: 10]
mrtg                    [Status: 301, Size: 292, Words: 19, Lines: 10]
                        [Status: 200, Size: 2890, Words: 453, Lines: 87]
:: Progress: [220560/220560] :: Job [1/1] :: 1086 req/sec :: Duration: [0:03:23] :: Errors: 0 ::
```

Attempting to access the `/mrtg` sub-directory redirects us to `/mrtg/`:

```bash
nc 10.0.0.35 80
GET /mrtg HTTP/1.0

HTTP/1.1 301 Moved Permanently
Date: Mon, 27 Jul 2020 07:32:55 GMT
Server: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
Location: http://127.0.0.1/mrtg/
Connection: close
Content-Type: text/html; charset=iso-8859-1

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<HTML><HEAD>
<TITLE>301 Moved Permanently</TITLE>
</HEAD><BODY>
<H1>Moved Permanently</H1>
The document has moved <A HREF="http://127.0.0.1/mrtg/">here</A>.<P>
<HR>
<ADDRESS>Apache/1.3.20 Server at 127.0.0.1 Port 80</ADDRESS>
</BODY></HTML>
```

Let's take a look using a web browser:

![Browsing to HTTP](/assets/img/VulnHub/Kioptrix1_3.PNG)

Interesting, it's hosting some network monitoring software called `Multi Router Traffic Grapher (MRTG)`.

Unfortunately, further Googling eventually shows this is a dead end.

# Exploitation

Let's go back to those version numbers we grabbed during the enumeration phase. If we google the Samba version (2.2.1a), we can quickly find an [interesting exploit](https://www.exploit-db.com/exploits/10).

This is a really old exploit, but so is the box.. Let's see if it works. We can start by grabbing it and compiling it using GCC.

```bash
wget https://www.exploit-db.com/download/10
```

Let's remove all the garbage comments from the top, and save it with the `.c` file extension.

```bash
vim 10
```

Now let's compile it using GCC

```bash
gcc exploit.c -o exploit
```

To run the compiled exploit, we'll need to specify the architecture via `-b 0` and then the IP as the positional argument `<Target IP>`.

![Running the exploit](/assets/img/VulnHub/Kioptrix1_4.PNG)

# Privilege Escalation

After running the exploit, we are dropped directly into a root shell. So that's it, rooted!
