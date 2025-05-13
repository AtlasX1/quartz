### Network Diagnostics

## iperf

**iperf/iPerf3** is a tool for active measurements of the maximum achievable bandwidth on IP networks. It supports tuning of various parameters related to timing, buffers and protocols (TCP, UDP, SCTP with IPv4 and IPv6). For each test it reports the bandwidth, loss, and other parameters. This is a new implementation that shares no code with the original iPerf and also is not backwards compatible. iPerf was orginally developed by NLANR/DAST. iPerf3 is principally developed by ESnet / Lawrence Berkeley National Laboratory. It is released under a three-clause BSD license.

Install

```
apt install iperf
```

```
apt install iperf3 -y
```

Start server on Terminal 1

```
 iperf3 -s
```

Switch to terminal 2 and print next command:

```yaml
iperf3  -c 172.30.1.2
```

This command connecting to host 172.30.1.2, port 5201.
### Network Diagnostics

## nuttcp

**nuttcp** is another network test tool, similar to iperf2 and iperf3, that has a number of unique features.

nuttcp is a network performance measurement tool intended for use by network and system managers. Its most basic usage is to determine the raw TCP (or UDP) network layer throughput by transferring memory buffers from a source system across an interconnecting network to a destination system, either transferring data for a specified time interval, or alternatively transferring a specified number of bytes. In addition to reporting the achieved network throughput in Mbps, nuttcp also provides additional useful information related to the data transfer such as user, system, and wall-clock time, transmitter and receiver CPU utilization, and loss percentage (for UDP transfers).

Install nuttcp:

```
apt install nuttcp -y
```

Run server

```
nuttcp -S
```

Run client:

```yaml
nuttcp -i1 172.30.1.2
```

Send 300 Mbps of UDP in bursts of 20 packets for 5 seconds

```yaml
nuttcp -u -Ri300m/20 -i 1 -T5 172.30.1.2 
```

### Remote Access and File Transfer

## ssh

Secure your critical data and communications between systems, automated applications, and people with defensive cybersecurity. Our solutions defend and safeguard your business secrets and access to them - now and in the future.

Connect to remote server via login / password

```
ssh ubuntu@172.30.1.2
```

To exit type **logout**

Create a ssh key:

```
ssh-keygen
```

Copy key to remote server:

```dockerfile
ssh-copy-id ubuntu@172.30.1.2
```

Connect to remote server with ssh key:

```
ssh ubuntu@172.30.1.2
```

### Remote Access and File Transfer

## scp

The scp command allows you to copy files over ssh connections. This is pretty useful if you want to transport files between computers, for example to backup something. The scp command uses the ssh command and they are very much alike. However, there are some important differences.

How to use SCP

- To copy from a (remote) server to your computer
- To copy from your computer to a (remote) server
- To copy from a (remote) server to another (remote) server In the third case, the data is transferred directly between the servers; your own computer will only tell the servers what to do.

Create a file:

```
ls > file.txt
```

These options are very useful for a lot of things that require files to be transferred, so let's have a look at the syntax of this command:

```javascript
scp file.txt ubuntu@172.30.1.2:/home/ubuntu
```

Connect to remote server:

```
ssh ubuntu@172.30.1.2
```

Check copied file:

```
ls
```