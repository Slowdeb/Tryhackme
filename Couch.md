Room: [Couch](https://tryhackme.com/room/couch)

Difficulty: Easy

Overview: This is a semi-guided challenge, where we will take advantage of a vulnerable/misconfigured database which will lead us to a simple privilege escalation just by doing some user reconnaissance taking advantage of a docker container.

------------------------------------------------------------------------------------------------------------------------------------------------------------------

For enumerating all open ports on the remote system we can use “nmap”:

```
nmap -sV -sC -p- couch.thm -oN nmap.txt

-sV       → Probe open ports to determine service
-sC       → Scan using the default set of scripts
-p-       → Scan all ports
couch.thm → ip address of the room
-oN       → Save the ouput of the scan in a file
```

![image](https://user-images.githubusercontent.com/76821053/124316231-314d4d80-db6d-11eb-8c6b-5b788f57d222.png)

Instead of running a directory brute forcer like “gobuster”, to find the path to the database management system we can simply search google for it:

![image](https://user-images.githubusercontent.com/76821053/124316313-53df6680-db6d-11eb-87a0-697069252326.png)

The web administration tool doesn't ask for any password accessing the path that we found above. 

Browsing through the options of http couchdb we can find the path to list all databases:

![image](https://user-images.githubusercontent.com/76821053/124316339-5fcb2880-db6d-11eb-9c23-c357eebcb46f.png)

In the web administrative tool we can see this all of this databases information:

![image](https://user-images.githubusercontent.com/76821053/124316371-68236380-db6d-11eb-94e1-26dc67c98480.png)

There is a special database called “secret” and inside there is some credentials:

![image](https://user-images.githubusercontent.com/76821053/124316391-71143500-db6d-11eb-9400-baf2e1fb6ea2.png)

With this credentials we can access the machine through “ssh”:

![image](https://user-images.githubusercontent.com/76821053/124316416-7a9d9d00-db6d-11eb-9da6-26920c2fe2e9.png)

The user.txt flag can be found at “atena” home directory:

![image](https://user-images.githubusercontent.com/76821053/124316438-81c4ab00-db6d-11eb-8581-96cc26a353d6.png)

Our privilege escalation is has simple has listing all the contents of “atena” home directory:

![image](https://user-images.githubusercontent.com/76821053/124316459-89844f80-db6d-11eb-8560-0d55f3fb6c16.png)

Reading “.bash_history” we will find a history of commands: 

![image](https://user-images.githubusercontent.com/76821053/124316493-9143f400-db6d-11eb-833c-14a6265164c5.png)

There is a docker container running at port 2375 which will lead us to our last flag.

![image](https://user-images.githubusercontent.com/76821053/124316515-9a34c580-db6d-11eb-8942-42954fb3f133.png)

From the above “.bash_history” file commands, we can see that the user removed a “/flag” folder and created a root.txt file, so we can search for “root.txt” in the docker container:

![image](https://user-images.githubusercontent.com/76821053/124316532-a3259700-db6d-11eb-92aa-1a7792b3e6ea.png)

Success, final flag found!!



