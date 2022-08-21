Room: [Mr Robot](https://tryhackme.com/room/mrrobot)

Difficulty: Medium

Overview: In this room we will take advantage of different services on a windows machine, abusing Kerberos pre-authentication to enumerate users, dumping and cracking hashes with the help of impacket tools and John the ripper to finaly privesc exploiting SeBackupPrivilege permissions.

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

I found a hash and by the looks of it is encoded in base64.

![decodehash](https://user-images.githubusercontent.com/76821053/185762231-7686c7d6-b8fb-4337-b674-747ca521271f.png)

After decoding the hash i found credentials for a user named "Elliot". With these credentials we can access the website admin login page:

![wpdashboard](https://user-images.githubusercontent.com/76821053/185762692-9b53da6b-2af4-4936-b93d-b0b0dc7e36b9.png)

Now,the Appearance page lets me edit themes so this is our way to get a foothold on the target system with a reverse shell. So i will use pentestmonkey php-reverse-shell.

Editing the page template.php or archives.php when the preview of the theme Twenty Thirteen runs we got ourself a reverse shell.

Sometimes you need to go to the link manually, example:


