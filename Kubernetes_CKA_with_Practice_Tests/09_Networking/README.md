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


## Network Namespaces


## CNI


## Pod Networking


## Service Networking


## Ingress

