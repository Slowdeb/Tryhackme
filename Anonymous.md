We will start enumerating the machine with nmap:

**nmap -sV -sC -A -p anonymous.thm -oN nmap.txt**

![nmap1](https://user-images.githubusercontent.com/76821053/118256768-c30dd800-b4a5-11eb-8835-6cbb9f426a20.png)
![nmap2](https://user-images.githubusercontent.com/76821053/118256851-d7ea6b80-b4a5-11eb-9455-def0bbef6f87.png)

Open ports and their versions running on the remote system:

21    →  vsftpd 2.0.8 

22    →  OpenSSH 7.6p1

139  →  netbios-ssn Samba smbd 3.X - 4.X

445  →  netbios-ssn Samba smbd 4.7.6

From the nmap scan we can imediately answer the first three questions.

Continuing enumerating the machine. 

The ftp server lets us login with user “anonymous” without any password.

![ftpdownload](https://user-images.githubusercontent.com/76821053/118257028-1122db80-b4a6-11eb-90cf-8b198bb70edf.png)

I downloaded three files: **clean.sh**, **removed_files.log**, **to_do.txt**.

**to_do.txt**:

Leaves a hint that the system is not safe. Meaning that maybe some services have anonymous login.

![todo](https://user-images.githubusercontent.com/76821053/118257209-45969780-b4a6-11eb-9215-7690384324ed.png)

**clean.sh**:

Is a bash script that when ran it checks if there is files in the /tmp directory and removes them. 

![readclean](https://user-images.githubusercontent.com/76821053/118257284-5ba45800-b4a6-11eb-9f3e-5db72feed54b.png)

**removed_files.log**:

When the bash script runs it posts status messages to this .log file.

![remotelog](https://user-images.githubusercontent.com/76821053/118257366-75de3600-b4a6-11eb-93e9-5790a2a1c534.png)

If we monitored the **removed_files.log** file we can see that the script **clean.sh** is running has a cronjob since it keeps posting to the .log file everyminute:

If we use "wc -l" we can check how many lines (post messages) the file has.

![remoteWC1](https://user-images.githubusercontent.com/76821053/118257588-b63db400-b4a6-11eb-882b-1693c6c60e0d.png)

 After a few minutes we have 72 entries.
 
 ![remoteWC2](https://user-images.githubusercontent.com/76821053/118257627-c05fb280-b4a6-11eb-8468-1d11bf8a02b3.png)
 
 Lets try to upload files to the ftp server:
 
 ![ftpuploadtest](https://user-images.githubusercontent.com/76821053/118257708-d9686380-b4a6-11eb-9d0e-098654b93554.png)
 
 Has we can see the system allows file upload, so lets change the **clean.sh** script a bit and upload it:
 
 ![cleantest](https://user-images.githubusercontent.com/76821053/118257824-f7ce5f00-b4a6-11eb-8b4f-6e3e435514d6.png)
 
 The new message was posted to the .log file after a minute. 
 
 ![removemessage](https://user-images.githubusercontent.com/76821053/118258135-6ca19900-b4a7-11eb-8c7f-6b00cb9cff3d.png)


 




