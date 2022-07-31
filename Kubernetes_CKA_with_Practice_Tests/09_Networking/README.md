# Networking

## Resources
- https://github.com/kubernetes/dns/blob/master/docs/specification.md
- https://coredns.io/plugins/kubernetes/

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

### CoreDNS
- configuration of a host as DNS server
- download and install CoreDNS
```
wget https://github.com/coredns/coredns/releases/download/1.4.0/coredns_1.4.0_linux_amd64.tgz
```
- extract the binary
```
tar -xzvf coredns_1.4.0_linux_amd64.tgz
```
- run the CoreDNS binary
```
./coredns
```
- default port is `53`
- put entries for DNS in local servers `/etc/hosts` file
- configuration of CoreDNS to use the `/etc/hosts` file
  - CoreDNS loads configuration from file named `CoreFile`
```
cat > Corefile
. {
    hosts /etc/hosts
}
```
- restart CoreDNS binary
```
./coredns
```


## Network Namespaces
- separating containers into network namespaces as defined, restricted areas
- admin on host has access to all namespaces, all processes within a separated namespace run assuming the namespace is the host system
- create a new network namespace
```
ip netns add <network-namespace-name>
```
- list all network namespaces
```
ip netns
```
- list interfaces in specific namespace
```
ip netns exec <namespace-name> ip link
ip -n <namespace-name> link
```
- same for ARP tables and routing table
```
ip netns exec <namespace-name> arp
ip netns exec <namespace-name> route
```
- connectivity between two namespaces
  - create virtual interfaces
  ```
  ip link add veth-<namespace1> type veth peer name veth-<namespace2>
  ```
  - link virtual interfaces to namespaces
  ```
  ip link set veth-<namespace1> netns <namespace1
  ```
  - add IP addresses to interfaces
  ```
  ip -n <namespace1> addr add 192.168.15.1 dev veth-<namespace1>
  ```
  - bring up the interface for each device in respective namespace
  ```
  ip -n <namespace1> link set veth-<namespace1> up
  ```
  - remove link from both namespaces
  ```
  ip -n <namespace1> link del veth-<namespace1>
  ```
- virtual switch on host for virtual network connecting several namespaces and connect namespaces to virtual switch
  - Linux Bridge for creating a virtual switch
  ```
  ip link add v-net-0 type bridge # name: v-net-0
  ```
  - virtual switch is just another interface from host perspective
  - turn up virtual switch
  ```
  ip link set dev v-net-0 up
  ```
  - create connection between interfaces namespaces to bridge
  ```
  ip link add veth-<namespace1> type veth peer name veth-red-br # name of virtual interface of bridge
  ```
  - connect namespace and bridge (in namespace red)
  ```
  ip link set veth-red netns red
  ip link set veth-red-br master v-net-0
  ```
  - create IP addresses
  ```
  ip -n red addr add 192.168.15.1 dev veth-red
  ```
  - turn devices up
  ```
  ip -n red link set veth-red up
  ```
  - to reach namespaces from host via switch, assign switch (=interface on host) an IP address
  ```
  ip addr add 192.168.15.5/24 dev v-net-0
  ```
  - configure bridge to reach outside networks 192.168.1.0/24 by adding a gateway on host and adding NAT for response resolution of hosts IP address
  ```
  ip netns exec red ip route add 192.168.1.0/24 via 192.168.15.5
  iptables -t nat -A POSTROUTE -s 192.168.15.0/24 -j MASQUERADE
  ```
  - adding a default gateway to host using IP of virtual interface
  ```
  ip netns exec red ip route add default via 192.168.15.5
  ```
  - IP forwarding
  ```
  iptables -t nat -A PREROUTING --dport 80 --to-destination 192.168.15.2:80 -j DNAT
  ```


## CNI - Container Network Interface
### CNI Plugin
- important default locations on controlplane node (access as root)
```
# check for running CNI
ps -aux | grep kubelet | grep --network-plugin
```
```
# location of available CNI plugins
ls /opt/bin/cni
```
```
# installed CNI plugin
ls /etc/cni/net.d/
```
```
# check for binary executable in network file (type parameter)
# file will be run by kubelet after a container and namespace are created
cat /etc/cni/net.d/<filename-of-network-plugin>
```

### Installing Weavenet
- installing weavenet: https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#-installation
- configure weavenet installation: https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#-changing-configuration-options
- install weavenet CNI with parameter to change the default IP range to avoid network overlapping
```
 kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.IPALLOC_RANGE=10.50.0.0/16"
```

### IP Adress Management
- CNI plugin is responsible for managing IP adresses of pods and bridge networks on nodes
- `ipam`-section in CNI configuration file
```
cat /etc/cni/net.d/net-script.conf
```
- `ipam.type: "host-local"` for local files on each node with IP information 
- dynamic allocation: DHCP
- Weavenet default range: `10.32.0.0/12` equals IP adresses from `10.32.0.1` to `10.47.255.254` results in 1.048.574 IP adresses
- Weavenet distributes these IP addresses over existing nodes in cluster such that each node gets an almost equally portion to assign IP addresses to pods 


## Pod Networking


## Service Networking


## Ingress

