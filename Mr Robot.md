Room: [Mr Robot](https://tryhackme.com/room/mrrobot)

Difficulty: Medium

Overview: In this room we will use different techniques/tools to enumerate the target machine since there is sensitive information exposed. After exploiting a webpage admin panel we will use SUID binaries to do privilege escalation and gain full control of the machine.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

To enumerate the machine we will start with "nmap" to check for open port on the system:

```
nmap -sV -sC -A -p- 10.10.232.56 -oN nmap.txt

-sV    →  Probe open ports to determine service
-sC    →  Scan using the default set of scripts
-p-    →  Scan all ports
-oN    →  Save the ouput of the scan in a file
```

![nmap](https://user-images.githubusercontent.com/76821053/185758960-07557961-4579-40ec-91eb-2f90a6352289.png)

There are two open ports on the target system:

22 → ssh (aparently closed)

80 → Apache httpd

443 → Apache httpd

Since there are two http servers, one on port 80 and the other on port 443. We can use a tool called "Gobuster" to find more information about any hidden directories or files in port 80:

```
gobuster dir --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt  --url http://10.10.232.56:80/ -x .txt,.cgi,.php,.log

dir             → Uses directory/file enumeration mode
--wordlist      → Path to wordslist 
--url           → specifies the path of the target url we want to find any hidden directories
-x              → Search for all files with the specified extentions 
```

![gobuster](https://user-images.githubusercontent.com/76821053/185759356-5426e858-a36f-4de9-80d4-3239fdbe3737.png)

After searching through some directories, we can see that there is a robots.txt file. A robots.txt file contains instructions for bots/crawlers on which pages they can and cannot access.

Inside there's two file directories:

![robotshttp](https://user-images.githubusercontent.com/76821053/185761286-1961b69d-3fea-4594-9ea9-313ec5ef5aef.png)

Found first flag:

![flag1](https://user-images.githubusercontent.com/76821053/185761507-25c3ab56-b750-4164-ab33-9276605ca8df.png)

I continued searching for more information and found a teaser on the license.txt file directory:

![licencetxt](https://user-images.githubusercontent.com/76821053/185761854-87496a4b-3947-49f0-8bae-4a7d38db47e1.png)

If we scroll down the page there is some valuable information:

<img width="1252" alt="Screenshot 2022-08-20 at 19 49 06" src="https://user-images.githubusercontent.com/76821053/185762146-e12f24da-e99b-4d8d-be52-239d65b30c9a.png">

At the bottom there is a hash and by the looks of it is encoded in base64.

![decodehash](https://user-images.githubusercontent.com/76821053/185762231-7686c7d6-b8fb-4337-b674-747ca521271f.png)

After decoding the hash i found credentials for a user named "Elliot". With these credentials we can access the website admin login page:

![wpdashboard](https://user-images.githubusercontent.com/76821053/185762692-9b53da6b-2af4-4936-b93d-b0b0dc7e36b9.png)

Now,the Appearance page lets us edit themes so this is our way to get a foothold on the target system with a reverse shell. For that we can use pentestmonkey php-reverse-shell.

To do so we will need to edit the page template.php or archives.php files and overwrite the contents of it with pentestmonkey reverse shell. When running the preview of the theme "Twenty Thirteen" we got ourselves a reverse shell.

![reverseshell](https://user-images.githubusercontent.com/76821053/185798374-fd0657da-d16f-49c7-bf0d-45625933d674.png)

Sometimes to trigger the exploit we need to go to the link manually, example:

![filephp](https://user-images.githubusercontent.com/76821053/185798206-ee8141f9-c958-4059-a348-53aacadb7115.png)

We need to start a netcat listener and ran the preview.
 
![nclistenner](https://user-images.githubusercontent.com/76821053/185799814-01ab194d-113d-40fc-87f6-73785179fc84.png)

I searched around and found key-2-of-3.txt but i don't have any permissions to open it.

![searchkey2](https://user-images.githubusercontent.com/76821053/185799969-2f6c1e13-88a3-4c34-964b-0197cfffa27e.png)

In the same directory there is a password.raw-md5 file with presumably credentials but encrypted with md5. We can use "john the ripper" to decrypt the file. If you remember earlier there is another file called "fsociety.dic" file in the robots.txt webpage. Since that file is a dictionary we can try to brute force the hash with it.

![john](https://user-images.githubusercontent.com/76821053/185799991-7d97c9c6-401b-4be2-aacc-96de65a0f33d.png)

Another way you can decrypt the password is by using the website “ https://hashes.com/en/decrypt/hash ”.

![hashsite](https://user-images.githubusercontent.com/76821053/185800020-6092c907-6b53-4b24-9120-d7adfb2b5eb2.png)

With the hash decrypted i can now try and access user "robot" on the target system but i need the shell to be interactive:

We can use this python string in order to get an interactive shell:

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

![pythonstring](https://user-images.githubusercontent.com/76821053/185800565-df9dceb3-574f-4ca9-9206-d9e49b6eb186.png)

Now i am able to perform some commands like "su" and use it to access "robot" account:

![surobot](https://user-images.githubusercontent.com/76821053/185800608-545f27d5-975a-4435-ba99-3d38aaa249f3.png)

Once we are user "robot" we can now read the second flag!

![key2](https://user-images.githubusercontent.com/76821053/185800698-6592f8b3-df49-4fd8-b161-2c2bd06729f7.png)

It is now time to start enumerating this linux machine, we could use "linpeas" but in this case we just need to search for SUID permissions on the machine: 

![find-perm](https://user-images.githubusercontent.com/76821053/185801007-ac0180cc-d840-4e83-9e4e-dda68a0cf4b0.png)

I have discovered that the system is running nmap. Knowing that it has SUID permissions, means that it allows users to execute a file as the file owner . For that, there is an awesome website called [GTFOBins](https://gtfobins.github.io) that has list of Unix binaries that can be used to bypass local security restrictions in misconfigured systems:

![GTFOBins](https://user-images.githubusercontent.com/76821053/185801037-064898df-e7f9-487b-80ea-61b12a6a1270.png)

Option b works like a charm:

![privesc](https://user-images.githubusercontent.com/76821053/185801071-1efd22f9-a5ac-4505-acdb-b891ec3da39c.png)

Success, now we have "root" privileges.

Now you just need to find where is the 3rd and final flag.

![finalflag](https://user-images.githubusercontent.com/76821053/185801261-547bc5c6-7944-4050-b123-2cbe8cb60f66.png)

 


