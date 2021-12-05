Room: [IDE](https://tryhackme.com/room/ide)

Difficulty: Easy

Overview: In this challenge room we will be focusing in enumeration. Exploiting a Codiad IDE framework server and gain an initial shell. Escalate horizontaly through “credential stuffing” to finaly take advantage of users sudo permissions to privesc vertically to root.

------------------------------------------------------------------------------------------------------------------------------------------------------------------

For enumerating all open ports on the remote system we can use “nmap”:

```
nmap -sV -sC -p- ide.thm -oN nmap.txt

-sV         → Probe open ports to determine service
-sC         → Scan using the default set of scripts
-p-         → Scan all ports
ide.thm     → ip address of the room
-oN         → Save the ouput of the scan in a file
```

![image](https://user-images.githubusercontent.com/76821053/144726736-0910d18b-4501-4991-8f61-016778387937.png)

There are four open ports in the system:

21       →  vsftpd 3.0.3

22       →  OpenSSH 7.6p1 Ubuntu

80       →  Apache httpd 2.4.29

62337 → Apache httpd 2.4.29 (Codiad 2.8.4)

Normally when manually enumerating the open ports i start by the http web servers.

At port 80 there is a Apache server, but it doesn't have anything of interest.

![image](https://user-images.githubusercontent.com/76821053/144726746-67f83f01-12d3-4d38-9581-305a3b8d157e.png)

At port 62337 there is a Apache server hosting a Codiad IDE framework login portal. 

![image](https://user-images.githubusercontent.com/76821053/144726750-18d3c7f5-8f20-4612-b996-da8207234929.png)

Since we do not have any credentials we can run “Gobuster” which can help us search for hidden directories or files in http servers:

```
gobuster dir --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt --url http://ide.thm:62337 -x .txt,.cgi,.php,.log,.bak,.xxx,.old

dir              →  Uses directory/file enumeration mode
--wordlist   → Path to wordlist 
--url            → specifies the path of the target url we want to find any hidden directories
-x               → Search for all files with the specified extentions 

```

![image](https://user-images.githubusercontent.com/76821053/144726760-6f91879f-74f3-4645-b9e1-0e301ea56ad9.png)

Looking at the “Nmap” results we can see that the ftp service allows “Anonymous” login. So we can start by exploring it:

```
ftp ide.thm
```

![image](https://user-images.githubusercontent.com/76821053/144726768-bed2d199-cb37-4987-9654-73c22fb41da9.png)

We have to pay close attention to the directory listing. At first glance it seemed to be empty, but there was a folder named “...” (pretty sneaky):

![image](https://user-images.githubusercontent.com/76821053/144726781-261260d9-fb7c-4eab-8a82-81654327d0cf.png)

This directory had one file inside. After dowloading the file, to be able to read it we need to use :

```
cat ./-
```

![image](https://user-images.githubusercontent.com/76821053/144726787-142b77b6-9e19-4ce0-8105-b5594015177f.png)

We have found a clue. Maybe it belongs to the login portal that we have found earlier.

We could try to brute force the login portal with Burp Suite or Hydra, but since user “john” has a default password we can try and guess it by hand. Which is pretty easy to guess:

![image](https://user-images.githubusercontent.com/76821053/144726795-489667b0-6a81-4ff6-b7f2-500adbe3afa2.png)

After login in, the server hosts many different projects. 

![image](https://user-images.githubusercontent.com/76821053/144726800-842adbfe-0f60-473f-9f57-e265eec75531.png)

To get a foothold in the target system we need to create a new project and upload a php reverse shell to it: 

![image](https://user-images.githubusercontent.com/76821053/144726805-ab21180d-d69b-4a5e-a05a-21c222cce3b1.png)

In this case i uploaded pentestmonkey php reverse shell:

![image](https://user-images.githubusercontent.com/76821053/144726814-f04597aa-5af2-465e-9b5f-deef9037f4c7.png)

When uploading the reverse shell do not forget to change this fields to match your ip and chosen port.

Now how to we trigger the payload? 

"Gobuster" found many hidden directories and one of them hinted us to a workspace directory:

![image](https://user-images.githubusercontent.com/76821053/144726819-f8ef6210-3bf6-4a98-aac1-4684fd243ce9.png)

This directory will host all the new projects that we create. Since i created a new project named “shell” it will appear here:

![image](https://user-images.githubusercontent.com/76821053/144726825-8152190f-7758-4000-b2d0-6b9d19d27d52.png)

Now, just start a netcat listener on your chosen port and hit the php reverse shell:

![image](https://user-images.githubusercontent.com/76821053/144726827-e1f2516b-7347-4559-80a3-e2e57bb34bd6.png)

Success we got a reverse shell back:

![image](https://user-images.githubusercontent.com/76821053/144726828-3b77a321-e140-4f9b-a214-4690cd96a054.png)

Since we are a low privilege user “www-data” we need to further enumerate this machine to see if we can privesc.

Searching the /home directory we can see that there is a user called “drac”. And when listing all the contents of his directory we do not have permissions to read the user.txt flag. 

But we can read .bash_history, which stores a history of user commands entered at the command prompt:

![image](https://user-images.githubusercontent.com/76821053/144726836-1fd31136-23e8-4abd-838b-c4e633c2a13e.png)

We have found “drac” credentials for a mysql database, but there is not a mysql database to connect to.

What can we do with these credentials? Password reuse is a big vulnerability that affects many users, so an attacker can take advantage of a technique known as "credential stuffing".

In our case “drac” uses the same password to connect to ssh:

![image](https://user-images.githubusercontent.com/76821053/144726848-c089f583-63ad-412f-8e3a-9d21c2e6d1c0.png)

With “drac” we can read the user.txt flag: 

![image](https://user-images.githubusercontent.com/76821053/144726858-5031fb5c-c63e-4c6c-a638-1090e25b4f82.png)

If we look at the sudo permissions for this user we can see that he can run /usr/sbin/service vsftpd restart with full privileges:

![image](https://user-images.githubusercontent.com/76821053/144726862-48b09620-07cc-4105-810d-e833fe5bc413.png)

When searching for the vsftpd service we can see that we have write privileges. 

```
find / -type f -name vsftpd.service 2>/dev/null

```

![image](https://user-images.githubusercontent.com/76821053/144726864-387bf518-410e-4b27-ae83-8249b95a7c51.png)

We have found our privesc, and to to so we need to edit the contents of the file. 

If we change the ExecStart field we can tell the service to create a copy of the /bin/sh binary to “drac” home directory and set Setuid bit to it:

```
/bin/bash -c 'cp /bin/sh /home/drac/sh | chmod u+s /home/drac/sh'
```

![image](https://user-images.githubusercontent.com/76821053/144726870-f430959c-14a3-44fc-8c84-8f630637428a.png)

To trigger this privilege escalation we just need to run:

```
sudo /usr/sbin/service vstpd restart
```

The vsftpd service will restart and run our commands:

![image](https://user-images.githubusercontent.com/76821053/144726876-4260d954-8670-46dd-b69f-30dd93a9d3be.png)

Success, if worked! Just run the “sh” binary with -p for persistence and we will get to “root”.

```
./sh -p
```

![image](https://user-images.githubusercontent.com/76821053/144726881-7fe9d2e3-9afd-4f1e-b58f-4d58d7a873bf.png)

Final flag is at "root's" home directory!































