room: [tomghost](https://tryhackme.com/room/tomghost)

Difficulty: Easy

Overview: 

-------------------------------------------------------------------------------------------------------------------------------------------------------------------

First we will use nmap to scan for all open ports on the remote host.

nmap -sV -sC -A -p- 10.10.191.121 -oN nmap.txt

![nmap](https://user-images.githubusercontent.com/76821053/119219288-9f1d4880-badc-11eb-851e-ec369e94bee5.png)

There are four ports open:

22     →  OpenSSH 7.2p2

53     →  tcpwrapped

8009   →  ajp13 Apache Jserv

8080   →  Apache Tomcat 9.0.30

By using gobuster we might find hidden directories:

![gobuster](https://user-images.githubusercontent.com/76821053/119219461-a2650400-badd-11eb-8da7-759dec1ebee7.png)

This directory was interesting but took me nowhere.

![httpexamples](https://user-images.githubusercontent.com/76821053/119219489-c7597700-badd-11eb-8ac1-33a2182c1aae.png)

![httpmanager](https://user-images.githubusercontent.com/76821053/119219478-bd377880-badd-11eb-9811-56fbbf4da0ed.png)

Since i could not login to the Apache server. I google searched and found a vulnerability in tomcat Apache version 9.0.30.



There is a LFI vulnerability named Ghostcat for Apache with the CVE-2020-1938. It is read/inclusion vulnerability in the AJP connector in Apache Tomcat. Which means we could read webapp configuration files or even upload and execute code to the target host by exploiting file inclusion through Ghostcat vulnerability.

Two good reads and proof of concept:
 
https://www.tenable.com/blog/cve-2020-1938-ghostcat-apache-tomcat-ajp-file-readinclusion-vulnerability-cnvd-2020-10487

https://github.com/vulhub/vulhub/tree/master/tomcat/CVE-2020-1938
 
After downloading and executing the python script for this vulnerability i found some ssh credentials for the target system in the webapp configuration file:

![CVE-tomghost](https://user-images.githubusercontent.com/76821053/119219579-2a4b0e00-bade-11eb-9d3d-4c85eac5502c.png)

This credentials lets us login to the machine through ssh:




User.txt found:



After some enumeration of the target system, using tools such has linpeas.sh i found two interesting files in skyfuck home directory:



These are encrypted credentials, and there might be a way to decrypt this credentials with “gpg”.

Let's see if we can decrypt them by first importing the tryhackme.asc and them decrypting the credentials.pgp:



When trying to decrypt credential.pgp it asks for a passphrase. 



Now, tryhackme.asc stores a private pgp key inside:



In order to crack this private key we need to download the files to our system system to crack them with john the ripper.

To download the files i used “scp”:

scp skyfuck@10.10.191.121:/home/skyfuck/* . 



To brute-force this key, first we will use gpg2john to convert the key into a hash:





Now just crack it with john the ripper:



alexandru        (tryhackme)

We found our passphrase, now we can decrypt the credential.pgp inside the target machine:



We have access to ther user “merlin”:



“merlin” has SUID permissions to run zip. With a quick search at GTFOBins we can find our way to escalate privileges to root.



https://gtfobins.github.io/gtfobins/zip/



We just need to tweek the commands a little bit.

TF=$(mktemp -u)
sudo /usr/bin/zip $TF /etc/hosts -T -TT 'sh #'     → we need to use sudo and specify the location of the zip binary.



Has we can see above we have root privileges! 



Success , found the last flag!

 

