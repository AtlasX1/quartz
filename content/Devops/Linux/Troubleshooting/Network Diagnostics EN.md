### Network Diagnostics

## tcpdump

The **tcpdump** utility is a command-line network packet analyser. It is absolutely essential for diagnosing networking issues from the server side.

Install

```javascript
apt-get install tcpdump
```
```
Syntax **tcpdump** [ -AdDefIKlLnNOpqRStuUvxX ] [ -B buffer_size ] [ -c count ]

[ -C file_size ] [ -G rotate_seconds ] [ -F file ] [ -i interface ] [ -m module ] [ -M secret ] [ -r file ] [ -s snaplen ] [ -T type ] [ -w file ] [ -W filecount ] [ -E spi@ipaddr algo:secret,... ] [ -y datalinktype ] [ -z postrotate-command ] [ -Z user ] [ expression ]

[https://www.tcpdump.org/](https://www.tcpdump.org/)[https://www.tcpdump.org/]
```

To capture packets for troubleshooting or analysis, tcpdump requires elevated permissions, so in the following examples most commands are prefixed with sudo.

To begin, use the command tcpdump --list-interfaces (or -D for short) to see which interfaces are available for capture:

```
sudo tcpdump -D
```

Let's use it to start capturing some packets. Capture all packets in any interface by running this command:

```php
sudo tcpdump --interface any
```

Tcpdump continues to capture packets until it receives an interrupt signal. You can interrupt capturing by pressing Ctrl+C.

In this case, since I am connected to this server using ssh, tcpdump captured all these packets. To limit the number of packets captured and stop tcpdump, use the -c (for count) option:

```yaml
sudo tcpdump -i any -c 5
```

**Protocol** To filter packets based on protocol, specifying the protocol in the command line. For example, capture ICMP packets only by using this command:

```
sudo tcpdump -i any -c5 icmp
```

In a different terminal, try to ping another machine:

```
ping opensource.com
```

Back in the tcpdump capture, notice that tcpdump captures and displays only the ICMP-related packets. In this case, tcpdump is not displaying name resolution packets that were generated when resolving the name opensource.com

**Port** To filter packets based on the desired service or port, use the port filter. For example, capture packets related to a web (HTTP) service by using this command:

```yaml
sudo tcpdump -i any -c5 -nn port 80
```

Complex expressions You can also combine filters by using the logical operators and and or to create more complex expressions. For example, to filter packets from source IP address 172.30.1.2 and service SSH only, use this command:

```yaml
sudo tcpdump -i any -c5 -nn src 172.30.1.2 and port 22
```

In a different terminal, try to connect to another machine via ssh (user ubuntu password ubuntu):

```
ssh ubuntu@172.30.1.2
```

Back in the tcpdump capture, notice that tcpdump captures and displays.

**Checking packet content** In the previous examples, we're checking only the packets' headers for information such as source, destinations, ports, etc. Sometimes this is all we need to troubleshoot network connectivity issues. Sometimes, however, we need to inspect the content of the packet to ensure that the message we're sending contains what we need or that we received the expected response. To see the packet content, tcpdump provides two additional flags: -X to print content in hex, and ASCII or -A to print the content in ASCII.

For example, inspect the SSH content of a request like this:

```yaml
sudo tcpdump -i any -c10 -nn -A port 22
```

In a different terminal, try to connect to another machine:

```
ssh ubuntu@172.30.1.2
```

Back in the tcpdump capture, notice that tcpdump captures and displays.

**Saving captures to a file** Another useful feature provided by tcpdump is the ability to save the capture to a file so you can analyze the results later. This allows you to capture packets in batch mode overnight, for example, and verify the results in the morning. It also helps when there are too many packets to analyze since real-time capture can occur too fast.

To save packets to a file instead of displaying them on screen, use the option -w (for write):

```yaml
sudo tcpdump -i any -c10 -nn -w ssh.pcap port 22
```

In a different terminal, try to connect to another machine:

```
ssh ubuntu@172.30.1.2
```

Back in the tcpdump capture, notice that tcpdump captures and displays.

```
cat ssh.pcap
```

### Network Diagnostics

## netcat

**netcat** (often abbreviated to nc) is a computer networking utility for reading from and writing to network connections using TCP or UDP. The command is designed to be a dependable back-end that can be used directly or easily driven by other programs and scripts. At the same time, it is a feature-rich network debugging and investigation tool, since it can produce almost any kind of connection its user could need and has a number of built-in capabilities. It is able to perform port scanning, file transferring and port listening.

```yaml
nc localhost 22
```

Ctr+C for exit

**Client/Server Connection** A simple client/server connection is between two devices. One device acts as a server (listens) while the other acts as a client (connects).

1. On terminal 1, run the nc command in listen mode and provide a port:

```yaml
nc -lv 1234
```

On terminal 2, run the nc command with the IP address of device 1 in terminal 1 and the port:

```yaml
nc -v 172.30.1.2 1234
```

To finish type Ctrl+C

**Ping Specific Port on Website** Use Netcat as an alternative to the ping command to test a specific port to a website. For example:

```yaml
nc -zv google.com 443 
```

**Scanning Ports** Use the nc command to scan for open ports.

```yaml
nc -zv 172.30.1.2 22
```

**Transfer Files** Netcat allows transferring files through established connections. To see how file transfers work, do the following:

1. Create a sample file on terminal 1 using the touch command:

```
touch file.txt
```

The command creates an empty text file.

1. Create a listening connection on terminal 1 and redirect the file to the nc command:

```yaml
nc -lv 1234 < file.txt
```

On terminal 2, connect to terminal 1 and redirect the file:

```yaml
nc -zv 10.0.2.4 1234 > file.txt
```

Confirm the file transfer is complete using the ls command.

The output shows the file name, indicating the transfer was succesful.