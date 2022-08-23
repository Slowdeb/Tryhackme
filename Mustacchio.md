Room: [Mustacchio]() ONGOING TUTORIAL

Difficulty: Hard

Overview: In this room enumeration is very important, searching for subdomains, modify an exploit to take advantage of a chat program has well a MongoDb web interface. Find and crack hashes to compromise a Bolt CMS server till we privesc to root through cap_setuid+ep capabilities.

xxe enumeration privesc web

------------------------------------------------------------------------------------------------------------------------------------------------------------------

We can start our enumeration fase by running "nmap". It will scan for all open ports on the target system:

```
nmap -sV -sC -p- mustacchio.thm -oN nmap.txt

mustacchio.thm → you can add the ip of the room to /etc/hosts

-Pn → To bypass the blocking of ping probes   

-sV → To determine service/version info

-p-  →  Scan all ports

-oN → Save scan to a file
```

![nmap](https://user-images.githubusercontent.com/76821053/186257404-a5196013-3e31-4a6b-9c30-10db5ce6c027.png)

There are three open ports:

22     →  ssh OpenSSH 7.2p2

80     →  http Apache httpd 2.4.18

8765   →  http nginx 1.10.3

Port 80 hosts a http web server:

![website80](https://user-images.githubusercontent.com/76821053/186258496-eed09ad7-fb11-4263-af26-369b744af45f.png)

To search for any hidden directories we can use “Gobuster”:

```
gobuster dir --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt --url http://mustacchio.thm/ -x .txt,.cgi,.php,.log,.bak,.xxx,.old

dir             → for directory brute-force

--wordlist   → Path of the wordlist

--url           →  Specify the url path

-x              → File extensions to search for
```

![gobuster](https://user-images.githubusercontent.com/76821053/186258583-84e51d26-ff8c-446b-86bd-00246df49f6b.png)

We can now start browsing the directories found with “gobuster”.

There is a robots.txt file but it is empty:

![robotstxt](https://user-images.githubusercontent.com/76821053/186258725-8fc63b98-3754-43af-a627-7d30f4f90a5e.png)

At /custom/js path there is an interesting file called users.bak:

![usersbak](https://user-images.githubusercontent.com/76821053/186259309-68a79b6f-4a2f-449b-9468-a1d746807f71.png)

After downloading the file we can see that it holds some credentials but we need to use strings to be able to read it:

![strings](https://user-images.githubusercontent.com/76821053/186259376-a34e8467-5b75-46e9-898d-098dbda16fe4.png)

This credentials are from a sql database, they are encrypted. Braking down the credentials we can try to assume that “admin” is the username and the rest of the hash is the encrypted password.

like so:

admin:1868e36a6d2b17d4c2745f1659433a54d4bc5f4b

So using “hash-identifier” we can identify the encryption type:

![hashidentifier](https://user-images.githubusercontent.com/76821053/186259480-5f22283a-efc4-4950-b07a-11f5c8dbf023.png)

Knowing the encryption type we can use “John the ripper” to crack this password:

First we put the hash inside a file and pass it to john.




