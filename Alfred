First we need to scan for all open ports using nmap, but since the machine doesn't respond to ping (ICMP) we need to adjust the scan:

nmap  -Pn -A -p- 10.10.101.250 -T4

-Pn   → Because the machine don't respond to ping
-A    → Enable OS detection, version detection, script scanning, and traceroute
-p-   → Scan all ports
-T4   → To accelerate the scan (this loses some accuracy in the scan), otherwise it would more time to complete.

![Alfred_nmap_scan](https://user-images.githubusercontent.com/76821053/118015957-c04b9f80-b34c-11eb-9308-aa706a8a37d0.png)

