Let's start our enumeration fase by running “nmap” to check for all open ports on the target system:

```
nmap -sV -sC -p- superspam.thm -oN nmap.txt

-sV    →  Probe open ports to determine service
-sC    →  Scan using the default set of scripts
-p-    →  Scan all ports
-oN    →  Save the ouput of the scan in a file
```

