# Networking Basics: Theoretical Foundations and Production Concepts

## Table of Contents
1. Network Models and Layered Architecture
2. IP Addressing and Routing Fundamentals
3. DNS: Domain Name System Architecture
4. Transport Layer Protocols (TCP and UDP)
5. Application Layer Protocols (HTTP/HTTPS)
6. Network Security Fundamentals
7. Content Delivery and Load Balancing
8. Production Networking and Performance

---

## 1. Network Models and Layered Architecture

### 1.1 The OSI Model

The **Open Systems Interconnection (OSI) model** is a conceptual framework defining network communication in 7 layers:

$$\text{Application} \rightarrow \text{Presentation} \rightarrow \text{Session} \rightarrow \text{Transport} \rightarrow \text{Network} \rightarrow \text{Data Link} \rightarrow \text{Physical}$$

Each layer provides services to the layer above and uses services from the layer below.

**Layer 1: Physical Layer**
- Transmission of raw bit streams over physical media
- Characteristics: Voltage levels, cable types, frequencies
- Equipment: Hubs, repeaters, cables
- Example: Copper wire carries 0V (binary 0) and 5V (binary 1)

**Layer 2: Data Link Layer**
- Reliable communication over local network segment
- Frames with MAC addresses (Media Access Control)
- Equipment: Switches, bridges
- Protocols: Ethernet, PPP, HDLC
- Function: Frame synchronization, error detection

**Layer 3: Network Layer**
- Routing across networks (IP)
- Logical addressing (IP addresses)
- Equipment: Routers
- Protocols: IP, ICMP, IGP, BGP
- Function: Best-effort packet forwarding

**Layer 4: Transport Layer**
- End-to-end communication reliability
- Port-based services
- Protocols: TCP (reliable), UDP (unreliable)
- Function: Flow control, error correction, multiplexing

**Layer 5: Session Layer**
- Dialog control and synchronization
- Session establishment, maintenance, termination
- Protocols: NetBIOS, RPC
- Function: Half-duplex/full-duplex conversation management

**Layer 6: Presentation Layer**
- Data format translation and encryption
- Character encoding (ASCII, Unicode)
- Compression and encryption
- Function: Data translation between application and network formats

**Layer 7: Application Layer**
- User services and applications
- Protocols: HTTP, SMTP, FTP, DNS, SSH, Telnet
- Function: Provides network services directly to user applications

**Data Encapsulation (PDU at each layer):**
```
Layer 7: Data
         ↓ (add application header)
Layer 6: APDU (Application Protocol Data Unit)
         ↓ (add presentation header)
Layer 5: PPDU
         ↓ (add session header)
Layer 4: SPDU → Segment (TCP) or Datagram (UDP)
         ↓ (add transport header)
Layer 3: TPDU → Packet (with IP header)
         ↓ (add network header)
Layer 2: NPDU → Frame (with Ethernet header)
         ↓ (add data link header)
Layer 1: Physical bits transmitted over wire
```

### 1.2 TCP/IP Model (DoD Model)

A more practical 4-layer model reflecting actual internet architecture:

**Layer 1: Link Layer**
- Combines OSI Physical and Data Link
- Hardware addressing (MAC)
- Physical transmission

**Layer 2: Internet Layer**
- IP routing and logical addressing
- Equivalent to OSI Network Layer
- Protocols: IP, ICMP, IGMP

**Layer 3: Transport Layer**
- TCP (connection-oriented, reliable)
- UDP (connectionless, unreliable)
- Equivalent to OSI Transport Layer

**Layer 4: Application Layer**
- Combines OSI Session, Presentation, Application
- HTTP, SMTP, DNS, SSH, FTP, Telnet
- End-user services

**Mapping Comparison:**
```
OSI Model              TCP/IP Model
─────────────────────────────────────────
7. Application        │
6. Presentation       ├→ 4. Application
5. Session            │
4. Transport        → 3. Transport
3. Network          → 2. Internet
2. Data Link        │
1. Physical         ├→ 1. Link
```

