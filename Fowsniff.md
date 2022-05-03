Room: [Fowsniff CTF](https://tryhackme.com/room/ctf)

Difficulty: Easy

Overview: 

------------------------------------------------------------------------------------------------------------------------------------------------------------------

For enumerating all open ports on the remote system we can use “nmap”:

```
nmap -sV -sC -p- ide.thm -oN nmap.txt

-sV         → Probe open ports to determine service
-sC         → Scan using the default set of scripts
-p-          → Scan all ports
fowsniff.thm  → ip address of the room
-oN         → Save the ouput of the scan in a file
```
nmap scan:

![image](https://user-images.githubusercontent.com/76821053/166565008-389610f8-322e-4664-863a-c1e9ec04125c.png)

