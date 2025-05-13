Linux network tools are a collection of command-line utilities and graphical applications designed to help users manage and troubleshoot networking-related tasks on Linux systems. These tools are widely used by system administrators, network engineers, and users to configure, monitor, and diagnose network connections and services. Here are some common scenarios and the corresponding Linux network tools:

1. **Checking Network Interfaces and IP Addresses:**
    - `ifconfig` : Display information about network interfaces and IP addresses.
    - `ip` : A more powerful tool for configuring network interfaces, routing, and more.
2. **Monitoring Network Connections:**
    - `netstat` : Display information about active network connections.
    - `ss` : A replacement for `netstat` with more features.
    - `nload` : Monitor network bandwidth usage in real-time.
    - `iftop` : Monitor network bandwidth usage by individual connections.
3. **Checking DNS Configuration:**
    - `nslookup` : Query DNS servers for domain name resolution.
    - `dig` : A more advanced tool for DNS queries.
    - `host` : Query DNS for hostnames.
4. **Pinging Hosts and Checking Connectivity:**
    - `ping` : Send ICMP echo requests to test network connectivity.
    - `traceroute` or `traceroute6` : Trace the route packets take to reach a destination host.
    - `mtr` : A combination of `ping` and `traceroute` for continuous monitoring.
5. **Configuring Network Settings:**
    - `ifconfig` or `ip` : Configure network interfaces, IP addresses, netmasks, and more.
    - `/etc/network/interfaces` : Configuration file for network interfaces on Debian-based systems.
    - `/etc/sysconfig/network-scripts/ifcfg-*` : Configuration files for network interfaces on Red Hat-based systems.
6. **Firewall Configuration:**
    - `iptables` : A command-line firewall tool for configuring packet filtering rules.
    - `ufw` : A user-friendly interface for managing `iptables` rules.
    - `firewalld` : Firewall management tool for systems using the `firewalld` service.
7. **Network Diagnostics:**
    - `tcpdump` : Capture and analyze network traffic.
    - `wireshark` : A graphical network protocol analyzer.
    - `netcat` (nc): Utility for reading/writing data across network connections.
    - `telnet` : Connect to remote hosts using the Telnet protocol.
8. **Remote Access and File Transfer:**
    - `ssh` : Securely access remote systems.
    - `scp` : Securely copy files between systems.
    - `rsync` : Synchronize files and directories between systems.
9. **Network Performance Testing:**
    - `iperf` : Measure network bandwidth between two systems.
    - `nuttcp` : Network performance measurement tool.
    - `pingplotter` : A graphical tool for visualizing network latency and packet loss.

## Checking Network Interfaces and IP Addresses:

The `ip` command in Linux is a powerful and versatile network configuration and management tool. It is used for a wide range of networking tasks, including configuring network interfaces, managing routing tables, and monitoring network performance. The `ip` command is part of the `iproute2` package and replaces several older networking utilities, making it the go-to tool for network-related tasks on modern Linux systems. Here's an overview of some common uses of the `ip` command:

1. **Display Network Interfaces and Addresses:**
    - `ip address show` or `ip addr show` : Display information about all network interfaces and their associated IP addresses.
    ```
    ip address show
    ```
    or
    ```
    ip addr show
    ```
    or
    ```
    ip a
    ```
    
2. **Configure Network Interfaces:**
    - `ip link set <interface> up` : Bring a network interface up (activate it).
    - `ip link set <interface> down` : Bring a network interface down (deactivate it).
    - `ip addr add <ip_address>/<subnet_mask> dev <interface>` : Assign an IP address and subnet mask to a network interface.
    - `ip addr delete <ip_address>/<subnet_mask> dev <interface>` : Remove an IP address from a network interface.
    - `ip route add default via <gateway>` : Set the default gateway for routing.
3. **Routing and Static Routes:**
    - `ip route show` : Display the routing table, including default and static routes.

- ip route add /<subnet_mask> via dev : Add a static route to the routing table.
- ip route del /<subnet_mask> : Delete a static route from the routing table.