### 1.3 Packet Structure and Flow

**Packet Encapsulation Through Layers:**
```
Application Data: "GET / HTTP/1.1\r\n..."
        ↓ (add TCP header: sport=random, dport=80, seq, ack, flags)
TCP Segment: [TCP Header | HTTP Request]
        ↓ (add IP header: src=192.168.1.10, dst=93.184.216.34)
IP Packet: [IP Header | TCP Header | HTTP Request]
        ↓ (add Ethernet header: src_MAC=aa:bb:cc:dd:ee:ff, dst_MAC=gateway)
Ethernet Frame: [Ethernet | IP | TCP | Data] [CRC]
        ↓ (convert to bits and transmit)
Physical bits: 101010110010101010...
```

**Path Through Network:**
```
Source Host:
  Application → Layer 4 (TCP adds port 80)
             → Layer 3 (IP adds destination address)
             → Layer 2 (Ethernet adds MAC of gateway)
             → Layer 1 (transmit on wire)

Router 1:
  Layer 1 → 2 (receive Ethernet frame)
         → 2 (check destination MAC - is it mine? yes)
         → 3 (examine IP header, check routing table)
         → 3 (encapsulate in new Ethernet frame to Router 2)
         → 1 (transmit on outgoing interface)

Destination Host:
  Layer 1 → 2 (receive frame, strip Ethernet header)
         → 3 (receive IP packet, check destination IP - is it mine? yes)
         → 4 (receive TCP segment, check port 80)
         → 7 (deliver HTTP request to web server)
```

---

## 2. IP Addressing and Routing Fundamentals

### 2.1 IPv4 Addressing

An **IPv4 address** is a 32-bit identifier represented in **dotted decimal notation**:

$$\text{Address} = \text{Network Bits} + \text{Host Bits}$$

**Example: 192.168.1.100**
```
Binary: 11000000.10101000.00000001.01100100
         ↑      ↑         ↑         ↑
         192    168       1         100
```

**Classful Addressing (Legacy):**
```
Class A: 0xxxxxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx (1-126.x.x.x)
         1 network byte, 3 host bytes
         Networks: 126, Hosts per network: 16,777,214

Class B: 10xxxxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx (128-191.x.x.x)
         2 network bytes, 2 host bytes
         Networks: 16,384, Hosts per network: 65,534

Class C: 110xxxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx (192-223.x.x.x)
         3 network bytes, 1 host byte
         Networks: 2,097,152, Hosts per network: 254

Class D: 1110xxxx (224-239.x.x.x) - Multicast
Class E: 1111xxxx (240-255.x.x.x) - Reserved
```

**CIDR Notation (Classless Inter-Domain Routing):**
```
192.168.1.0/24
         └─ /24 means first 24 bits are network
            remaining 32-24=8 bits are host bits

Network address: 192.168.1.0 (host bits = 0)
Broadcast: 192.168.1.255 (host bits = all 1s)
Usable hosts: 192.168.1.1 to 192.168.1.254
Total addresses: 2^8 = 256, usable = 254
```

**Subnet Mask:**
```
CIDR /24 = Subnet mask 255.255.255.0
           11111111.11111111.11111111.00000000

To find network address:
192.168.1.100 AND 255.255.255.0 = 192.168.1.0

To find host bits:
192.168.1.100 AND 0.0.0.255 = 0.0.0.100
```

**Special Addresses:**
```
127.0.0.1 - Loopback (localhost)
0.0.0.0 - This network
255.255.255.255 - Broadcast
169.254.x.x - Link-local (APIPA)
224.0.0.0/4 - Multicast
```

### 2.2 Routing Fundamentals

**Routing Process:**
```
Packet arrives at router:
  1. Extract destination IP
  2. Consult routing table for matching route
  3. Find longest prefix match (most specific route)
  4. Get outgoing interface and next-hop gateway
  5. Encapsulate in Ethernet frame with next-hop MAC
  6. Forward to next-hop
```

