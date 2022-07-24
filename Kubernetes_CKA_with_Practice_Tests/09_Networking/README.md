# Networking

## Resources
- 

---
## Switching and Routing
### Switching
- switch connects machines (PCs, VMs, etc.) to build a network
- each machine or host needs interface such as `eth0` to connect to switch
- list interfaces:
```commandline
ip link
```
- assign IP addresses to systems on same network
```commandline
ip addr add 192.168.1.10/24 dev eth0 # system1
```
```commandline
ip addr add 192.168.1.11/24 dev eth0 # system2
```
- communication between both systems via switch
```
ping 192.168.1.11
```
- switches also have IP addresses themselves, for example `192.168.1.0` in network1 and `192.168.2.0` in network2

### Routing
- router as another network device (server) connects two networks to enable communication
- router get an IP assigned for each network, for example: `192.168.1.1` in network1 and `192.168.2.1` in network2
- Gateway is "door" to other networks
- show kernels routing table
```
route
```
- add entry to routing table of host system to connect to second network
```
ip route add 192.168.2.0/24 via 192.168.2.1 # connect switch of network1 with IP of router in network2
ip route add 172.217.194.0/24 via 192.168.2.1 # connect google server to router to connect to internet
ip route add default via 192.168.2.1 # add route to unknown servers to connect to via this router
```
- by default on Linux, packages are not forwarded to other interfaces such as from `eth0` to `eth1`
  - change default behaviour: 
    - `echo 1 > /proc/sys/net/ipv4/ip_forward`
    - value `net.ipv4.ip_forward = 1` in `/etc/sysctl.conf`

#### set values in `/etc/`-network-interfaces file for consistency after rebooting


## DNS and CoreDNS
- resolving domain names to corresponding IP addresses via single entries in `/etc/hosts` file
  - `192.168.1.10   db`
- centralization due to increasing number of servers and addresses to resolve via DNS server
- host has DNS resolution file at `/etc/resolv.conf`
  - specification of address of DNS: `nameserver   192.168.1.100`
- for host individual entries to resolve create entry in local host `/etc/hosts` file
- by default: local host file is preferred before dns
  - `/etc/nsswitch.conf`
- add entr to DNS to forward unknown requests (`www.github.com`) to another DNS to resolve other addresses of the internet, such as `8.8.8.8`

### Domain Names
- `www.google.com`
  - `.`: Root
  - `.com`: Top Level Domain
  - `google`: 
  - `www`: Subdomain (i.e.: `maps`, `apps`, etc.)
- chaining of several DNS to reach the final IP address
  - Root DNS
  - .com DNS
  - google DNS
  - forward to server referring to subdomain and responding with IP address of server

#### Resolving addresses for internal use
- add entry to `/etc/resolv.conf` to resolve `web` instead of `web.mycompany.com`
```
search    mycompany.com  prod.mycompany.com
```
- subdomain is resolved using `search` entry

### Testing DNS lookup
- `nslookup www.google.com` 
  - searches DNS server, not the local `/etc/hosts`
- `dig www.google.com`
  - shows more details


## Network Namespaces


## CNI


## Pod Networking


## Service Networking


## Ingress