4.**Managing Network Bridge Devices:**

- ip link add name <bridge_name> type bridge: Create a network bridge device.
- ip link set dev master <bridge_name>: Add a network interface to a bridge.
- ip link set dev nomaster: Remove a network interface from a bridge.

5.**Tunneling and VPNs:**

- `ip tunnel add <tunnel_name> mode <tunnel_mode> remote <remote_address> local <local_address>` : Create a network tunnel interface.
- `ip tunnel delete <tunnel_name>` : Delete a network tunnel interface.

6.**Quality of Service (QoS):**

- `ip link set dev <interface> qlen <queue_length>` : Set the length of the transmit queue for a network interface.
- `ip link set dev <interface> txqueuelen <queue_length>` : Set the transmit queue length for a network interface.

7.**Monitoring Network Statistics:**
- `ip -s link show <interface>` : Display detailed statistics for a network interface.
- `ip -s route show` : Display statistics for the routing table.

8.**Multicast Configuration:**
- `ip maddress show` : Display multicast addresses.
- `ip mroute show` : Display multicast routing information.

9.**Policy-Based Routing:**

- `ip rule add from <source> table <table>` : Add a policy-based routing rule.
- `ip rule del from <source> table <table>` : Delete a policy-based routing rule.

10.**Network Namespace:**

- `ip netns add <namespace_name>` : Create a network namespace.
- `ip netns exec <namespace_name> <command>` : Run a command within a specific network namespace.
- `ip netns list` : List available network namespaces.

```dockerfile
ip netns add softserve
```

```php
ip netns list
```

The `ip` command offers a comprehensive set of features for managing various aspects of networking in Linux. It is a versatile tool that can be used for both basic and advanced networking tasks, making it essential for network administrators and Linux users. You can access detailed help and usage information for the `ip` command by running `ip --help` or `man ip` in your terminal.
### Monitoring Network Connections

## netstat

1.`netstat` : Display information about active network connections. `netstat` is a command-line network utility tool available in most Unix-like operating systems, including Linux. Setup

```
apt install net-tools
```

Describe the primary purpose of netstat:

- To display information about active network connections.
    
- To reveal network-related statistics.
    
- To help troubleshoot network-related issues.
    
    Basic Syntax:
    

> netstat [options]

Some common options used with netstat: -t: Display TCP connections.

```
netstat -t
```

-u: Display UDP connections.

```
netstat -u
```

-l: Show listening ports.

```
netstat -l
```

-n: Display numeric addresses and ports (avoids DNS resolution).

```
netstat -n
```

**ss** : A replacement for `netstat` with more features.

### Monitoring Network Connections

## ss (socket statistics)

The **ss (socket statistics)** tool is a CLI command used to show network statistics. The ss command is a simpler and faster version of the now obsolete netstat command. Together with the ip command, ss is essential for gathering network information and troubleshooting network issues

Syntax: ss OR ss <option 1> <option 2> <option 3> Main options: -n, --numeric : Do not try to resolve service names. Show exact bandwidth values, instead of human-readable. -a, --all Display both listening and non-listening (for TCP this means established connections) sockets. -l, --listening Display only listening sockets (these are omitted by default). -e, --extended Show detailed socket information. -i, --info Show internal TCP information.

