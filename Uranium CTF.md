Room: [Uranium CTF](https://tryhackme.com/room/superspamr)

Difficulty: Hard

Overview: Good reconaissance and enumeration of the machine is the key, gather information about users, access misconfigured FTP, crack a .cap wifi file which will lead us to a reverse shell. Go through encryption files to achieve a horizontal privesc and eventualy take advantage of VNC service to reach root.   

--------------------------------------------------------------------------------------------------------------------------------------------------------------------

Let's start our enumeration fase by running “nmap” to check for all open ports on the target system:

```
nmap -sV -sC -p- uranium.thm -oN nmap.txt

-sV  →  Probe open ports to determine service
-sC  →  Scan using the default set of scripts
-p-  →  Scan all ports
-oN  →  Save the ouput of the scan in a file
uranium.thm → ip of the room
```