**Routing Table Example:**
```
Destination       Netmask           Gateway         Interface
─────────────────────────────────────────────────────────────
192.168.1.0      255.255.255.0     192.168.1.1     eth0
192.168.2.0      255.255.255.0     192.168.1.254   eth0
0.0.0.0          0.0.0.0           192.168.1.1     eth0 (default)

Packet to 192.168.2.50:
  Check 192.168.2.50 & 255.255.255.0 = 192.168.2.0 ✓
  Gateway: 192.168.1.254
  Send to 192.168.1.254
```

**Longest Prefix Match Algorithm:**
```
Destination: 10.1.2.3

Route 1: 10.0.0.0/8 (matches first 8 bits)
Route 2: 10.1.0.0/16 (matches first 16 bits) ← CHOSE THIS
Route 3: 10.1.2.0/24 (matches first 24 bits) ← OR THIS (most specific)

Most specific wins (longest prefix = /24)
```

**Distance Vector vs. Link State Routing:**

**Distance Vector (RIP):**
- Each router knows: Distance (hop count) to each destination
- Routers exchange routing tables with neighbors
- "Route via neighbor X to reach destination D"
- Problem: Slow convergence, count-to-infinity

**Link State (OSPF):**
- Each router knows: Complete topology of network
- Routers flood link information to all routers
- Each router calculates shortest path (Dijkstra's algorithm)
- Advantage: Fast convergence, accurate routing

---

## 3. DNS: Domain Name System Architecture

### 3.1 DNS Hierarchy and Resolution

**DNS Hierarchy:**
```
Root Name Servers (13 worldwide)
  ↓
Top-Level Domain (TLD) Servers (.com, .org, .edu)
  ↓
Authoritative Name Servers (owned by domain owner)
```

**DNS Resolution Process (Recursive):**
```
Client: Resolve example.com?
  ↓ (query)
Recursive Resolver (ISP DNS): Not in cache, ask root
  ↓ (query)
Root Server: Try TLD server for .com
  ↓ (referral to TLD)
TLD Server: Try authoritative for example.com
  ↓ (referral to authoritative)
Authoritative Server: example.com = 93.184.216.34
  ↓ (answer)
Recursive Resolver: Cache result, return to client
  ↓ (answer)
Client: example.com → 93.184.216.34
```

**Query Types:**
```
A Record: IPv4 address (example.com → 93.184.216.34)
AAAA Record: IPv6 address (example.com → 2606:2800:220:1...)
MX Record: Mail exchange (example.com → mail.example.com priority 10)
CNAME Record: Canonical name (alias → example.com)
NS Record: Name server (example.com authority → ns1.example.com)
TXT Record: Text data (used for SPF, DKIM, DMARC)
SOA Record: Start of Authority (zone metadata)
```

**DNS Caching:**
```
TTL (Time-To-Live): How long can record be cached

High TTL (86400 seconds = 1 day):
  ✓ Fewer queries to DNS servers
  ✗ Slower to propagate changes (1 day delay)

Low TTL (300 seconds = 5 minutes):
  ✓ Quick propagation of changes
  ✗ More DNS queries (higher load)

Typical: 3600 seconds (1 hour)
```

**DNS Query Examples:**
```
$ dig example.com
; <<>> DiG 9.10.6 <<>> example.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 3, ADDITIONAL: 0

;; QUESTION SECTION:
;example.com.                   IN      A

;; ANSWER SECTION:
example.com.            3600    IN      A       93.184.216.34

;; Query time: 53 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
```

---

## 4. Transport Layer Protocols (TCP and UDP)

### 4.1 UDP (User Datagram Protocol)

**Characteristics:**
- Connectionless (no handshake)
- Unreliable (packets may be lost)
- Low latency (no flow control or retransmission)
- Minimal overhead (8-byte header)
- Stateless (no connection tracking)

**UDP Header:**
```
Source Port (16 bits)
Destination Port (16 bits)
Length (16 bits) = header + data
Checksum (16 bits) = optional error detection
Data (variable)
```

**Use Cases:**
- DNS queries (need quick response, low overhead)
- Video streaming (occasional loss acceptable)
- Online gaming (fast updates > accuracy)
- VoIP (real-time, loss acceptable)
- DHCP (simple discovery)

**UDP Communication:**
```
Client (port 50000):           Server (port 53)
  send("8.8.8.8") ─────────→
                   ←───────── recv("93.184.216.34")
  send("google.com") ────────→
                   ←───────── recv("142.251.32.142")
  (connection-less, can send multiple queries without setup)
```

### 4.2 TCP (Transmission Control Protocol)

**Characteristics:**
- Connection-oriented (three-way handshake)
- Reliable (guaranteed delivery, in-order)
- Flow control (prevent overwhelming receiver)
- Congestion control (respect network capacity)
- Overhead: Higher due to sequencing/acknowledgments

**Three-Way Handshake:**
```
Client                                    Server
  │                                         │
  │─────── SYN (seq=1000) ──────────────→  │
  │                                    ACK received, send SYN-ACK
  │  ←─── SYN-ACK (seq=5000, ack=1001) ─── │
  │  ACK received, connection established, send ACK
  │─────── ACK (seq=1001, ack=5001) ──────→ │
  │                                    Connection established
  │ ═══════════════════════════════════════ │
  │      Connected, can exchange data      │
  │ ═══════════════════════════════════════ │
```

**TCP Header:**
```
Source Port (16 bits)
Destination Port (16 bits)
Sequence Number (32 bits) - Identifies data order
Acknowledgment Number (32 bits) - Confirms received data
Flags: SYN, ACK, FIN, RST, PSH, URG
Window Size (16 bits) - Flow control
Checksum (16 bits)
Urgent Pointer (16 bits)
Options (variable)
```

**Sequence Number and Acknowledgment:**
```
Client sends 100 bytes with seq=1000:
  Bytes 1000-1099 transmitted
  
Server receives and sends:
  ACK with ack=1100 (next expected sequence)
  
Client sends next 100 bytes with seq=1100:
  Bytes 1100-1199 transmitted
  
If server detects missing byte 1050-1099:
  Still sends ack=1050 (last received in order)
  
Client retransmits bytes 1050-1099
```

**Connection Termination (Four-Way Handshake):**
```
Client                                    Server
  │                                         │
  │─────── FIN (seq=5000) ────────────────→ │
  │                                    Send ACK
  │ ←─── ACK (ack=5001) ─────────────────── │
  │                              Close connection, send FIN
  │ ←─── FIN (seq=7000) ─────────────────── │
  │  Send ACK
  │─────── ACK (ack=7001) ────────────────→ │
  │                                    Connection closed
```

**TCP vs. UDP Summary:**
```
                TCP                UDP
────────────────────────────────────────────
Connection      Oriented           Connectionless
Reliability     Guaranteed          Best effort
Ordering        In-order            No guarantee
Speed           Slower (overhead)   Faster
Overhead        High (20+ bytes)    Low (8 bytes)
Flow Control    Yes                 No
Congestion      Yes                 No
Use             HTTP, SMTP, SSH     DNS, gaming, streaming
```

---

## 5. Application Layer Protocols (HTTP/HTTPS)

### 5.1 HTTP/1.1 Fundamentals

**Request-Response Model:**
```
Client sends HTTP Request:
  Method (GET, POST, PUT, DELETE, HEAD, OPTIONS)
  URI (/index.html)
  HTTP Version (1.1)
  Headers (Host, User-Agent, Accept, etc.)
  Body (optional, for POST/PUT)

Server sends HTTP Response:
  Status Code (200, 404, 500, etc.)
  Reason Phrase (OK, Not Found, Internal Server Error)
  Headers (Content-Type, Content-Length, Server, etc.)
  Body (HTML, JSON, binary data, etc.)
```

**HTTP Request Example:**
```
GET /index.html HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: text/html,application/xhtml+xml
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate
Connection: keep-alive
```

**HTTP Response Example:**
```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 1234
Server: nginx/1.18.0
Cache-Control: public, max-age=3600
ETag: "abc123def456"
Set-Cookie: sessionid=abc123; Path=/; HttpOnly

<!DOCTYPE html>
<html>
  <head><title>Example Domain</title></head>
  <body>...</body>
</html>
```

**Status Codes:**
```
1xx: Informational (100 Continue, 101 Switching Protocols)
2xx: Success (200 OK, 201 Created, 204 No Content)
3xx: Redirection (301 Moved Permanently, 302 Found, 304 Not Modified)
4xx: Client Error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found)
5xx: Server Error (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable)
```

**HTTP Methods:**
```
GET: Retrieve resource (safe, idempotent)
POST: Create resource (unsafe, not idempotent)
PUT: Update resource (idempotent)
DELETE: Remove resource (idempotent)
PATCH: Partial update (not idempotent)
HEAD: Like GET but without body (safe, idempotent)
OPTIONS: Describe communication options (safe, idempotent)
```

**HTTP/1.1 Keep-Alive:**
```
Without Keep-Alive:
  Request 1 → Response 1 → Close connection
  Request 2 → Open connection → Response 2 → Close connection
  Request 3 → Open connection → Response 3 → Close connection
  (3 connections, 3 handshakes = overhead)

With Keep-Alive (default in HTTP/1.1):
  Request 1 → Response 1 → Keep connection open
  Request 2 → Response 2 → Keep connection open
  Request 3 → Response 3 → Keep connection open
  (1 connection, 1 handshake = efficient)
  
  Header: Connection: keep-alive
  Timeout: Connection closes after idle time
```

### 5.2 HTTPS (HTTP Secure)

**HTTPS = HTTP + TLS (Transport Layer Security)**

**TLS Handshake:**
```
Client                                          Server
  │                                               │
  │─ ClientHello (TLS version, ciphers) ────────→│
  │                                        ServerHello + Certificate
  │←─────────────────────────────────────────────│
  │                                    ServerKeyExchange + ServerHelloDone
  │
  │─ ClientKeyExchange ──────────────────────────→│
  │   (ephemeral key + MAC verification)    │
  │─ ChangeCipherSpec ─────────────────────────→ │
  │ Finished                              ServerKeyExchange + Finished
  │←─────────────────────────────────────────────│
  │                                    ChangeCipherSpec
  │ ═════════════════════════════════════════════
  │    Encrypted connection established
```

**Certificate Chain Validation:**
```
Server sends certificate chain:
  1. Leaf Certificate (example.com)
     └─ Signed by Intermediate CA
        └─ Intermediate Certificate
           └─ Signed by Root CA
              └─ Root Certificate (trusted by OS)

Client validates:
  1. Check leaf certificate is for example.com
  2. Verify signature (Intermediate signed with its private key)
  3. Verify Intermediate's signature
  4. Verify Root is in trust store
  ✓ All valid → Trust the connection
  ✗ Any invalid → Reject connection
```

**Cipher Suites:**
```
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
  │        │    │       │         │
  │        │    │       │         └─ Hash algorithm (SHA384)
  │        │    │       └─────────── Encryption (AES-256-GCM)
  │        │    └─────────────────── Key exchange (RSA)
  │        └─────────────────────── Key agreement (ECDHE - Perfect Forward Secrecy)
  └──────────────────────────────── Protocol (TLS)

Modern recommendation:
  - Use ECDHE (Perfect Forward Secrecy)
  - Use AEAD (Authenticated Encryption: GCM, ChaCha20-Poly1305)
  - Disable: RC4, DES, 3DES, MD5
```

**Perfect Forward Secrecy (PFS):**
```
With Static RSA:
  Server long-term key compromised → All past sessions decryptable

With ECDHE (Ephemeral):
  For each connection:
    1. Server generates ephemeral key pair
    2. Client and server agree on ephemeral shared secret
    3. Connection encrypted with ephemeral secret
    4. Ephemeral key discarded
    
  If server long-term key compromised → Past sessions still safe
  (ephemeral keys were destroyed)
```

---

## 6. Network Security Fundamentals

### 6.1 Common Attack Vectors

**Man-in-the-Middle (MITM):**
```
Attacker on same network:
  
Normal flow:
  Client ──→ Server

MITM:
  Client ──→ Attacker ──→ Server
       ↑                    │
       └────────────────────┘
  
Attacker sees all traffic, can modify packets.
Prevention: HTTPS (TLS encryption + authentication)
```

**Packet Sniffing:**
```
Attacker with packet capture tool (tcpdump, Wireshark):
  
HTTP traffic (unencrypted):
  GET /login?user=admin&pass=secret123

HTTPS traffic (encrypted):
  Encrypted blob (appears as random data)
  TLS provides confidentiality
```

**DNS Spoofing:**
```
Normal:
  Client: What is example.com?
  Server: 93.184.216.34

Spoofing:
  Attacker intercepts/responds first:
  Client: What is example.com?
  Attacker (pretending to be DNS): 192.168.1.100 (attacker's IP!)
  Client visits attacker's site thinking it's example.com
  
Prevention: DNSSEC (cryptographically sign DNS responses)
```

### 6.2 TLS and Certificate Security

**Certificate Pinning:**
```
During development:
  App developer pins (stores) server's certificate public key
  
When connecting:
  1. Normal TLS handshake
  2. Extract server's certificate public key
  3. Compare with pinned key
  4. Only continue if keys match
  
Prevents: Compromised intermediate CAs issuing fake certificates
Trade-off: Hard to rotate certificates (app update required)
```

**Hashing and Signatures:**
```
Server has:
  - Private key (secret)
  - Public key (distributed in certificate)

Server signs certificate:
  hash = SHA256(certificate_data)
  signature = RSA_sign(hash, private_key)
  
Client verifies:
  hash = SHA256(certificate_data)
  recovered_hash = RSA_verify(signature, public_key)
  if hash == recovered_hash:
    ✓ Certificate authentic (signed by private key owner)
```

---

## 7. Content Delivery and Load Balancing

### 7.1 Content Delivery Networks (CDNs)

**Problem:** User in Sydney fetching content from New York server
- Latency: ~200ms (speed of light over fiber)
- Congestion: Overseas links expensive and congested
- User experience: Slow page loads

**CDN Solution:**
```
Origin Server (New York):
  └─ CDN Cache Layer (globally distributed)
     ├─ Edge Server (Sydney)
     ├─ Edge Server (London)
     ├─ Edge Server (Tokyo)
     └─ Edge Server (São Paulo)

User in Sydney:
  1. Request goes to nearest edge server (Sydney)
  2. If cache hit: Respond immediately (~5ms)
  3. If cache miss: Fetch from origin, cache locally

Result: 200ms → 5ms latency reduction
```

**CDN Architecture:**
```
User request to example.com:
  1. DNS query for example.com
     ↓
  2. CNAME alias: example.com → cdn.example.com.cdnprovider.com
     ↓
  3. CDN's DNS returns IP of nearest edge server
     ↓
  4. User connects to edge server
     ↓
  5. Edge server has content cached (or fetches from origin)
     ↓
  6. User receives content from fast, nearby server
```

**Cache Headers for CDN:**
```
Cache-Control: public, max-age=3600
  - public: Any cache can store
  - max-age=3600: Valid for 3600 seconds (1 hour)
  
Cache-Control: private, max-age=0
  - private: Only browser cache (not CDN)
  - max-age=0: Don't cache (always fetch fresh)
  
ETag: "abc123def456"
  - Entity tag for cache validation
  - If-None-Match: "abc123def456"
    Server responds 304 Not Modified if unchanged
```

### 7.2 Load Balancing

**Problem:** Single server can't handle millions of requests

**Load Balancer:**
```
                    ┌─ Server 1 (port 8001)
Client 1 ─→         │
Client 2 ─→ Load ───┼─ Server 2 (port 8002)
Client 3 ─→ Balancer│
Client 4 ─→         │
            (public  └─ Server 3 (port 8003)
             IP)
             
Load balancer distributes requests across servers
```

**Load Balancing Algorithms:**

**Round Robin:**
```
Client 1 → Server 1
Client 2 → Server 2
Client 3 → Server 3
Client 4 → Server 1 (cycle repeats)
```

**Least Connections:**
```
Server 1: 10 active connections
Server 2: 5 active connections ← Route to this
Server 3: 8 active connections
```

**Weighted Round Robin:**
```
Server 1: High-power machine, weight=3
Server 2: Low-power machine, weight=1

Distribution: Server1:Server2:Server3 = 3:1
```

**IP Hash:**
```
hash(client_IP) % num_servers = server_index

Same client always routes to same server (sticky)
Useful for session state stored on servers
```

**Sticky Sessions:**
```
Client A: Route to Server 1 (first request)
Client A: Route to Server 1 (subsequent requests)
Client B: Route to Server 2 (first request)
Client B: Route to Server 2 (subsequent requests)

Implementation:
  - Cookie with server ID
  - Load balancer checks cookie, routes accordingly
  - If server 1 down, must re-route and lose session
  
Better approach: Store session in shared cache (Redis, Memcached)
```

**Health Checks:**
```
Load balancer periodically checks each server:
  
GET /health HTTP/1.1
Host: server1.internal

Response:
  200 OK {"status": "healthy"}
  
If server doesn't respond or returns error:
  Mark as unhealthy
  Stop routing requests to it
  
Once healthy again:
  Resume routing
```

---

## 8. Production Networking and Performance

### 8.1 Latency and Bandwidth

**Latency vs. Bandwidth:**

$$\text{Latency} = \text{Time for packet to reach destination}$$
$$\text{Bandwidth} = \text{Maximum data rate (bits/second)}$$

**Analogy:**
- Latency = Time to travel from city A to city B
- Bandwidth = Width of the highway (how many cars simultaneously)

**Typical Latencies:**
```
L1 Cache hit: 1-2 ns
L2 Cache hit: 3-4 ns
L3 Cache hit: 10-20 ns
RAM access: 60-100 ns
Disk seek: 1-10 ms
Network round-trip (local): 1-10 ms
Network round-trip (intercontinental): 100-200 ms

Rule of thumb:
  Each 10x slower = 10x more latency
```

**Latency Composition for HTTP Request:**
$$\text{Total Latency} = \text{DNS lookup} + \text{TCP handshake} + \text{TLS handshake} + \text{HTTP request/response}$$

```
Example:
  DNS lookup: 50 ms (resolver query)
  TCP handshake: 30 ms (SYN, SYN-ACK, ACK)
  TLS handshake: 60 ms (ClientHello, ServerHello, etc.)
  HTTP request/response: 50 ms (network round-trip + processing)
  ──────────────────
  Total: ~190 ms
```

**Bandwidth Calculation:**
```
100 Mbps internet connection:
  100 bits per second = 12.5 MB per second
  
Download 1 GB file:
  1 GB / 12.5 MB/s = 80 seconds

Note: Real-world slower due to:
  - Protocol overhead (IP, TCP headers)
  - Network congestion
  - Hardware limitations
  - Realistic speed: 70-80% of theoretical
```

### 8.2 Network Optimization Techniques

**TCP Optimization:**

**Window Scaling:**
```
Default TCP window: 65,535 bytes (64 KB)
On high-latency links: Small window = wasted capacity

Example: 200 ms latency, 100 Mbps:
  Bytes in flight = 100 Mbps × 0.2 s = 2.5 MB
  But window only 64 KB → Severely underutilized
  
Solution: Window scaling option
  Window size: 1 MB (allows better utilization)
```

**Congestion Control (CUBIC):**
```
When packet loss detected:
  1. Reduce sending rate (congestion avoidance)
  2. Gradually increase sending rate (recovery)
  3. Repeat
  
Modern: CUBIC algorithm (replaces older Reno)
  - Faster recovery
  - Better fairness
  - Handles high-BDP networks
```

**DNS Optimization:**

**DNS Prefetching:**
```
HTML hint:
  <link rel="dns-prefetch" href="https://cdn.example.com">
  
Browser starts DNS lookup early, saves 50-100 ms on first request
```

**HTTP/2 and HTTP/3:**

**HTTP/2 Multiplexing:**
```
HTTP/1.1 (with keep-alive):
  Request 1 ──→ Response 1 ──→ Request 2 ──→ Response 2
  (sequential)

HTTP/2 Multiplexing:
  Request 1 ──→ Response 1
  Request 2 ──→ Response 2
  (concurrent over single connection)
  
Benefit: Avoid head-of-line blocking
Cost: Single connection = single congestion window
Trade-off: Better for high-latency links
```

**HTTP/3 (QUIC):**
```
HTTP/2 over TCP:
  TCP reliability ensures ordering
  If packet lost → entire connection delays
  
HTTP/3 over QUIC:
  QUIC multiplexes streams
  If stream 1 packet lost → only stream 1 affected
  Other streams continue
  
Benefit: Better for lossy networks (4G/5G mobile)
Cost: More complex implementation
```

### 8.3 Performance Monitoring

**Key Metrics:**
```
Latency (p50, p95, p99):
  p50: 50th percentile (median)
  p99: 99th percentile (tail latency)
  
  p50 = 10ms (typical user)
  p99 = 500ms (unhappy users)
  
Throughput:
  Requests per second (RPS)
  Kilobytes per second (KB/s)
  
Error Rate:
  Failed requests / Total requests
  Acceptable: < 0.1%
  
Bandwidth Utilization:
  Used / Available
  Keep below 70% (headroom for spikes)
```

**Monitoring Tools:**
```
Packet Analysis: tcpdump, Wireshark
Network Paths: traceroute, mtr
DNS Resolution: nslookup, dig
Connectivity: ping, nc (netcat)
Throughput: iperf, speedtest
```

---

## Key Takeaways

1. **OSI Model**: 7 layers (Physical → Application) provide conceptual framework. TCP/IP is practical 4-layer model.

2. **IP Addressing**: IPv4 32-bit addresses, CIDR notation for subnetting. Routing via longest prefix match through routing tables.

3. **DNS**: Hierarchical system (Root → TLD → Authoritative) resolves domains to IPs. Caching critical for performance.

4. **Transport Layer**:
   - **UDP**: Unreliable, fast, connectionless (DNS, gaming)
   - **TCP**: Reliable, slower, connection-oriented (HTTP, email)

5. **HTTP/1.1**: Request-response model, keep-alive for efficiency, status codes for results.

6. **HTTPS**: HTTP + TLS encryption. Handshake establishes encryption, certificates provide authentication.

7. **Security**:
   - MITM prevented by HTTPS
   - DNS spoofing prevented by DNSSEC
   - Certificate pinning for extra security

8. **CDNs**: Cache content globally near users, dramatically reduce latency (200ms → 5ms).

9. **Load Balancing**: Distribute requests across servers (round-robin, least connections, IP hash, sticky sessions).

10. **Performance**: 
    - Latency composition: DNS + TCP + TLS + HTTP
    - Bandwidth vs. latency (both matter)
    - TCP window scaling, congestion control
    - HTTP/2 multiplexing, HTTP/3 QUIC

11. **Monitoring**: Track p50/p99 latency, throughput, error rates, bandwidth utilization.

12. **Modern Protocols**: HTTP/2 for web, HTTP/3 for mobile, TLS 1.3 for security.
