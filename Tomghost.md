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

<img width="1343" alt="exploit-db_ghostcat" src="https://user-images.githubusercontent.com/76821053/119219986-05579a80-bae0-11eb-95b9-ca69ba9d0a46.png">


There is a LFI vulnerability named Ghostcat for Apache with the CVE-2020-1938. It is read/inclusion vulnerability in the AJP connector in Apache Tomcat. Which means we could read webapp configuration files or even upload and execute code to the target host by exploiting file inclusion through Ghostcat vulnerability.

Two good reads and proof of concept:
 
https://www.tenable.com/blog/cve-2020-1938-ghostcat-apache-tomcat-ajp-file-readinclusion-vulnerability-cnvd-2020-10487

https://github.com/vulhub/vulhub/tree/master/tomcat/CVE-2020-1938
 
After downloading and executing the python script for this vulnerability i found some ssh credentials for the target system in the webapp configuration file:

![CVE-tomghost](https://user-images.githubusercontent.com/76821053/119219579-2a4b0e00-bade-11eb-9d3d-4c85eac5502c.png)

This credentials lets us login to the machine through ssh:

![sshskyfuck](https://user-images.githubusercontent.com/76821053/119220050-6aab8b80-bae0-11eb-9005-328ea8acc296.png)

User.txt found:

![usertxt](https://user-images.githubusercontent.com/76821053/119220176-2076da00-bae1-11eb-9ed9-015c2e55ff53.png)

After some enumeration of the target system, using tools such has linpeas.sh i found two interesting files in skyfuck home directory:

![credentials_folder](https://user-images.githubusercontent.com/76821053/119220825-4651ae00-bae4-11eb-8456-dd52880ef419.png)

These are encrypted credentials, and there might be a way to decrypt this credentials with “gpg”.

Let's see if we can decrypt them by first importing the tryhackme.asc and them decrypting the credentials.pgp:

![import_tryhackme](https://user-images.githubusercontent.com/76821053/119220857-6f723e80-bae4-11eb-8f55-01ff91a2ffe0.png)

![decrypt_no_pass](https://user-images.githubusercontent.com/76821053/119220868-78631000-bae4-11eb-8cfc-05ab9a5f2f55.png)

When trying to decrypt credential.pgp it asks for a passphrase. 

Now, tryhackme.asc stores a private pgp key inside:

![cat_tryhackme](https://user-images.githubusercontent.com/76821053/119220952-dee82e00-bae4-11eb-899f-72360b58d9c5.png)

In order to crack this private key we need to download the files to our system system to crack them with john the ripper.

To download the files from the target to my kali machine i used a tool called “scp”:

scp skyfuck@10.10.191.121:/home/skyfuck/* . 

![scpupload](https://user-images.githubusercontent.com/76821053/119221028-35ee0300-bae5-11eb-9043-70b17b255e72.png)

To brute-force this key, first we will use "gpg2john" to convert the key into a hash:

![gpg2john_hash](https://user-images.githubusercontent.com/76821053/119221110-ab59d380-bae5-11eb-8b24-f0857333939c.png)

![cat_hash](https://user-images.githubusercontent.com/76821053/119221121-b7459580-bae5-11eb-95e8-4e969e917bfd.png)

Now just crack it with john the ripper:

![john_the_ripper](https://user-images.githubusercontent.com/76821053/119221136-c4fb1b00-bae5-11eb-88de-0b32ddcb702d.png)

We found our passphrase, now we can decrypt the credential.pgp inside the target machine:

![decrypt_gpg](https://user-images.githubusercontent.com/76821053/119221153-dc3a0880-bae5-11eb-8ef2-f54f1bb16f72.png)

We have access to ther user “merlin”:

![su_merlin](https://user-images.githubusercontent.com/76821053/119221223-3935be80-bae6-11eb-89e7-f386e0f57127.png)

“merlin” has SUID permissions to run zip. 

![merlin_sudo](https://user-images.githubusercontent.com/76821053/119221232-494d9e00-bae6-11eb-8c5b-fe13e864bb69.png)

With a quick search at GTFOBins we can find our way to escalate privileges to root.

https://gtfobins.github.io/gtfobins/zip/

![gtfobins_zip](https://user-images.githubusercontent.com/76821053/119221276-769a4c00-bae6-11eb-9973-07b4541585f6.png)

We just need to tweek the commands a little bit.

TF=$(mktemp -u)

sudo /usr/bin/zip $TF /etc/hosts -T -TT 'sh #'     → we need to use sudo and specify the location of the zip binary.

![zip_privesc](https://user-images.githubusercontent.com/76821053/119221296-8fa2fd00-bae6-11eb-9537-29a05c1e07c9.png)

Has we can see above we have root privileges! 

![root_flag](https://user-images.githubusercontent.com/76821053/119221305-9b8ebf00-bae6-11eb-8105-5b3b9aa1ea0f.png)

Success , found the last flag!

 

