Room: [Uranium CTF](https://tryhackme.com/room/uranium)

Difficulty: Hard

Overview: This room is a good representation of how social engeneering can be exploited by hackers. Through an email we will get a reverse shell and in a series of meticulous linux privesc exploits we will take over the room. 

--------------------------------------------------------------------------------------------------------------------------------------------------------------------

Let's start our enumeration fase by running “nmap” to check for all open ports on the target system:

```
nmap -sV -sC -p- uranium.thm -oN nmap.txt

-sV  →  Probe open ports to determine service
-sC  →  Scan using the default set of scripts
-p-  →  Scan all ports
-oN  →  Save the ouput of the scan in a file
uranium.thm → ip address of the room
```

