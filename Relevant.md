## Relevant
---
THM Relevant Room Walkthrough: https://tryhackme.com/room/relevant

## Summary
---

## Walkthrough
---
Let's first begin by enumerating the machine with nmap. We will run nmap with

```bash
nmap -sS -sV -sC -Pn  <TARGET_IP>
```

![[attachments/Screenshot 2025-02-01 160003.png]]

We can see we have 5 open ports. It's good to keep in mind we only scanned the default number of ports with nmap, which by default should be the 1000 most popular ports.

The 3 most interesting ports we have here are 80, 445, and 3389. (Webserver, SMB shares, RDP access)

Let's start by looking into these. The webserver seems to host this page: 

![[attachments/Screenshot 2025-02-01 160429.png]]

The first thing I think of when I see this and port 445 open is a possible EternalBlue exploit depending on the version of Windows Server.

The only other thing on this website is a link to https://www.iis.net/?utm_medium=iis-deployment with no other interesting information. There may also be something hidden in the image, but let's not get too deep.

Let's check the SMB port, 445. We will use smbclient with the command

```bash
smbclient -L //10.10.71.74
```

We see we have some shares available. We have 3 hidden shares indicated by the $ and then a regular share. 

![[attachments/Screenshot 2025-02-01 161102.png]]

Let's try to connect to them anonymously without passing a -U (User) param.

![[attachments/Screenshot 2025-02-01 161728.png]]

I was able to connect to the IPC share and the last share. I would not expect to find anything in the IPC share because it's used for Inter-Process Communication and not file storage. What's interesting is we were able to see that the last share nt4wrksv actually has a file in it. Let's download it and check it out.

![[attachments/Screenshot 2025-02-01 162800.png]]

We see that we have some encoded passwords. It looked like Base64 was used to encode them so let's decode them to see if we have anything here.

![[attachments/Screenshot 2025-02-01 163104.png]]

We get a username and some passwords! 

```bash
Bill - Juw4nnaM4n420696969!$$$
Bob - !P@$$W0rD!123 
```

The only place I can think to use them is RDP since that port is also open. Let's try 