[https://man7.org/linux/man-pages/man8/ss.8.html](https://man7.org/linux/man-pages/man8/ss.8.html)

**List all** listening and non-listening connections with:

```
ss -a
```

To display **only listening sockets**, which are omitted by default, use:

```
ss -l
```

To list **TCP connections**, add the **-t** option to the **ss** command:

```
ss -t
```

### List All TCP Connections

Combine the options -a and -t with the ss command to output a list of all the TCP connections:

```
ss -at
```

### List Connections to a Specific IP Address

List connections to a specific destination IP address with:

```yaml
ss dst 172.17.0.1
```

### Check Process IDs

To show process IDs (PID), use:

```
ss -p
```

### List Summary Statistics

List the summary statistics for connections with:

```
ss -s
```

### Filter Connections

The ss command allows advanced filtering of results and searching for specific ports or TCP states.

### Filter Using TCP States

Filter TCP connections using the TCP predefined states: ss state For example, to find all listening TCP connections:

```
ss -t state listening
```

### Filter by Port Number

Filter for a specific destination port number or port name: ss dst :

For example:

```
ss dst :53512
```

```
ss dst :ssh
```

Combine multiple queries for more advanced filtering. For example, find all connections with a destination port 5228 or source port mysql:

```
ss -a dst :5228 
ss -a src :ssh
```

To show all listening TCP connections, use the following:

```
ss -tln
```

This command will show all listening TCP connections on the system, along with the corresponding port number and the process ID that is listening on that port.

```
ss -tulpn
```

### Monitoring Network Connections

## nload

**nload** is a command-line tool to keep an eye on network traffic and bandwidth usage in real time. It helps you to **monitor incoming and outgoing traffic** using graphs and provides additional information such as the total amount of transferred data and min/max network usage.

#### Install nload

On Debian/Ubuntu, nload can be installed from the default system repositories as shown.

```
apt install nload
```

#### How to Use nload to Monitor Linux Network Usage

Once you have started nload, you can switch between the devices (which you can specify either on the command-line or which were auto-detected) by pressing the left and right arrow keys:

```
nload
```

Available Key Shortcuts After running nload, you may use these shortcut keys below:

Use left and right arrow keys or Enter/Tab key to switch the display to the next network device or when started with the -m flag, to the next page of devices. Use F2 to show the option window. Use F5 to save current settings to the user’s config file. Use F6 to reload settings from the config files. Use q or Ctrl+C to quit nload.

Use -a period to set the length in seconds of the time window for average calculation (default is 300):

```yaml
 nload -a 400
```

The -t interval flag sets the refresh interval of the display in milliseconds (default value is 500). Note that specifying refresh intervals shorter than about 100 milliseconds makes traffic calculation very unprecise:

```yaml
nload -ma 400 -t 600
```

## iftop

Interface TOP (IFTOP) is a real time console-based network bandwidth monitoring tool. Install iftop

```
apt install iftop
```

Run

```
iftop
```

## lsof

The “lsof” command tool in Linux is one of the many built-in tools that’s super useful for checking out the “list of open files”. Yes, the term “lsof” is the abbreviation of the task.

For finding out all the processes that are currently using a certain port, call “lsof” with the “-i” flag followed by the protocol and port information.

lsof -i<46><@hostname|host_address> :<service|port>

For example, to check out all the programs currently accessing port 22 over TCP/IP protocol, run the following command.

```
lsof -i TCP:22
```

This method can also be used to show all the processes that are using ports within a certain range, for example, 1 to 1000. The command structure is similar to before with a little magic at the port number part.

```
lsof -i TCP:1-1000
```

Listing network connections The following command will report all the network connections from the current system

```
lsof -i
```
### Checking DNS Configuration

## nslookup

**Nslookup** (stands for “Name Server Lookup”) is a useful command for getting information from the DNS server. It is a network administration tool for querying the Domain Name System (DNS) to obtain domain name or IP address mapping or any other specific DNS record. It is also used to troubleshoot DNS-related problems. To exit interactive mode, type: **exit**

Syntax

nslookup [option] [hosts]

```
nslookup softserveinc.com
```

**-domain=[domain-name]** allows you to change the default DNS name

```
nslookup -domain=google.com
```

-port=[port-number] Use the -port option to specify the port number for queries. By default, nslookup uses port 53 for DNS queries

```
nslookup -port=53 google.com
```

Lookup for any record We can also view all the available DNS records using the -type=any option.

```go
nslookup -type=any google.com
```

Lookup for an ns record. NS (Name Server) record maps a domain name to a list of DNS servers authoritative for that domain. It will output the name serves which are associated with the given domain.

```go
nslookup -type=ns google.com
```

Lookup for an mx record. MX (Mail Exchange) maps a domain name to a list of mail exchange servers for that domain. The MX record says that all the mails sent to “google.com” should be routed to the Mail server in that domain.

```go
nslookup -type=mx google.com
```

### Checking DNS Configuration

## dig

**DIG** (Domain Information Groper command) is a network tool with a basic command-line interface that serves for making different DNS (domain name system) queries. You can use the DIG command to: Diagnose your name servers. Check all of them or each individual server and their response.

Find the website’s IP address

```
dig google.com
```

Finds the IP address of a certain domain name, for example Google.

It will do a DNS query, looking for the A records. They have the IP addresses which correspond to the domain name form the query.

This dig command will give you a lot of extra information too. Data like the version of the DIG command you are using, a header that shows you what you did and who answered you, the port and protocol you used (usually UDP), the time it took for the query, the TTL of the record, and the server which answered you and other.

If you don’t want so much information, go for the short answer with this command:

```
dig google.com +short
```

Find the name servers, responsible for your domain.

```
dig NS google.com +short
```

Which is the responsible mail server for your domain?

```
dig MX google.com +short
```

Reverse DNS check, IP address to hostname. Allows you to determine which IP address the domain name is associated with, for example:

```yaml
dig -x 142.250.186.46
```

See when the cache with the answer will expire, for example:

```
dig google.com +noall +answer
```

### Checking DNS Configuration

## host

The **host** command in Linux is used for DNS lookup operations. This command is used to find the IP address of a specific domain name or if you want to know the domain name of a specific IP address. You can also find more specific information about a domain by specifying the appropriate option along with the domain name.

Syntax:

host [-aCdlriTWV] [-c class] [-N ndots] [-t type] [-W time] [-R number] [-m flag] hostname [server]

Print version number and exit:

```
host -V
```

Рrint the IP address details of the specified domain:

```
host google.com
```

Рrint the domain details record of the specified IP address:

```
host google.com
```

Indicates the type of query or whether verbose output is enabled :

```
host -a google.com
```

```
host -v google.com
```

Show the type of query:

```
host -t ns google.com
```

### Pinging Hosts and Checking Connectivity

## ping

ping is the primary TCP/IP command used to troubleshoot connectivity, reachability, and name resolution. Used without parameters, this command displays Help content. You can also use this command to test both the computer name and the IP address of the computer.

Note: Ping has a different syntax for key parameters on Windows and Linux.

Syntax and help:

```
ping -h 
```

Pinging host by name 5 times:

```yaml
ping -c 5 google.com
```

Pinging host by IP address 5 times:

```yaml
ping -c 5 172.30.1.2
```

### Pinging Hosts and Checking Connectivity

## tracert

The tracert command is a Command Prompt command that's used to show several details about the path that a packet takes from the computer or device you're on to whatever destination you specify.

You might also sometimes see the tracert command referred to as the trace route command or traceroute command.

Setup:

```
apt install traceroute 
```

Syntax:

tracert traceroute [ -46dFITnreAUDV ] [ -f first_ttl ] [ -g gate,... ] [ -i device ] [ -m max_ttl ] [ -N squeries ] [ -p port ] [ -t tos ] [ -l flow_label ] [ -w MAX,HERE,NEAR ] [ -q nqueries ] [ -s src_addr ] [ -z sendwait ] [ --fwmark=num ] host [ packetlen ]

Help about traceroute options:

```
traceroute
```

Test network:

```
traceroute google.com
```

### Pinging Hosts and Checking Connectivity

## mtr

The mtr command is a combination of ping and traceroute commands. It is a network diagnostic tool that continuously sends packets showing ping time for each hop. It also displays network problems of the entire route taken by the network packets.

Using mtr:

```
mtr
```

```
mtr -t localhost
```

```
mtr -t google.com
```

The -t option indicates we want to see the output in the curses-based terminal. If we hadn’t used this option, we’d receive output in GUI mode, if possible. The numbers change depending on the network activity. To quit this curses-based terminal, we could press q (quit).

Changing the Displayed Columns Suppose we are only interested in the Snt and Avg columns. We could launch mtr with the -o option:

```javascript
mtr -o 'SA' -t google.com
```

