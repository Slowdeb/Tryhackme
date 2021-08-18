Room: [Super-Spam](https://tryhackme.com/room/superspamr)

Difficulty: Medium

Overview: Good reconaissance and enumeration of the machine is the key, gather information about users, access misconfigured FTP, crack a .cap wifi file which will lead us to a reverse shell. Go through encryption files to achieve a horizontal privesc and eventualy take advantage of VNC service to reach root.   

--------------------------------------------------------------------------------------------------------------------------------------------------------------------

Let's start our enumeration fase by running “nmap” to check for all open ports on the target system:

```
nmap -sV -sC -p- superspam.thm -oN nmap.txt

-sV    →  Probe open ports to determine service
-sC    →  Scan using the default set of scripts
-p-    →  Scan all ports
-oN    →  Save the ouput of the scan in a file
```

![image](https://user-images.githubusercontent.com/76821053/129763941-c9629b1a-db8e-438f-8195-f6c0ed9b5066.png)

There are five open ports on the target system:

80   →  HTTP Apache 2.4.29

4012 →  OpenSSH 7.6p1

4019 →  FTP vsftpd 3.0.3

5901 →  VNC (protocol 3.8)

6001 →  X11 (unknown service)

When visiting the webserver at port 80 i found a website built with Concrete5 CMS:

![image](https://user-images.githubusercontent.com/76821053/129764014-361c5e05-421b-4fe4-98c2-47194f27a5f4.png)

To check the version of the CMS website we can use a Firefox browser extension called Wappalyzer that uncovers the technologies used on websites, like in this case content management systems:

![image](https://user-images.githubusercontent.com/76821053/129764132-05c7355e-92f4-45cd-aa7b-29d8cc49b72d.png)

Another way the get the version of CMS is by checking the source code of the page:

![image](https://user-images.githubusercontent.com/76821053/129764157-c23c8d51-48d1-442c-a871-0bf42b367514.png)

Further enumerating the website there are usernames inside each blog post:

![image](https://user-images.githubusercontent.com/76821053/129764242-e6aeea38-7516-455b-b414-7552d4287ac0.png)

For example:

![image](https://user-images.githubusercontent.com/76821053/129764270-bd388ed8-241e-4f42-a6b3-7954595e4a3a.png)

I search the posts for all the usernames that i could find and created a list:

```
Adam_Admin
Donald_Dump
Lucy_Loser
Benjamin_Blogger
``` 

There is a concrete5 login portal and potentialy one of these users might grants us access to the server.

![image](https://user-images.githubusercontent.com/76821053/129764377-70121f36-e603-41d6-a0b6-11218addb0cb.png)

We could try and brute force the login page, but let's leave aside the webserver for a bit to explore the vsftp service, since nmap script found out that “Anonymous” login was allowed:

```
ftp target_ip port
```

![image](https://user-images.githubusercontent.com/76821053/129764411-3b1c55c1-4809-4bf7-9dfd-fe16698be762.png)

It is important that we list the contents of the directories with “ls -la” to show us all the hidden files/directories in the server. 

Downloaded three interesting files:

note.txt:

![image](https://user-images.githubusercontent.com/76821053/129764449-76129a62-4b42-40e7-9439-76dea9fe0da2.png)

This note had a hint sugesting that we would find something in the wireshark pcap files. I downloaded the pcap files from IDS_logs directory and explored it a bit and found a SMB hash that could possible be cracked. But this i looked aside since i found a new file that had the unswer i needed.

![image](https://user-images.githubusercontent.com/76821053/129764553-fc38e0e2-bb0d-4cb1-9a4e-9f725d7c7247.png)

After accessing the .cap directory i found another clue:

.quicknote.txt:

![image](https://user-images.githubusercontent.com/76821053/129764634-0dcf1371-b716-48a3-9a4f-c58bf994ae28.png)

In the same directory was SamsNetwork.cap file. After checking its contents with wireshark it seem that it is a capture attack data from a wifi attack:

![image](https://user-images.githubusercontent.com/76821053/129764667-ccb2ebea-3750-498e-b885-82784576a850.png)

Le's send the SamsNetwork.cap file to aircrack.ng to crack it:

```
aircrack-ng SamsNetwork.cap -w /usr/share/wordlists/rockyou.txt

-w → wordlist path
```

![image](https://user-images.githubusercontent.com/76821053/129764706-58acdaed-05b4-464f-93b3-94a99902fcb4.png)

After a bit the .cap file was cracked and we have a password!

Thinking like the hacker that attacked this machine, maybe we can use this wifi cracked password and see if we can use it elsewhere.

We can start by checking if any of the users we found earlier can login with this password:

I used hydra to brute force the concrete5 login portal even though we could do it manually since we only found four users on the webserver:

```
hydra -L users.txt -p xxxxxxxx superspam.thm http-post-form "/concrete5/index.php/login/authenticate/concrete:uName=^USER^&uPassword=^PASS^&ccm_token=1629024922%3A3e10e526c4bd44ec5ebeb20ca546d598:F=Invalid username or password" -V
``` 

![image](https://user-images.githubusercontent.com/76821053/129764754-11002e6a-c0ae-49a3-9271-f8308e99c4b0.png)

This password belongs to user “Donald_Dump” and we can successfully login and head to the dashboard to see if there is some way to get a reverse shell connection:

![image](https://user-images.githubusercontent.com/76821053/129764789-effe4d65-fd3f-457f-b3f7-b9fc9d234fd8.png)

When searching around there is an option inside the dashboard “system & settings” tab that allows us to edit file permissions:

![image](https://user-images.githubusercontent.com/76821053/129764839-dda34657-c1ad-4ccd-8cc6-4b51fd750c1a.png)

We just have to add php extention and i am going to use the pentestmonkey php reverse shell:

![image](https://user-images.githubusercontent.com/76821053/129764889-d815bb48-a91f-418d-b181-7e97042ee886.png)

Edit pentestmonkey php-reverse-shell and change these fields to match your ip and chosen port:

![image](https://user-images.githubusercontent.com/76821053/129764931-5a53d322-16e9-4bcb-b7cd-d75d73383067.png)

Now we just have to upload the payload to the target system and hopefully receive a connection back in our netcat listener:

![image](https://user-images.githubusercontent.com/76821053/129764952-84aa1c71-df7e-4967-904f-e8db50688a9c.png)

Setup the netcat listener on your chosen port and to trigger the payload just press the URL to File link:

![image](https://user-images.githubusercontent.com/76821053/129764978-c58b3b8f-7ee0-4175-8e2c-c740d744f27f.png)

Success we are in! The user flag was in “/home/personal/Work” directory:

![image](https://user-images.githubusercontent.com/76821053/129765003-9e8bf0de-64ba-4551-8209-7a6824559b18.png)

There are some clues and messages inside the users home directories:

super-spam home directory:

![image](https://user-images.githubusercontent.com/76821053/129765028-cb4b8848-9a90-4c89-a994-84cadac598de.png)

personal home directory:

![image](https://user-images.githubusercontent.com/76821053/129765046-b1d80bf4-e774-483f-a4db-744d53a3d3b7.png)

lucy_loser home directory:

![image](https://user-images.githubusercontent.com/76821053/129765084-3404b95c-b648-4b0a-9fc2-3276ae1aae7e.png)

Inside lucy_loser home directory there was a set of picture files and a python script:

![image](https://user-images.githubusercontent.com/76821053/129765128-62bbaf46-ce2d-48b7-af89-38047e6f824c.png)

To investigate these files i had to transfer them into my system. The target machine runs python so we can start an http server in it and download the files:

![image](https://user-images.githubusercontent.com/76821053/129765399-9648c459-069b-4647-915a-03fc44d1c189.png)

After downloading all files a password was found inside d.png file. 

![image](https://user-images.githubusercontent.com/76821053/129765446-7e8fbee0-554b-45b6-8889-ff7511fe0715.png)

Another way to reach this password is by using the xored.py python script that was in the same directory.

Essentialy this script meshes the two images and obfuscates the text. When we combined the "right" .png files it will give us a clearer picture of the password.

For that i had to run the python script and combine pictures c2.png and c8.png has suggested in lucy_loser note.txt hint: 

![image](https://user-images.githubusercontent.com/76821053/129765474-788c4c0d-48e2-4375-b31d-b7353c1e3ac5.png)

Here is the result:

![image](https://user-images.githubusercontent.com/76821053/129765503-f08c7320-db15-4a29-a66f-dd1b147bb396.png)

Now we have a new password and for what or whom?

This password belongs to user donalddump. When i tried to privesc horizontally to the other users in the machine i was successful with him:  

![image](https://user-images.githubusercontent.com/76821053/129765555-ade309f3-d5d5-4300-b934-40ccd6cc6e42.png)

Since we have good credentials we can try to ssh into the machine with this user:

![image](https://user-images.githubusercontent.com/76821053/129765588-5da5c8db-b41a-4929-b954-5ca542eb0c86.png)

I tried to access donalddump home directory but got a permission denied error:

![image](https://user-images.githubusercontent.com/76821053/129765625-5a0c5ccc-995b-4429-9ea8-19dbaf2026dd.png)

When i checked the folder permissions i saw that i needed to set a new one for it:

![image](https://user-images.githubusercontent.com/76821053/129765837-4e21749f-42fd-478c-80f2-0c19af15c4e7.png)

So we can do that just by:

```
chmod +x 777 donalddump
```

![image](https://user-images.githubusercontent.com/76821053/129765864-17a7f9e2-6a81-4de5-87c2-1a533665122b.png)

The only interesting file in this directory is passwd.  But it has somekind of encryption:

![image](https://user-images.githubusercontent.com/76821053/129765897-1c4903f9-2a2c-4acd-9c9b-93da5c6aeba8.png)

To analyse it better i transfered the file to my kali machine using scp:

```
scp -P 4012 donalddump@superspam.thm:/home/donalddump/passwd ~/Tryhackme/superspam/passwd                  1 ⨯
```

![image](https://user-images.githubusercontent.com/76821053/129765932-1ba60d5f-284e-4d13-8303-8779493256c9.png)

After searching online for a way to crack this encryption i reviewed my notes and saw that the target machine is also running VNC.

And just by doing a quick google search i found that VNC password files are normally stored at:

![image](https://user-images.githubusercontent.com/76821053/129765947-ba6d2dcd-e8de-4532-bdde-33bd58c54bb9.png)

In our case we have the passwd file in the home directory. Let's try to loggin through vncviewer:

```
vncviewer -passwd passwd superspam.thm::5901

-passwd → specify the password file
```

After connecting we got this shell:

![image](https://user-images.githubusercontent.com/76821053/129766216-b9c86b2a-de48-408a-a9ea-6c61008bef16.png)

Let's change this shell into a ssh session since this one is kinda hard to work with and when i listed the contents of /root directory i can see that .ssh is already there and even with the authorized_keys file.

![image](https://user-images.githubusercontent.com/76821053/129766246-2c5de207-a560-4392-a9f4-bb3400348aee.png)

To create a ssh key we have to use, i left my passphrase empty:

```
ssh-keygen
```

![image](https://user-images.githubusercontent.com/76821053/129766280-7bf51445-d058-4709-a76d-84f7a209505e.png)

After generating a password we need to copy the contents of id_rsa.pub into authorized_keys files: 

![image](https://user-images.githubusercontent.com/76821053/129766317-ec6e593b-b6ac-44f1-8f8e-d8876330f47f.png)

To make a copy of id_rsa to our machine i had to start a python3 http server on the target machine:

![image](https://user-images.githubusercontent.com/76821053/129766480-9c3ed93d-41cd-48d9-8d35-6d3ff0d57e04.png)

Now we just need to download the id_rsa file and give proper permissions:

![image](https://user-images.githubusercontent.com/76821053/129766527-2e7ff1d5-47d7-483d-b8d0-f3515dd42643.png)

And voila, we can now connect as root through ssh:

```
ssh -i id_rsa root@superspam.thm -p 4012

-i → select the id_rsa key file
-p → specify port
```

![image](https://user-images.githubusercontent.com/76821053/129766584-dd4a5a33-3c08-4c76-b036-dc815678b9a7.png)

The final flag was inside .nothing folder inside roots home directory:

![image](https://user-images.githubusercontent.com/76821053/129766620-eb733ade-1cc6-45f4-be9b-62e9da1906a5.png)

Still a final challenge. The flag was encoded in base32:

![image](https://user-images.githubusercontent.com/76821053/129766647-2bf7a4eb-5800-4048-834a-67c23d6e1daf.png)

If you cannot recognize the encoding there are some websites that can do that for you, like [dcode.fr](https://www.dcode.fr/cipher-identifier).

To decode and retrive the final flag:

```
echo 'MZWGCZ33NF2GKZKLMRRHKPJ5NBVEWNWCU5MXKVLVG4WTMTS7PU======' | base32 -d
```

![image](https://user-images.githubusercontent.com/76821053/129766688-473278e6-9d00-4269-89eb-94d8b2593aad.png)

Final teaser!!

![image](https://user-images.githubusercontent.com/76821053/129766715-7e73d062-44c1-4881-8084-f0c716632ddb.png)







