# Networking Fundamentals
### A Complete Reference for DevOps Engineers

> **How to use this document:** Read it top to bottom the first time. Every section builds on the previous. By the end, when someone asks "what happens when you type google.com in a browser" — you will be able to explain every single step in detail. That's the benchmark.

---

## Table of Contents

1. [What is a Network?](#1-what-is-a-network)
2. [The OSI Model — 7 Layers](#2-the-osi-model--7-layers)
3. [The TCP/IP Model — The Real World Version](#3-the-tcpip-model--the-real-world-version)
4. [IP Addressing](#4-ip-addressing)
5. [Subnetting & CIDR Notation](#5-subnetting--cidr-notation)
6. [MAC Addresses & ARP](#6-mac-addresses--arp)
7. [Routing & How Packets Travel](#7-routing--how-packets-travel)
8. [TCP — Transmission Control Protocol](#8-tcp--transmission-control-protocol)
9. [UDP — User Datagram Protocol](#9-udp--user-datagram-protocol)
10. [TCP vs UDP — When to Use Which](#10-tcp-vs-udp--when-to-use-which)
11. [Ports & Sockets](#11-ports--sockets)
12. [DNS — Domain Name System](#12-dns--domain-name-system)
13. [HTTP & HTTPS](#13-http--https)
14. [TLS/SSL — How Encryption Works](#14-tlsssl--how-encryption-works)
15. [Load Balancing](#15-load-balancing)
16. [Firewalls & iptables](#16-firewalls--iptables)
17. [NAT — Network Address Translation](#17-nat--network-address-translation)
18. [Common Protocols You Must Know](#18-common-protocols-you-must-know)
19. [Network Interfaces & the Linux Network Stack](#19-network-interfaces--the-linux-network-stack)
20. [Virtual Networking — VLANs, VPNs, Tunnels](#20-virtual-networking--vlans-vpns-tunnels)
21. [Cloud Networking Concepts](#21-cloud-networking-concepts)
22. [Network Troubleshooting — Tools & Methodology](#22-network-troubleshooting--tools--methodology)
23. [What Happens When You Type google.com](#23-what-happens-when-you-type-googlecom)
24. [Quick Reference Cheat Sheet](#24-quick-reference-cheat-sheet)

---

## 1. What is a Network?

A **network** is two or more devices connected together in a way that allows them to exchange data.

At its most fundamental level, networking is about answering three questions:
1. **Addressing** — How do we identify who we're talking to?
2. **Routing** — How do we get data from here to there?
3. **Reliability** — How do we ensure data arrives correctly?

### Types of Networks

| Type | Scope | Example |
|---|---|---|
| **PAN** (Personal Area Network) | A few meters | Bluetooth devices, USB |
| **LAN** (Local Area Network) | A building or campus | Your home Wi-Fi, office network |
| **MAN** (Metropolitan Area Network) | A city | ISP connecting neighbourhoods |
| **WAN** (Wide Area Network) | Countries / worldwide | The Internet |
| **VPN** (Virtual Private Network) | Any size — over the internet | Remote employee accessing office resources |

### The Internet vs a Network

- A **network** is any group of connected devices
- The **Internet** is the global network of networks — millions of LANs and WANs connected together using shared protocols (TCP/IP)
- The Internet has no central owner — it's a cooperative system of interconnected networks run by ISPs, companies, and governments

### Protocols

A **protocol** is an agreed-upon set of rules for how communication happens. Without protocols, devices couldn't understand each other.

Think of it as language — two people must speak the same language to communicate. HTTP, TCP, DNS — these are all protocols. Each defines exactly:
- How the communication starts
- What format messages take
- How errors are handled
- How the communication ends

---

## 2. The OSI Model — 7 Layers

The **OSI Model (Open Systems Interconnection)** is a conceptual framework that describes how network communication works by dividing it into 7 distinct layers.

It was created to standardize networking so different vendors' hardware and software can work together.

> **Important:** The OSI model is a conceptual model, not a strict implementation. Real protocols don't always map perfectly to exactly one layer. It's a mental framework for reasoning about networking.

### The 7 Layers

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 │ APPLICATION  │ HTTP, HTTPS, DNS, SMTP, FTP, SSH      │
├──────────┼──────────────┼─────────────────────────────────────  │
│  Layer 6 │ PRESENTATION │ TLS/SSL, compression, encoding        │
├──────────┼──────────────┼─────────────────────────────────────  │
│  Layer 5 │ SESSION      │ Session management, authentication    │
├──────────┼──────────────┼─────────────────────────────────────  │
│  Layer 4 │ TRANSPORT    │ TCP, UDP — ports, reliability         │
├──────────┼──────────────┼─────────────────────────────────────  │
│  Layer 3 │ NETWORK      │ IP, ICMP, routing — logical addresses │
├──────────┼──────────────┼─────────────────────────────────────  │
│  Layer 2 │ DATA LINK    │ Ethernet, Wi-Fi — MAC addresses       │
├──────────┼──────────────┼─────────────────────────────────────  │
│  Layer 1 │ PHYSICAL     │ Cables, signals, bits on the wire     │
└─────────────────────────────────────────────────────────────────┘
```

### Layer by Layer — What Each One Does

#### Layer 1 — Physical
- The actual physical medium: copper cables, fiber optic, radio waves (Wi-Fi), light pulses
- Deals with **bits** (0s and 1s) being transmitted as electrical signals, light, or radio
- Devices: Network cables (Cat5e, Cat6), fiber, hubs, repeaters, modems
- Problems here: broken cable, bad connector, interference, signal degradation

#### Layer 2 — Data Link
- Transfers data between two **directly connected** devices on the same local network
- Deals in **frames** (structured groups of bits)
- Uses **MAC addresses** to identify devices on the local network
- Handles error detection for each hop (not end-to-end)
- Protocols: Ethernet (IEEE 802.3), Wi-Fi (IEEE 802.11)
- Devices: **Switches** (operate at Layer 2 — forward frames based on MAC addresses)

#### Layer 3 — Network
- Handles **logical addressing** and routing across multiple networks
- Deals in **packets**
- Uses **IP addresses** to identify source and destination (which can be across the world)
- Decides the best path to get a packet from source to destination
- Protocols: **IP (IPv4, IPv6)**, ICMP (ping uses this)
- Devices: **Routers** (operate at Layer 3 — forward packets based on IP addresses)

#### Layer 4 — Transport
- Provides **end-to-end communication** between specific applications on source and destination hosts
- Deals in **segments** (TCP) or **datagrams** (UDP)
- Uses **port numbers** to identify which application data belongs to
- **TCP** provides reliability — guaranteed delivery, ordering, error correction
- **UDP** is lightweight — no guarantees, but fast
- Protocols: **TCP, UDP**

#### Layer 5 — Session
- Manages **sessions** — establishes, maintains, and terminates connections between applications
- Handles **synchronization** and **checkpointing** (resuming interrupted transfers)
- In practice, mostly absorbed into Layer 4/7 in real implementations
- Examples: NetBIOS, RPC, SQL sessions

#### Layer 6 — Presentation
- Translates data between the network format and the application format
- Handles **encryption/decryption** (TLS), **compression**, **encoding** (ASCII, Unicode, JPEG)
- In practice, mostly absorbed into the application layer in modern stacks

#### Layer 7 — Application
- The layer closest to the user — provides network services directly to applications
- Defines **how applications communicate** over the network
- Protocols: **HTTP, HTTPS, DNS, SMTP, FTP, SSH, DHCP**

### The Mnemonic

To remember the order (top to bottom):

> **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing

**A**pplication → **P**resentation → **S**ession → **T**ransport → **N**etwork → **D**ata Link → **P**hysical

### Encapsulation — How Data Travels Down the Stack

When you send data, it goes down the OSI stack on the sender, and up the stack on the receiver:

```
Sender                                    Receiver
──────                                    ────────
App data                                  App data ← your browser
  ↓ + TCP header (ports, seq#)              ↑ TCP header removed
  ↓ + IP header (src/dst IP)                ↑ IP header removed
  ↓ + Ethernet frame (src/dst MAC)          ↑ Ethernet header removed
  ↓                                         ↑
  └──────── bits on the wire ───────────────┘
```

Each layer **wraps** the data with its own header. The receiver **unwraps** each header as it moves up the stack. This wrapping is called **encapsulation**.

### Why DevOps Engineers Care About OSI

When debugging network issues, the OSI model tells you **where to look**:

| Symptom | Likely Layer | Tools to check |
|---|---|---|
| Can't ping at all | Layer 1 or 2 | Check cables, `ip link` |
| Can ping IP but not hostname | Layer 7 (DNS) | `nslookup`, `dig` |
| Can connect but TLS fails | Layer 6 | `openssl s_client` |
| Connection refused | Layer 4 | Check if port is open, `ss -tlnp` |
| Can connect but app returns error | Layer 7 | Application logs, `curl -v` |

---

## 3. The TCP/IP Model — The Real World Version

The **TCP/IP model** (also called the Internet model) is what the real internet actually uses. It's a simpler, 4-layer model that combines the OSI layers more practically.

```
OSI Model               TCP/IP Model         Examples
─────────               ────────────         ────────
Application  ┐
Presentation ├─────────→ Application    ─── HTTP, DNS, SSH, SMTP
Session      ┘
Transport    ──────────→ Transport      ─── TCP, UDP
Network      ──────────→ Internet       ─── IP, ICMP, ARP
Data Link    ┐
Physical     ├─────────→ Network Access ─── Ethernet, Wi-Fi
```

In practice, when engineers say "Layer 3" or "Layer 7" they're still referring to OSI layer numbers — even though we use TCP/IP. The OSI model is the universal language for discussing where in the stack something operates.

---

## 4. IP Addressing

An **IP address** is the logical address that uniquely identifies a device on a network. Like a mailing address — it tells the network where to deliver data.

### IPv4

IPv4 addresses are **32 bits** written as 4 decimal numbers (octets), each 0–255, separated by dots.

```
192  .  168  .   1   .  100
 ↑        ↑       ↑      ↑
8 bits  8 bits  8 bits  8 bits  =  32 bits total
```

Total possible IPv4 addresses: 2³² = ~4.3 billion (we've run out — hence IPv6)

### IPv4 Address Classes (Historical)

Originally, the internet divided addresses into classes:

| Class | Range | Default Subnet | Purpose |
|---|---|---|---|
| **A** | 1.0.0.0 – 126.255.255.255 | /8 (16M hosts) | Large organizations |
| **B** | 128.0.0.0 – 191.255.255.255 | /16 (65K hosts) | Medium organizations |
| **C** | 192.0.0.0 – 223.255.255.255 | /24 (254 hosts) | Small networks |
| **D** | 224.0.0.0 – 239.255.255.255 | — | Multicast |
| **E** | 240.0.0.0 – 255.255.255.255 | — | Reserved/experimental |

Classful addressing is obsolete today — replaced by CIDR. But you'll still hear "Class C network" in casual conversation.

### Private vs Public IP Addresses

**Private addresses** are reserved for use inside private networks (your home, office, cloud VPC). They are **not routable on the public internet** — routers on the internet will not forward packets destined for private IPs.

| Range | CIDR | Common Use |
|---|---|---|
| `10.0.0.0 – 10.255.255.255` | 10.0.0.0/8 | Large enterprise, AWS VPCs |
| `172.16.0.0 – 172.31.255.255` | 172.16.0.0/12 | Docker default network |
| `192.168.0.0 – 192.168.255.255` | 192.168.0.0/16 | Home routers, small offices |

**Public addresses** are globally unique and routable on the internet. Your ISP assigns you one (or more).

### Special IP Addresses

| Address | Meaning |
|---|---|
| `127.0.0.1` | Loopback — "this machine itself" — traffic never leaves the host |
| `0.0.0.0` | "All interfaces" — when a server listens on 0.0.0.0, it accepts connections on all its network interfaces |
| `255.255.255.255` | Limited broadcast — send to all hosts on current network |
| `169.254.x.x` | Link-local — auto-assigned when DHCP fails (APIPA) |

```bash
# Check your IP addresses
ip addr show                  # All interfaces and their IP addresses
ip addr show eth0             # Specific interface
hostname -I                   # Just the IP addresses
curl ifconfig.me              # Your public IP address
```

### IPv6

IPv6 addresses are **128 bits** written as 8 groups of 4 hex digits separated by colons:

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
```

Rules for shortening:
- Leading zeros in a group can be omitted: `0db8` → `db8`
- One group of consecutive all-zero groups can be replaced by `::`: `2001:db8::8a2e:370:7334`

Special IPv6 addresses:
- `::1` → loopback (equivalent to 127.0.0.1)
- `fe80::/10` → link-local addresses
- `::` → all zeros (equivalent to 0.0.0.0)

IPv6 solves IPv4 exhaustion: 2¹²⁸ = 340 undecillion addresses — effectively unlimited.

```bash
ip -6 addr show                # IPv6 addresses
ping6 ::1                      # Ping IPv6 loopback
curl -6 https://ipv6.google.com  # Test IPv6 connectivity
```

---

## 5. Subnetting & CIDR Notation

### What is a Subnet?

A **subnet (subnetwork)** divides a larger network into smaller, logical segments. It's how you organize and isolate parts of a network.

Benefits:
- **Security:** Isolate sensitive servers in their own subnet
- **Performance:** Reduce broadcast traffic (only reaches devices in the same subnet)
- **Organization:** Group related systems logically (web servers, databases, management)

### The Subnet Mask

Every IP address works together with a **subnet mask** that defines which part of the IP address identifies the **network** and which part identifies the **host** within that network.

```
IP Address:   192.168.1.100
Subnet Mask:  255.255.255.0

Binary:
IP:     11000000.10101000.00000001.01100100
Mask:   11111111.11111111.11111111.00000000
        ──────────────────────────  ────────
              Network part           Host part
             (192.168.1)              (.100)
```

The `1`s in the mask identify the network portion. The `0`s identify the host portion.

All devices with the same network portion are on the same subnet and can communicate directly without a router.

### CIDR Notation

**CIDR (Classless Inter-Domain Routing)** is the modern way to express IP addresses and their subnet masks. Instead of writing out the full subnet mask, you write a `/` followed by the number of `1` bits in the mask.

```
192.168.1.0/24
             ↑
             24 bits are the network part
             (32 - 24 = 8 bits for hosts → 2⁸ = 256 addresses, 254 usable)
```

Common CIDR values:

| CIDR | Subnet Mask | # of Hosts | Common Use |
|---|---|---|---|
| `/8` | 255.0.0.0 | 16,777,214 | Large organization (Class A) |
| `/16` | 255.255.0.0 | 65,534 | Cloud VPC, enterprise |
| `/24` | 255.255.255.0 | 254 | Typical office subnet |
| `/25` | 255.255.255.128 | 126 | Half of a /24 |
| `/28` | 255.255.255.240 | 14 | Small subnet, AWS security group |
| `/30` | 255.255.255.252 | 2 | Point-to-point links between routers |
| `/32` | 255.255.255.255 | 1 | A single host (specific route, firewall rule) |

### Key Addresses in a Subnet

For any subnet, there are reserved addresses:

```
Subnet: 192.168.1.0/24

Network address:   192.168.1.0   → Identifies the subnet (not assignable)
First usable:      192.168.1.1   → Usually the gateway/router
...
Last usable:       192.168.1.254
Broadcast address: 192.168.1.255 → Sends to all hosts in subnet (not assignable)
```

**Total addresses** = 2^(host bits)
**Usable hosts** = 2^(host bits) - 2 (subtract network + broadcast)

### Subnetting Example

You're given `10.0.0.0/8` and need to create 4 equal subnets.

- Original: 10.0.0.0/8 — 8 bits for network, 24 bits for hosts
- Need 4 subnets → need 2 extra bits for subnet identification (2² = 4)
- New prefix: /8 + 2 = /10

The 4 subnets:
```
10.0.0.0/10     → 10.0.0.1   to 10.63.255.254
10.64.0.0/10    → 10.64.0.1  to 10.127.255.254
10.128.0.0/10   → 10.128.0.1 to 10.191.255.254
10.192.0.0/10   → 10.192.0.1 to 10.255.255.254
```

### Checking Subnets in Linux

```bash
ip route show                    # Show routing table — shows your subnets
ip addr show eth0                # See IP and CIDR of an interface
ipcalc 192.168.1.0/24            # Calculate subnet details (may need: apt install ipcalc)
```

---

## 6. MAC Addresses & ARP

### MAC Addresses

A **MAC address (Media Access Control)** is a hardware-level address burned into every network interface card (NIC). It's used for communication within a local network segment.

```
AA:BB:CC:DD:EE:FF
──────────  ──────
   OUI       NIC-specific
(Vendor ID)
```

- 48 bits total, written as 6 hex pairs
- The first 3 bytes = **OUI (Organizationally Unique Identifier)** — identifies the manufacturer
- The last 3 bytes = unique to that specific device
- Example: `00:50:56:xx:xx:xx` = VMware virtual machine

**MAC vs IP:**
- **MAC** = physical/hardware address = used within a local network (Layer 2)
- **IP** = logical/software address = used across networks (Layer 3)
- A MAC address identifies *which device* on the local segment; an IP address identifies *where* in the global network

```bash
ip link show                    # Show interfaces with MAC addresses
ip link show eth0 | grep ether  # Show MAC of eth0
```

### ARP — Address Resolution Protocol

**ARP** is the protocol that bridges Layer 3 (IP) and Layer 2 (MAC). When your computer wants to send data to an IP address on the **same subnet**, it needs to know that device's MAC address to construct the Ethernet frame.

**The ARP Process:**

```
Your PC (192.168.1.10) wants to talk to 192.168.1.20

Step 1: Check ARP cache — "Do I already know 192.168.1.20's MAC?"
        If yes → use cached MAC. Done.

Step 2: If not cached → broadcast ARP Request:
        "HEY EVERYONE on this network — who has 192.168.1.20? Tell 192.168.1.10"
        Destination MAC: FF:FF:FF:FF:FF:FF (broadcast — all devices receive this)

Step 3: Device with 192.168.1.20 replies directly (unicast):
        "I have 192.168.1.20. My MAC is AA:BB:CC:DD:EE:FF"

Step 4: Your PC caches this MAC → IP mapping
        Now constructs Ethernet frame with destination MAC = AA:BB:CC:DD:EE:FF
```

```bash
arp -n                     # Show ARP cache (IP → MAC mappings)
ip neigh show              # Modern equivalent — show neighbor table
arp -d 192.168.1.20        # Delete an ARP cache entry
```

**ARP Poisoning / Spoofing:** A security attack where a malicious device sends fake ARP replies, poisoning other devices' ARP caches to redirect traffic through the attacker. This is a Layer 2 attack.

### Sending Data to Another Network

If the destination IP is on a **different subnet**, your machine doesn't ARP for the destination directly. Instead, it:
1. Recognizes the destination is off-network (by comparing with subnet mask)
2. ARPs for the **default gateway's** MAC address
3. Sends the frame to the gateway's MAC, with the destination IP set to the final destination

The gateway (router) then handles getting the packet to the right network.

---

## 7. Routing & How Packets Travel

**Routing** is the process of forwarding packets from a source to a destination across one or more networks.

### The Routing Table

Every device that participates in IP networking has a **routing table** — a list of rules that says "to reach network X, send packets to next-hop Y via interface Z."

```bash
ip route show
# Output:
default via 192.168.1.1 dev eth0    # Default route — send everything to gateway
192.168.1.0/24 dev eth0 proto kernel # This subnet is directly connected
10.0.0.0/8 via 10.10.0.1 dev eth1  # Reach 10.x.x.x via 10.10.0.1

route -n    # Older command — same result
netstat -rn # Another option
```

### How a Packet Gets Routed

When a packet arrives at a router, the router looks at the **destination IP** and checks its routing table using **longest prefix match** — the most specific matching route wins.

```
Packet destination: 10.0.5.100

Routing table:
  10.0.0.0/8      via 1.2.3.4
  10.0.5.0/24     via 5.6.7.8    ← more specific, wins!
  default          via 9.9.9.9

Result: Send via 5.6.7.8
```

### Default Gateway

The **default gateway** is where your device sends packets when no more specific route exists. It's typically your router.

When you configure a server, you set:
- IP address + subnet → defines the local network
- Default gateway → where to send traffic that's not local

```bash
ip route add default via 192.168.1.1  # Set default gateway
ip route del default                   # Remove default gateway
```

### How a Packet Travels Across the Internet

```
Your laptop (192.168.1.10) → google.com (142.250.x.x)

1. Laptop → Home Router (default gateway)
   - Ethernet frame: src=your-MAC, dst=router-MAC
   - IP packet: src=192.168.1.10, dst=142.250.x.x

2. Home Router → ISP Router
   - NAT rewrites src IP to your public IP (see NAT section)
   - Forwards to ISP based on routing table

3. ISP Router → Internet backbone routers
   - Each router performs longest-prefix match
   - BGP (Border Gateway Protocol) shares routes between ISPs

4. Google's Router → Google's Server
   - Finally reaches Google's data center network
   - Google's router delivers to the server at 142.250.x.x

Each hop: IP header stays the same (src/dst IP unchanged)
          MAC addresses change at every hop (Layer 2 is hop-by-hop)
```

### TTL — Time To Live

Every IP packet has a **TTL** field (0–255). Each router that forwards the packet decrements TTL by 1. When TTL reaches 0, the router drops the packet and sends an ICMP "Time Exceeded" message back to the sender.

**Purpose:** Prevents packets from looping forever in case of routing loops.

**Useful fact:** `traceroute` works by sending packets with TTL=1, then TTL=2, TTL=3, etc. — each time, a different router returns a "Time Exceeded" message, revealing the path.

```bash
traceroute google.com    # Trace the path to google.com
tracepath google.com     # Similar, no root required
mtr google.com           # Continuous traceroute — shows live RTT per hop
```

---

## 8. TCP — Transmission Control Protocol

**TCP** is the protocol that provides **reliable, ordered, error-checked delivery** of data between applications. It's the foundation of most internet communication — HTTP, HTTPS, SSH, email, databases all run over TCP.

### What TCP Guarantees

1. **Delivery** — Packets that are lost get retransmitted
2. **Order** — Data arrives in the same order it was sent
3. **Error checking** — Corrupted data is detected and retransmitted
4. **Flow control** — Sender doesn't overwhelm the receiver
5. **Congestion control** — Sender backs off when the network is congested

### The TCP Three-Way Handshake

Before any data can be sent, TCP establishes a connection. This is called the **three-way handshake**:

```
Client                        Server
  │                              │
  │─────── SYN ─────────────────→│  "I want to connect. My seq# starts at 100"
  │                              │
  │←────── SYN-ACK ─────────────│  "OK. My seq# starts at 300. ACK your 100"
  │                              │
  │─────── ACK ─────────────────→│  "ACK your 300. Connection established!"
  │                              │
  │════════ DATA FLOWS ══════════│
```

**SYN** = Synchronize sequence numbers
**ACK** = Acknowledge (I received up to seq# X)

After the handshake, both sides know each other's starting sequence numbers and the connection is established.

### Sequence Numbers & ACKs

TCP assigns a **sequence number** to every byte of data. The receiver sends back **ACKs** (acknowledgments) confirming receipt.

```
Client sends: [SEQ=1, DATA=bytes 1-100]
              [SEQ=101, DATA=bytes 101-200]
              [SEQ=201, DATA=bytes 201-300]

Server ACKs:  [ACK=101]  "I got up to byte 100, send from 101"
              [ACK=201]  "I got up to byte 200"
              [ACK=301]  "I got up to byte 300"
```

If the client doesn't receive an ACK within a timeout, it retransmits the segment.

### TCP Connection Teardown — Four-Way Handshake

Closing a TCP connection is a four-step process:

```
Client                        Server
  │─────── FIN ─────────────────→│  "I'm done sending"
  │←─────── ACK ────────────────│  "OK, noted"
  │←─────── FIN ────────────────│  "I'm done sending too"
  │─────── ACK ─────────────────→│  "OK"
  │                              │
  (Client waits in TIME_WAIT for ~2 mins before fully closing)
```

**TIME_WAIT state:** After the last ACK, the client waits before fully closing the connection. This ensures the final ACK arrived and prevents a new connection on the same port from receiving old packets.

If you see many connections in TIME_WAIT on a busy server, it's normal — it means a lot of short-lived connections. Tuning `tcp_fin_timeout` can help if it becomes a problem.

### TCP Connection States

```bash
ss -tan                    # Show all TCP connections with states
netstat -tan               # Older equivalent
```

| State | Meaning |
|---|---|
| **LISTEN** | Server is waiting for incoming connections |
| **SYN_SENT** | Client sent SYN, waiting for SYN-ACK |
| **SYN_RECEIVED** | Server got SYN, sent SYN-ACK, waiting for ACK |
| **ESTABLISHED** | Connection is open, data can flow |
| **FIN_WAIT_1** | Sent FIN, waiting for ACK |
| **FIN_WAIT_2** | Got ACK of FIN, waiting for FIN from other side |
| **TIME_WAIT** | Waiting to ensure remote side got the final ACK |
| **CLOSE_WAIT** | Got FIN from remote, waiting for local app to close |
| **CLOSED** | No connection |

### TCP Flow Control — Window Size

TCP uses a **sliding window** mechanism for flow control. The receiver tells the sender how much buffer space it has available (the **receive window**). The sender can only send that much data before waiting for an ACK.

```
Receiver says: "My window is 64KB"
Sender sends up to 64KB without waiting for ACK
Receiver ACKs and says: "Window is now 48KB" (buffer is getting full)
Sender slows down
```

### TCP Congestion Control

When packets are dropped (assumed to be due to network congestion), TCP reduces its sending rate. Algorithms: Slow Start, Congestion Avoidance, CUBIC (default in Linux).

```bash
sysctl net.ipv4.tcp_congestion_control     # See current algorithm
sysctl -w net.ipv4.tcp_congestion_control=bbr  # Switch to BBR (Google's algorithm — better for modern networks)
```

---

## 9. UDP — User Datagram Protocol

**UDP** is the lightweight, connectionless counterpart to TCP. It sends data with no handshake, no acknowledgment, no ordering, no retransmission.

### What UDP Provides (and Doesn't)

| Feature | TCP | UDP |
|---|---|---|
| Connection setup | Yes (3-way handshake) | No |
| Delivery guarantee | Yes | No |
| Ordering | Yes | No |
| Error detection | Yes | Basic checksum only |
| Retransmission | Yes | No |
| Overhead | Higher | Lower |
| Speed | Slower | Faster |

### How UDP Works

```
Client                        Server
  │                              │
  │──── [DATA, no connection] ──→│  Fire and forget
  │                              │
  No ACK, no state, no guarantee
```

UDP just fires packets and doesn't care if they arrive. The application layer handles any reliability if needed.

### Why UDP Exists

Sometimes you don't want TCP's overhead:

- **Latency is critical:** A video call can tolerate a dropped frame — but it cannot tolerate waiting for retransmission (which makes the call jitter). Better to just drop and keep going.
- **Many small requests:** DNS queries are usually tiny one-shot request/response pairs. The overhead of establishing a TCP connection would outweigh the benefits.
- **Broadcasting/Multicasting:** TCP is point-to-point; UDP can be sent to multiple receivers at once.
- **Custom reliability:** Some applications (game engines, QUIC protocol) implement their own, more efficient reliability on top of UDP.

---

## 10. TCP vs UDP — When to Use Which

| Use Case | Protocol | Why |
|---|---|---|
| **Web browsing (HTTP/HTTPS)** | TCP | Need all data, in order, no corruption |
| **DNS queries** | UDP (default) | Small, fast, acceptable to retry if no response |
| **DNS zone transfers** | TCP | Large data transfer, needs reliability |
| **Video streaming (Netflix)** | TCP | Buffered — reliability preferred |
| **Live video calls (Zoom)** | UDP | Low latency preferred over perfect delivery |
| **Online gaming** | UDP | Low latency — a dropped packet is better than a delayed one |
| **SSH** | TCP | Every character must arrive in order |
| **Email (SMTP)** | TCP | Must be reliable — can't lose email |
| **DHCP** | UDP | Bootstrap protocol, no IP yet |
| **NTP (time sync)** | UDP | Simple request/response, no connection needed |
| **VPN (WireGuard)** | UDP | Lower overhead, handles reliability in own layer |
| **Database connections** | TCP | Data integrity is critical |

> **Rule of thumb:** If data loss is unacceptable → TCP. If latency matters more than perfection → UDP.

---

## 11. Ports & Sockets

### What is a Port?

A **port** is a 16-bit number (0–65535) that identifies a specific process or service on a host.

An IP address gets you to the right machine. A port number gets you to the right application on that machine.

Think of it like an apartment building:
- **IP address** = the building address
- **Port** = the apartment number

```
192.168.1.10:80   → web server
192.168.1.10:443  → HTTPS
192.168.1.10:22   → SSH daemon
192.168.1.10:5432 → PostgreSQL
```

### Port Ranges

| Range | Name | Description |
|---|---|---|
| 0 – 1023 | **Well-known ports** | Standardized for common services. Requires root to bind. |
| 1024 – 49151 | **Registered ports** | Used by applications, registered with IANA |
| 49152 – 65535 | **Ephemeral/Dynamic ports** | Used by OS for client-side connections (randomly assigned) |

### Well-Known Ports You Must Know

| Port | Protocol | Service |
|---|---|---|
| 20, 21 | TCP | FTP (data, control) |
| 22 | TCP | SSH |
| 23 | TCP | Telnet (insecure, obsolete) |
| 25 | TCP | SMTP (email sending) |
| 53 | TCP/UDP | DNS |
| 67, 68 | UDP | DHCP (server, client) |
| 80 | TCP | HTTP |
| 110 | TCP | POP3 (email retrieval) |
| 123 | UDP | NTP (time sync) |
| 143 | TCP | IMAP (email retrieval) |
| 443 | TCP | HTTPS |
| 465, 587 | TCP | SMTP with TLS |
| 993, 995 | TCP | IMAP/POP3 over TLS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 8080, 8443 | TCP | Alternate HTTP/HTTPS (non-privileged) |
| 27017 | TCP | MongoDB |

### What is a Socket?

A **socket** is the combination of an IP address and a port number. It's the endpoint of a network connection.

```
Socket = IP address + Port
192.168.1.10:80    ← the server's socket
192.168.1.50:52341 ← the client's socket (ephemeral port assigned by OS)
```

A **connection** is identified by a 5-tuple:
```
(Protocol, Source IP, Source Port, Destination IP, Destination Port)
(TCP, 192.168.1.50, 52341, 192.168.1.10, 80)
```

This is why a server can handle millions of connections on the same port (80/443) — each connection has a different source IP:port combination, making it unique.

### Checking Ports and Sockets

```bash
ss -tlnp              # TCP listening ports + which process (t=tcp, l=listening, n=numeric, p=process)
ss -ulnp              # UDP listening ports
ss -tan               # All TCP connections with states
ss -s                 # Summary statistics

netstat -tlnp         # Older equivalent to ss
lsof -i :80           # What process is using port 80
lsof -i tcp           # All TCP connections

# Test if a remote port is open
nc -zv 192.168.1.10 80        # Test TCP port 80
nc -zvu 192.168.1.10 53       # Test UDP port 53
telnet 192.168.1.10 80        # Old-school port test
```

### Ephemeral Ports

When your browser connects to a web server, the OS assigns a random **source port** from the ephemeral range (49152–65535 on Linux, configured in `/proc/sys/net/ipv4/ip_local_port_range`).

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
# 32768    60999    ← default on most Linux systems

# Expand the range for high-connection servers
echo "1024 65535" > /proc/sys/net/ipv4/ip_local_port_range
```

---

## 12. DNS — Domain Name System

**DNS** translates human-readable domain names (`google.com`) into IP addresses (`142.250.80.46`). It's the phone book of the internet.

### Why DNS Exists

Humans remember names, computers route by IP addresses. DNS is the translation layer.

Without DNS, you'd have to type `142.250.80.46` instead of `google.com` — and every time Google changed their servers, you'd need new IP addresses.

### DNS Hierarchy

DNS is a distributed, hierarchical database — no single server holds all records.

```
                        Root (.)
                       /         \
              .com           .org      .io    .net  ...
             /     \             \
          google   amazon     wikipedia
         /     \
       www    mail
```

- **Root nameservers:** 13 logical root servers (labeled a–m.root-servers.net) that know the authoritative nameservers for every TLD
- **TLD nameservers:** Know the authoritative nameservers for all domains under that TLD (.com, .org, etc.)
- **Authoritative nameservers:** The final authority for a specific domain — hold the actual DNS records

### DNS Resolution — Step by Step

When you type `www.google.com` in your browser:

```
Browser                 Resolver              Root NS        .com NS     Google NS
   │                       │                     │              │            │
   │ 1. Check browser cache│                     │              │            │
   │ 2. Check OS cache     │                     │              │            │
   │ 3. Ask resolver ─────→│                     │              │            │
   │                       │ 4. Ask root: .com?─→│              │            │
   │                       │ ←─ Root says: ask   │              │            │
   │                       │    a.gtld-servers.net│             │            │
   │                       │ 5. Ask .com NS: ──────────────────→│            │
   │                       │    google.com?       │             │            │
   │                       │ ←─ .com says: ask    │             │            │
   │                       │    ns1.google.com    │             │            │
   │                       │ 6. Ask Google NS: ────────────────────────────→│
   │                       │    www.google.com?   │             │            │
   │                       │ ←─ Answer: 142.250.80.46 ──────────────────────│
   │                       │ 7. Cache the answer  │             │            │
   │ ←───────────────────  │                     │              │            │
   │  142.250.80.46        │                     │              │            │
```

- **Steps 1-2:** Browser checks its own cache, then the OS cache (`/etc/hosts`, systemd-resolved cache)
- **Step 3:** Asks the configured DNS resolver (usually your router or ISP, or 8.8.8.8)
- **Steps 4-6:** The resolver does the recursive work (this is a **recursive resolver**)
- **Step 7:** Resolver caches the result for TTL seconds

### DNS Record Types

| Type | Purpose | Example |
|---|---|---|
| **A** | Maps domain to IPv4 address | `google.com → 142.250.80.46` |
| **AAAA** | Maps domain to IPv6 address | `google.com → 2607:f8b0:...` |
| **CNAME** | Alias — points to another domain name | `www.example.com → example.com` |
| **MX** | Mail exchange — where to send email | `example.com → mail.example.com (priority 10)` |
| **NS** | Nameserver — who is authoritative for this domain | `google.com → ns1.google.com` |
| **TXT** | Free-form text — used for SPF, DKIM, domain verification | `"v=spf1 include:..."` |
| **SOA** | Start of Authority — meta info about the zone | Serial, refresh intervals, admin contact |
| **PTR** | Reverse DNS — IP to hostname | `46.80.250.142.in-addr.arpa → google.com` |
| **SRV** | Service location — host + port for a service | Used by SIP, Kubernetes, etc. |

### TTL — Time To Live (DNS Context)

Each DNS record has a **TTL** (seconds) that tells resolvers how long to cache it.

- **High TTL (86400 = 1 day):** Fewer DNS queries, faster lookups, but changes take longer to propagate
- **Low TTL (60 = 1 minute):** Changes propagate quickly, useful before migrations, but more DNS load

Before a major DNS change (migrating servers), lower the TTL days in advance so clients pick up the change quickly.

### DNS Tools

```bash
# Basic lookup
nslookup google.com            # Simple DNS query
dig google.com                 # Detailed DNS query (better than nslookup)
host google.com                # Simple lookup

# dig examples
dig google.com A               # Query A record
dig google.com MX              # Query MX records
dig google.com +short          # Just the answer, no extra info
dig @8.8.8.8 google.com        # Use Google's DNS server (8.8.8.8) instead of default
dig google.com +trace          # Trace the full resolution path (root → TLD → authoritative)
dig -x 142.250.80.46           # Reverse DNS — IP to hostname

# Check what DNS server your system uses
cat /etc/resolv.conf
resolvectl status              # systemd-resolved status
```

### /etc/hosts — The Local Override

Before DNS, `/etc/hosts` was the internet's phone book. Today it still takes priority over DNS for local resolution:

```bash
cat /etc/hosts
# 127.0.0.1 localhost
# 192.168.1.10 myserver myserver.local
# 10.0.0.5 database db
```

Very useful for:
- Local development (map `myapp.local` to `127.0.0.1`)
- Blocking domains (map `ads.tracking.com` to `127.0.0.1`)
- Overriding DNS in emergencies

---

## 13. HTTP & HTTPS

**HTTP (HyperText Transfer Protocol)** is the protocol that powers the web. It defines how browsers and servers communicate.

### HTTP Basics

HTTP is a **request-response protocol** — a client sends a request, the server sends a response.

It runs over TCP (port 80 for HTTP, 443 for HTTPS) and is a **stateless protocol** — each request is independent; the server doesn't remember previous requests by default. Cookies and sessions are built on top to add state.

### HTTP Request Structure

```
GET /api/users HTTP/1.1
Host: api.example.com
User-Agent: Mozilla/5.0
Accept: application/json
Authorization: Bearer eyJhbGci...
Connection: keep-alive

[Optional request body — for POST, PUT, PATCH]
```

Components:
- **Method** — what action to perform
- **Path** — the resource URL
- **HTTP version** — 1.0, 1.1, 2, 3
- **Headers** — key-value metadata
- **Body** — data payload (optional)

### HTTP Methods

| Method | Purpose | Idempotent? | Has Body? |
|---|---|---|---|
| **GET** | Retrieve a resource | Yes | No |
| **POST** | Create a new resource | No | Yes |
| **PUT** | Replace a resource entirely | Yes | Yes |
| **PATCH** | Partially update a resource | No | Yes |
| **DELETE** | Delete a resource | Yes | Optional |
| **HEAD** | Like GET but returns headers only (no body) | Yes | No |
| **OPTIONS** | Get supported methods for a URL (used in CORS) | Yes | No |

**Idempotent** means calling the same request multiple times has the same result as calling it once. GET/PUT/DELETE are idempotent. POST is not — submitting a form twice creates two records.

### HTTP Response Structure

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 128
Cache-Control: max-age=3600
Date: Wed, 22 Apr 2026 10:00:00 GMT

{"users": [{"id": 1, "name": "Alice"}, ...]}
```

### HTTP Status Codes — Know These Cold

**1xx — Informational**
- `100 Continue` — Server received the request headers, client should continue

**2xx — Success**
- `200 OK` — Request succeeded
- `201 Created` — Resource created (response to POST)
- `204 No Content` — Success but no response body (DELETE responses)

**3xx — Redirection**
- `301 Moved Permanently` — URL has permanently changed (browsers cache this)
- `302 Found` — Temporary redirect
- `304 Not Modified` — Cached version is still valid (based on ETag/Last-Modified)

**4xx — Client Errors**
- `400 Bad Request` — Malformed request syntax
- `401 Unauthorized` — Not authenticated (need to log in)
- `403 Forbidden` — Authenticated but not authorized (no permission)
- `404 Not Found` — Resource doesn't exist
- `405 Method Not Allowed` — Wrong HTTP method for this endpoint
- `408 Request Timeout` — Client took too long
- `409 Conflict` — Resource conflict (duplicate, version mismatch)
- `429 Too Many Requests` — Rate limiting

**5xx — Server Errors**
- `500 Internal Server Error` — Generic server error
- `502 Bad Gateway` — Upstream server returned invalid response
- `503 Service Unavailable` — Server is down or overloaded
- `504 Gateway Timeout` — Upstream server didn't respond in time

> **401 vs 403:** 401 = "I don't know who you are." 403 = "I know who you are, but you can't do this."
> **502 vs 504:** 502 = upstream responded with garbage. 504 = upstream didn't respond at all (timeout).

### Important HTTP Headers

**Request headers:**
```
Host: example.com                   # Which virtual host to reach (required in HTTP/1.1)
Authorization: Bearer <token>       # Auth credential
Content-Type: application/json      # Format of the request body
Accept: application/json            # What format the client wants in response
User-Agent: curl/7.68.0             # Who is making the request
Cookie: session=abc123              # Session cookies
X-Forwarded-For: 203.0.113.5       # Real client IP (set by load balancers/proxies)
```

**Response headers:**
```
Content-Type: text/html; charset=utf-8    # What type of data is in the body
Content-Length: 1024                       # Size of response body in bytes
Cache-Control: max-age=86400               # Caching instructions
Set-Cookie: session=xyz; HttpOnly; Secure  # Set a cookie
Location: https://new.example.com          # Used with 3xx redirects
Strict-Transport-Security: max-age=31536000  # Force HTTPS (HSTS)
X-Content-Type-Options: nosniff           # Security header
```

### HTTP Versions

| Version | Key Feature |
|---|---|
| **HTTP/1.0** | One TCP connection per request — very slow |
| **HTTP/1.1** | Persistent connections (keep-alive), pipelining |
| **HTTP/2** | Multiplexing (multiple requests over one connection), header compression, binary protocol |
| **HTTP/3** | Runs over QUIC (UDP-based) instead of TCP — lower latency, better for unreliable networks |

```bash
curl -v https://example.com           # See full request + response headers
curl -I https://example.com           # HEAD request — headers only
curl -w "%{http_code}" https://ex.com # Print just the status code
curl --http2 https://example.com      # Force HTTP/2
```

---

## 14. TLS/SSL — How Encryption Works

**TLS (Transport Layer Security)** is the protocol that encrypts communication between a client and server. When you see `https://`, TLS is running underneath HTTP.

**SSL (Secure Sockets Layer)** is the old name for TLS. SSL is deprecated (versions 2.0 and 3.0 are insecure). Modern systems use **TLS 1.2 or TLS 1.3**. When people say "SSL certificate," they really mean a TLS certificate.

### What TLS Provides

1. **Encryption** — Data in transit cannot be read by anyone intercepting it
2. **Authentication** — You can verify you're talking to the real server (not an imposter)
3. **Integrity** — Data cannot be tampered with in transit without detection

### Symmetric vs Asymmetric Encryption

Understanding TLS requires understanding these two types of encryption:

**Symmetric Encryption:**
- Both sides use the **same key** to encrypt and decrypt
- Very fast — used for bulk data encryption
- Problem: How do you securely share the key initially?
- Examples: AES-256, AES-128

**Asymmetric Encryption (Public Key Cryptography):**
- Uses a **key pair** — a public key and a private key
- What one key encrypts, only the other can decrypt
- **Public key** can be shared with anyone
- **Private key** is kept secret by the owner
- Sender encrypts with recipient's public key → only recipient can decrypt with private key
- Slow — only used for key exchange and digital signatures
- Examples: RSA, ECDSA, Diffie-Hellman

**TLS uses both:** Asymmetric encryption is used to securely exchange a symmetric key, then the symmetric key encrypts all actual data (fast).

### The TLS Handshake (TLS 1.3 Simplified)

```
Client                              Server
  │                                    │
  │──── ClientHello ──────────────────→│
  │     (supported TLS versions,       │
  │      cipher suites, random value)  │
  │                                    │
  │←─── ServerHello ──────────────────│
  │     (chosen cipher, random value,  │
  │      server's certificate)         │
  │                                    │
  │  Client verifies certificate:      │
  │  - Not expired?                    │
  │  - Signed by trusted CA?           │
  │  - Matches the domain?             │
  │                                    │
  │──── Key Exchange ────────────────→│
  │     (using asymmetric crypto to    │
  │      agree on a session key)       │
  │                                    │
  │←══════ Encrypted Data ════════════│
  │        (using the session key)     │
```

### TLS Certificates

A **TLS certificate** is a digital document that:
1. Contains the server's public key
2. States which domain(s) it's valid for
3. Has an expiry date
4. Is **signed by a Certificate Authority (CA)**

The signature from a trusted CA is what makes the certificate trustworthy. If anyone could issue a certificate for `google.com`, attackers could impersonate Google. Only CAs that browser vendors trust can issue certificates.

### Certificate Authority (CA) Chain of Trust

```
Root CA (self-signed — built into browsers/OS)
    ↓ signs
Intermediate CA
    ↓ signs
Server Certificate (your domain)
```

Browsers ship with ~100 trusted root CAs. When you connect to a server:
1. Server presents its certificate + intermediate certificate(s)
2. Browser verifies: intermediate cert is signed by root CA, server cert signed by intermediate
3. If the chain is valid and the root CA is trusted → secure connection

```bash
# Inspect a website's certificate
openssl s_client -connect example.com:443 -servername example.com

# Check certificate expiry date
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# Check a local certificate file
openssl x509 -in certificate.crt -noout -text

# View certificate from a file — just the dates
openssl x509 -in cert.pem -noout -enddate
```

### Let's Encrypt

**Let's Encrypt** is a free, automated CA that made HTTPS adoption widespread. Certificates are:
- Free
- Valid for 90 days (auto-renewed)
- Issued via the **ACME protocol** — automated domain ownership verification

```bash
# Get a certificate with certbot
certbot --nginx -d example.com -d www.example.com
certbot renew --dry-run   # Test renewal
certbot renew             # Renew all certificates
```

### Common TLS Issues

| Error | Cause |
|---|---|
| `SSL_ERROR_EXPIRED_CERT` | Certificate past expiry date |
| `SSL_ERROR_UNKNOWN_ISSUER` | CA not trusted (self-signed or private CA) |
| `SSL_ERROR_BAD_CERT_DOMAIN` | Certificate domain doesn't match the URL |
| `SSL certificate problem: unable to get local issuer certificate` | Missing intermediate certificate in chain |
| `SSL_ERROR_HANDSHAKE_FAILURE` | Protocol version or cipher mismatch |

---

## 15. Load Balancing

A **load balancer** distributes incoming network traffic across multiple backend servers to:
- Improve availability (if one server dies, others handle traffic)
- Improve performance (no single server is overwhelmed)
- Enable horizontal scaling (add more servers behind the LB)

### Layer 4 vs Layer 7 Load Balancing

#### Layer 4 Load Balancer (Transport Layer)
- Routes based on **IP addresses and TCP/UDP ports**
- Does not look inside the packet contents
- Very fast — just routes the connection
- Cannot make routing decisions based on URL, headers, or cookies
- Example: AWS Network Load Balancer (NLB)

```
Client → Load Balancer (sees src/dst IP and port only) → Backend Server
```

#### Layer 7 Load Balancer (Application Layer)
- Routes based on **HTTP content** — URL path, Host header, cookies, request content
- Can terminate TLS (decrypt the request to read its content)
- Smarter routing — e.g., route `/api/` requests to API servers, `/static/` to CDN
- Can modify requests/responses, add headers, do health checks on HTTP paths
- Slightly more overhead than L4 (needs to parse HTTP)
- Examples: nginx, HAProxy, AWS Application Load Balancer (ALB), Traefik

```
Client → Load Balancer (reads URL, headers, cookies) → Correct backend pool
         /api/* → api-servers
         /static/* → static-servers
```

### Load Balancing Algorithms

| Algorithm | How it works | Best for |
|---|---|---|
| **Round Robin** | Send each request to the next server in rotation | Servers with equal capacity, stateless apps |
| **Least Connections** | Send to server with fewest active connections | Servers with varying connection lengths |
| **Weighted Round Robin** | Like Round Robin, but some servers get more traffic | Servers with different capacities |
| **IP Hash** | Always send same client IP to same server | Apps that need client affinity |
| **Least Response Time** | Send to the fastest-responding server | Latency-sensitive applications |
| **Random** | Pick a random server | Simple and surprisingly effective |

### Sticky Sessions (Session Affinity)

Some applications store session data in memory. If a user's requests go to different servers, they lose their session.

**Sticky sessions** ensure a user always goes to the same backend server. The load balancer sets a cookie (or uses IP hash) to remember which backend to use.

**Problem:** Defeats the purpose of load balancing if some users are "heavy" — one server gets overloaded. Modern architecture solves this by storing sessions externally (Redis) instead.

### Health Checks

Load balancers constantly probe backends to check if they're healthy:
- **TCP health check:** Can I open a TCP connection to port X? (Layer 4)
- **HTTP health check:** Does `GET /healthz` return 200 OK? (Layer 7)

If a backend fails health checks, the LB stops sending it traffic until it recovers.

```nginx
# nginx upstream with health check config
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}
```

---

## 16. Firewalls & iptables

A **firewall** is a network security system that monitors and controls incoming and outgoing traffic based on rules.

### Types of Firewalls

| Type | How it works |
|---|---|
| **Packet filter** | Filters based on IP, port, protocol — stateless |
| **Stateful firewall** | Tracks connection state — knows if a packet is part of an established connection |
| **Application firewall (WAF)** | Inspects application-layer content (SQL injection, XSS patterns) |

### iptables — Linux Firewall

**iptables** is the traditional Linux firewall tool. It uses rules organized into **chains** within **tables**.

#### Tables

| Table | Purpose |
|---|---|
| **filter** | Default — accept/drop/reject packets (what most people use) |
| **nat** | Network Address Translation — modify source/destination addresses |
| **mangle** | Modify packet headers |
| **raw** | Control connection tracking |

#### Chains (in the filter table)

| Chain | Applies to |
|---|---|
| **INPUT** | Packets destined for this machine |
| **OUTPUT** | Packets originating from this machine |
| **FORWARD** | Packets passing through this machine (routing) |

#### How a Packet Flows Through iptables

```
Incoming packet
      ↓
   PREROUTING (nat table — DNAT)
      ↓
  Is it for this machine?
   ↙              ↘
INPUT          FORWARD
(local)        (routing)
   ↓                ↓
Local process  POSTROUTING (nat table — SNAT/MASQUERADE)
   ↓
OUTPUT
   ↓
POSTROUTING
```

#### Common iptables Commands

```bash
# View current rules
iptables -L                          # List all filter rules
iptables -L -n -v --line-numbers     # Verbose with line numbers and no DNS lookup
iptables -L INPUT -n -v              # Just the INPUT chain

# Allow/block traffic
iptables -A INPUT -p tcp --dport 80 -j ACCEPT    # Allow incoming TCP port 80
iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # Allow SSH
iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT   # Allow from subnet
iptables -A INPUT -j DROP                         # Drop everything else (default deny)

# Delete a rule
iptables -D INPUT -p tcp --dport 80 -j ACCEPT    # Delete specific rule
iptables -D INPUT 3                               # Delete rule by line number

# Allow established connections (stateful)
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Save rules (varies by distro)
iptables-save > /etc/iptables/rules.v4            # Save current rules
iptables-restore < /etc/iptables/rules.v4         # Restore saved rules
```

### nftables — The Modern Replacement

`nftables` replaced `iptables` in modern Linux (kernel 3.13+). Same concepts, cleaner syntax, better performance.

```bash
nft list ruleset              # View all nftables rules
```

### firewalld — The User-Friendly Wrapper

`firewalld` is a front-end for nftables/iptables used on RHEL/CentOS/Fedora:

```bash
firewall-cmd --list-all                          # Show current config
firewall-cmd --add-port=8080/tcp --permanent     # Open port 8080
firewall-cmd --add-service=http --permanent      # Allow HTTP service
firewall-cmd --reload                            # Apply changes
firewall-cmd --zone=public --list-ports          # Show open ports in public zone
```

### ufw — Uncomplicated Firewall (Ubuntu)

`ufw` is Ubuntu's simplified firewall interface:

```bash
ufw status                      # Show firewall status
ufw enable                      # Turn on firewall
ufw allow 22                    # Allow SSH
ufw allow 80/tcp                # Allow HTTP
ufw deny 23                     # Block Telnet
ufw delete allow 80/tcp         # Remove a rule
ufw allow from 192.168.1.0/24  # Allow from subnet
```

---

## 17. NAT — Network Address Translation

**NAT** modifies IP address information in packet headers as packets pass through a router/firewall.

### Why NAT Exists

Private IP addresses are not routable on the internet. But every device at home (using `192.168.x.x`) still needs internet access. NAT is the solution — the router translates private IPs to its single public IP.

### How NAT Works (SNAT / Masquerade)

```
Home Network                       Internet
─────────────                      ────────
192.168.1.10 ──┐
192.168.1.11 ──┼──→ Router (NAT) ──→ Public IP: 203.0.113.5 ──→ Google
192.168.1.12 ──┘

Packet leaving home network:
  Before NAT: src=192.168.1.10:52341, dst=142.250.80.46:443
  After NAT:  src=203.0.113.5:34521,  dst=142.250.80.46:443

Router maintains NAT table:
  203.0.113.5:34521 ↔ 192.168.1.10:52341

Response comes back to 203.0.113.5:34521
Router looks up table → forwards to 192.168.1.10:52341
```

This is called **Source NAT (SNAT)** or **IP Masquerading** — the source IP is rewritten.

### DNAT — Destination NAT (Port Forwarding)

**Destination NAT** changes the destination address. This is how **port forwarding** works.

```
Internet user sends to: 203.0.113.5:80
Router DNAT rule: forward port 80 to internal 192.168.1.100:80

Packet:
  Before DNAT: dst=203.0.113.5:80
  After DNAT:  dst=192.168.1.100:80
```

```bash
# Port forwarding with iptables
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:80
iptables -t nat -A POSTROUTING -j MASQUERADE  # SNAT for outgoing

# View NAT rules
iptables -t nat -L -n -v
```

### NAT in Container and Cloud Networking

NAT is everywhere in modern infrastructure:
- **Docker:** Container IPs (172.17.x.x) are SNATted to the host IP when talking to the internet
- **Kubernetes:** Pod IPs are NATted by kube-proxy for external access
- **AWS:** Instances in private subnets use a NAT Gateway to reach the internet

---

## 18. Common Protocols You Must Know

### SSH — Secure Shell

SSH provides encrypted remote access to servers. Runs on TCP port 22.

```bash
ssh user@server               # Connect to server
ssh -p 2222 user@server       # Connect on non-standard port
ssh -i ~/.ssh/mykey user@server  # Use specific private key
ssh -L 8080:localhost:80 user@server  # Local port forwarding
ssh -R 9090:localhost:3000 user@server  # Remote port forwarding

# SSH config for convenience (~/.ssh/config)
Host myserver
    HostName 192.168.1.10
    User ubuntu
    IdentityFile ~/.ssh/mykey
    Port 22
# Then just: ssh myserver
```

**How SSH key authentication works:**
1. Generate a key pair: `ssh-keygen -t ed25519`
2. Put public key on server: `~/.ssh/authorized_keys`
3. Client proves identity by signing a challenge with the private key — server verifies with public key
4. No password required — the private key is the credential

### DHCP — Dynamic Host Configuration Protocol

DHCP automatically assigns IP configuration to devices when they join a network.

```
Device joins network
  ↓
DHCP Discover (broadcast: "Is there a DHCP server? I need an IP!")
  ↓
DHCP Offer (server: "Here, take 192.168.1.50 for 24 hours")
  ↓
DHCP Request (device: "I'll take that IP, 192.168.1.50")
  ↓
DHCP Acknowledge (server: "It's yours. Here's your subnet, gateway, DNS too.")
```

DHCP assigns: IP address, subnet mask, default gateway, DNS servers, lease time.

### NTP — Network Time Protocol

NTP synchronizes clocks across machines. UDP port 123.

Why it matters for DevOps: Log correlation, TLS certificates, Kerberos authentication, database transactions — all require accurate time. Even a few seconds off can cause problems.

```bash
timedatectl                    # Show time and sync status
timedatectl set-ntp true       # Enable NTP sync
chronyc tracking               # If using chrony — check NTP sync status
ntpq -p                        # If using ntpd — show NTP peers
```

### SMTP — Simple Mail Transfer Protocol

SMTP sends email between servers. TCP ports 25 (server-to-server), 587 (client submission), 465 (SMTPS).

The flow: Your email client → your mail server (SMTP/587) → recipient's mail server (SMTP/25) → recipient's inbox (IMAP/POP3).

### FTP — File Transfer Protocol

FTP transfers files. TCP ports 20 (data) and 21 (control).

**Active vs Passive FTP:**
- **Active:** Server initiates data connection back to client — problematic with firewalls
- **Passive:** Client initiates both connections — works better with NAT/firewalls

FTP sends credentials in plaintext — never use FTP over the internet. Use **SFTP** (SSH File Transfer Protocol, runs over SSH) or **FTPS** (FTP + TLS) instead.

```bash
sftp user@server               # Connect via SFTP
sftp> get remote_file.txt      # Download a file
sftp> put local_file.txt       # Upload a file
sftp> ls                       # List remote directory
```

---

## 19. Network Interfaces & the Linux Network Stack

### Network Interfaces

A **network interface** is the OS representation of a network connection — physical (Ethernet NIC) or virtual (loopback, tunnel, bridge).

```bash
ip link show                   # All interfaces with state (UP/DOWN)
ip addr show                   # All interfaces with IP addresses
ip link show eth0              # Specific interface
```

Common interface names:
- `eth0`, `eth1` — Physical Ethernet (old naming)
- `ens3`, `eno1`, `enp2s0` — Physical Ethernet (predictable naming)
- `wlan0`, `wlp2s0` — Wi-Fi
- `lo` — Loopback (127.0.0.1) — always present
- `docker0` — Docker bridge network
- `veth...` — Virtual Ethernet pair (used by containers)
- `tun0`, `tap0` — VPN tunnel interfaces
- `br0` — Network bridge
- `bond0` — Bonded interface (multiple NICs combined)

```bash
# Manage interfaces
ip link set eth0 up            # Bring interface up
ip link set eth0 down          # Bring interface down
ip addr add 192.168.1.10/24 dev eth0  # Assign IP to interface
ip addr del 192.168.1.10/24 dev eth0  # Remove IP

# Network statistics
ip -s link show eth0           # Packet counts, errors for eth0
cat /proc/net/dev              # Raw network statistics
ethtool eth0                   # NIC speed, duplex, driver info
```

### Network Interface Bonding / Teaming

**Bonding** combines multiple physical interfaces into one logical interface for:
- **Active-backup:** One interface active, others standby (failover)
- **LACP (802.3ad):** Active load balancing across multiple links (requires switch support)

### The Linux Network Stack Briefly

When a packet arrives at a NIC:
```
NIC receives packet
  ↓
Hardware interrupt → kernel gets packet into ring buffer
  ↓
NIC driver processes interrupt
  ↓
Network stack: Ethernet layer removes frame header → hands IP packet up
  ↓
IP layer: checks destination IP
  Is it for this machine?
    Yes → hands to Transport layer (TCP/UDP demux by port)
         → hands to socket → application reads it
    No  → forward (if routing enabled) or drop
```

### Network Namespaces (Revisiting from OS section)

Each **network namespace** has its own network interfaces, routing table, firewall rules, and sockets. This is how containers get isolated networking.

```bash
ip netns list                             # List network namespaces
ip netns add mynamespace                  # Create a namespace
ip netns exec mynamespace ip addr show    # Run command inside namespace
```

---

## 20. Virtual Networking — VLANs, VPNs, Tunnels

### VLANs — Virtual LANs

A **VLAN** logically segments a physical network into isolated broadcast domains, without needing physical separation.

On a managed switch, you can tag ports as belonging to different VLANs. Traffic from VLAN 10 cannot directly reach VLAN 20 without going through a router.

```
Physical switch
├── Port 1-5:  VLAN 10 (Web servers)
├── Port 6-10: VLAN 20 (Database servers)
└── Port 11:   VLAN 30 (Management)

Web server in VLAN 10 cannot directly communicate with DB in VLAN 20
→ Must go through a router/Layer 3 switch
→ Where firewall rules can be applied
```

In Linux:
```bash
# Create a VLAN interface (requires 8021q kernel module)
modprobe 8021q
ip link add link eth0 name eth0.10 type vlan id 10
ip addr add 10.0.10.1/24 dev eth0.10
ip link set eth0.10 up
```

### VPNs — Virtual Private Networks

A **VPN** creates an encrypted tunnel between two endpoints over a public network (the internet). Traffic appears to come from inside the private network.

**Use cases:**
- Remote employee accessing company resources
- Connecting two data center networks (site-to-site VPN)
- Encrypting traffic on untrusted networks

**Common VPN protocols:**

| Protocol | Description | Port |
|---|---|---|
| **WireGuard** | Modern, fast, minimal codebase — the new standard | UDP 51820 |
| **OpenVPN** | Mature, widely supported, SSL/TLS based | UDP/TCP 1194 |
| **IPsec/IKEv2** | Industry standard, built into most OSes | UDP 500, 4500 |
| **OpenConnect/AnyConnect** | Common in enterprise | TCP 443 |

```bash
# WireGuard basics
wg show                         # Show WireGuard interfaces and status
wg-quick up wg0                 # Bring up wg0 interface
wg-quick down wg0               # Bring down wg0 interface
```

### Tunnels

A **tunnel** encapsulates packets of one protocol inside another. VPNs use tunnels, but tunnels are also used for other purposes.

**Common tunnel types:**
- **GRE (Generic Routing Encapsulation):** Encapsulate any network protocol inside IP. No encryption — often combined with IPsec.
- **VXLAN (Virtual Extensible LAN):** Encapsulates Layer 2 frames in UDP. Used in Kubernetes, OpenStack for overlay networking.
- **IPIP:** IP inside IP — simple tunnel.

Kubernetes pod networking (Flannel, Calico) uses VXLAN or other overlay tunnels to route pod traffic across nodes.

---

## 21. Cloud Networking Concepts

### VPC — Virtual Private Cloud

A **VPC** is your private, isolated section of the cloud provider's network. It's a logically isolated virtual network where you launch your resources.

```
VPC: 10.0.0.0/16
├── Public Subnet:  10.0.1.0/24  (has Internet Gateway route)
│   ├── Load Balancer
│   └── Bastion Host
│
├── Private Subnet: 10.0.2.0/24  (no direct internet route)
│   ├── App Servers
│   └── NAT Gateway traffic goes to: 10.0.3.0/24
│
└── Database Subnet: 10.0.3.0/24 (most restricted)
    └── RDS instances
```

### Security Groups vs NACLs (AWS)

| | Security Groups | Network ACLs |
|---|---|---|
| **Level** | Instance level | Subnet level |
| **State** | Stateful (return traffic auto-allowed) | Stateless (must explicitly allow both directions) |
| **Rules** | Allow only | Allow and Deny |
| **Evaluation** | All rules evaluated | Rules evaluated in order (lowest number first) |
| **Default** | Deny all inbound, allow all outbound | Allow all inbound and outbound |

### Internet Gateway vs NAT Gateway (AWS)

- **Internet Gateway:** Allows resources in **public subnets** to communicate with the internet (bidirectional)
- **NAT Gateway:** Allows resources in **private subnets** to initiate outbound internet connections (e.g., download updates), but blocks inbound connections from the internet (one-directional)

### DNS in AWS

- **Route 53:** AWS's DNS service — authoritative DNS + health checks + routing policies
- **VPC DNS Resolver (169.254.169.253):** Built-in DNS within every VPC — resolves both public DNS and internal service names
- **Private Hosted Zones:** DNS zones visible only within your VPC

### CDN — Content Delivery Network

A **CDN** is a globally distributed network of servers that caches and serves content from locations geographically close to the user.

```
User in Mumbai requests mysite.com/logo.png
  ↓
DNS resolves to nearest CDN edge node (e.g., in Mumbai)
  ↓
If CDN has it cached → serve from cache (fast!)
  ↓
If not cached → CDN fetches from origin server → caches it → serves user
  Next Mumbai user gets it from cache
```

Benefits: Reduced latency, reduced origin server load, DDoS protection (edge absorbs attack).

Examples: Cloudflare, AWS CloudFront, Fastly, Akamai.

---

## 22. Network Troubleshooting — Tools & Methodology

### The Troubleshooting Mindset

Always go **layer by layer, bottom up:**
1. Is the physical connection up? (Layer 1)
2. Is there a valid IP and ARP working? (Layer 2/3)
3. Can I ping the destination? (Layer 3)
4. Is the port reachable? (Layer 4)
5. Is the service responding correctly? (Layer 7)

### Essential Troubleshooting Tools

#### ping — Test reachability (ICMP)
```bash
ping 8.8.8.8                  # Ping by IP — tests Layer 3 connectivity
ping google.com               # Ping by hostname — also tests DNS
ping -c 4 8.8.8.8             # Send exactly 4 packets then stop
ping -i 0.2 8.8.8.8           # Send packets every 0.2 seconds
```
If ping to IP works but ping to hostname fails → **DNS problem**.

#### traceroute / tracepath — Trace the network path
```bash
traceroute google.com         # Show every hop to google.com with RTT
tracepath google.com          # Similar, no root required
mtr google.com                # Continuous traceroute — best for packet loss per hop
```
Look for:
- Where latency jumps suddenly → bottleneck at that hop
- `* * *` (no response) → firewall blocking ICMP at that hop (not necessarily a problem)
- Routing loop (same IPs repeat) → routing misconfiguration

#### curl — Test HTTP/HTTPS endpoints
```bash
curl -v https://example.com                   # Verbose — shows full request/response headers
curl -I https://example.com                   # Headers only
curl -o /dev/null -w "%{http_code}" https://example.com  # Just the status code
curl -k https://example.com                   # Skip TLS certificate verification
curl --resolve example.com:443:1.2.3.4 https://example.com  # Override DNS — test specific IP
curl -H "Host: example.com" http://1.2.3.4   # Send to specific IP with custom Host header
```

#### ss / netstat — Socket statistics
```bash
ss -tlnp          # TCP listening sockets with process names
ss -ulnp          # UDP listening
ss -tan           # All TCP connections with states
ss -s             # Summary
ss -o state established '( dport = :80 or sport = :80 )'  # Connections to/from port 80
```

#### netcat (nc) — The network Swiss Army knife
```bash
nc -zv 10.0.0.1 80             # Test if TCP port 80 is open
nc -zvn 10.0.0.1 1-1000        # Scan ports 1-1000 (slow)
nc -l 1234                     # Listen on port 1234
echo "hello" | nc 10.0.0.1 1234  # Send data to port 1234
nc 10.0.0.1 22                 # Connect to SSH port (see the banner)
```

#### tcpdump — Capture and analyze network traffic
```bash
tcpdump -i eth0                          # Capture everything on eth0
tcpdump -i eth0 port 80                  # Only traffic on port 80
tcpdump -i eth0 host 1.2.3.4            # Only traffic to/from IP
tcpdump -i eth0 -n                       # Don't resolve hostnames (faster)
tcpdump -i eth0 -w capture.pcap         # Save to file for Wireshark analysis
tcpdump -i any 'tcp port 443 and host google.com'  # Complex filter
```

#### nmap — Network scanner
```bash
nmap 192.168.1.1                         # Scan a host for open ports
nmap 192.168.1.0/24                      # Scan entire subnet
nmap -p 80,443,22 192.168.1.1           # Scan specific ports
nmap -sV 192.168.1.1                     # Detect service versions
nmap -O 192.168.1.1                      # OS detection
```

#### dig — DNS diagnostic
```bash
dig google.com                           # Standard A record lookup
dig google.com MX                        # MX records
dig @8.8.8.8 google.com                 # Use specific DNS server
dig google.com +trace                    # Full resolution trace
dig -x 8.8.8.8                          # Reverse DNS
```

#### ip — Network configuration
```bash
ip addr show                             # IP addresses
ip route show                            # Routing table
ip link show                             # Interface status
ip neigh show                            # ARP table
ip route get 8.8.8.8                    # Which route would be used to reach 8.8.8.8
```

### Common Problems and How to Diagnose

**"Cannot connect to server"**
```bash
# Step 1: Is there network connectivity?
ping 8.8.8.8
# If fails → no basic connectivity. Check interface, gateway, routing.

# Step 2: Is DNS working?
dig google.com
# If fails but ping 8.8.8.8 works → DNS problem. Check /etc/resolv.conf

# Step 3: Can we reach the server's IP?
ping 10.0.0.5
# If fails → routing issue, firewall blocking ICMP, or server down

# Step 4: Is the port open?
nc -zv 10.0.0.5 80
# If "Connection refused" → port closed, service not running
# If "No route to host" → firewall dropping the connection

# Step 5: Is the service listening?
ssh user@10.0.0.5
sudo ss -tlnp | grep :80
# If not listening → service crashed or misconfigured

# Step 6: Is the service responding correctly?
curl -v http://10.0.0.5:80
# If connection established but error → application-level problem
```

**"Intermittent packet loss"**
```bash
mtr --report google.com   # Shows per-hop packet loss over time
```

**"High latency"**
```bash
mtr google.com   # Identify which hop introduces the latency jump
```

**"TLS certificate error"**
```bash
echo | openssl s_client -connect example.com:443 2>&1 | grep -E 'Verify|depth|error'
curl -v https://example.com 2>&1 | grep -i ssl
```

**"Too many connections / port exhaustion"**
```bash
ss -s              # Show connection summary — see if TIME_WAIT is very high
cat /proc/sys/net/ipv4/ip_local_port_range   # Check ephemeral port range
```

---

## 23. What Happens When You Type google.com

This is the classic question that tests your entire networking knowledge. Here is every step:

```
You type: https://www.google.com and press Enter
```

### Step 1: Browser Cache Check
Browser checks if it already has a cached IP for `www.google.com` that hasn't expired. If yes, skip DNS. If no, continue.

### Step 2: OS DNS Resolution
Browser asks the OS to resolve `www.google.com`.
1. OS checks `/etc/hosts` — is there a manual entry? No.
2. OS checks its local DNS cache (systemd-resolved) — not cached.
3. OS sends DNS query to the configured DNS resolver (e.g., `8.8.8.8` or your router).

### Step 3: DNS Recursive Resolution
Your DNS resolver (if it doesn't have it cached):
1. Asks a root nameserver: "Who handles .com?"
2. Root says: "Ask `a.gtld-servers.net`"
3. Asks `.com` TLD nameserver: "Who handles `google.com`?"
4. TLD says: "Ask `ns1.google.com`"
5. Asks Google's authoritative nameserver: "What is `www.google.com`?"
6. Google's NS returns: `A record → 142.250.80.46` (example)
7. Resolver caches this (TTL ~300 seconds) and returns it to your OS → browser.

### Step 4: TCP Connection (Three-Way Handshake)
Browser opens a TCP connection to `142.250.80.46:443`.
1. **SYN** → client to server
2. **SYN-ACK** ← server to client
3. **ACK** → client to server
Connection established.

*Before the SYN leaves your machine:*
- Browser checks routing table → send via default gateway (your router)
- ARP lookup for router's MAC address (or uses cached ARP entry)
- Packet sent to router with src=your private IP, dst=142.250.80.46

*At your router:*
- NAT rewrites src IP from `192.168.1.10` → your public IP `203.0.113.5`
- Forwards packet to your ISP

*Across the internet:*
- Multiple routers forward the packet based on routing tables and BGP
- Each hop: same IP header, different Ethernet frame (MAC changes hop-by-hop)

### Step 5: TLS Handshake
Since it's HTTPS, after TCP is established, a TLS handshake occurs:
1. **ClientHello** — browser says what TLS versions and ciphers it supports
2. **ServerHello** — Google chooses TLS 1.3, sends its certificate
3. Browser verifies certificate (not expired, valid for `*.google.com`, signed by trusted CA)
4. Key exchange — both sides agree on an encryption key
5. Encrypted channel established.

### Step 6: HTTP Request
Browser sends an encrypted HTTP/2 GET request over the TLS connection:
```
GET / HTTP/2
Host: www.google.com
User-Agent: Chrome/120...
Accept: text/html,...
Accept-Language: en-US
Cookie: ...
```

### Step 7: Google Processes the Request
Google's infrastructure:
- Load balancer receives the request
- Routes it to an available web server (or CDN edge node)
- Application processes the request
- Returns HTML response

### Step 8: HTTP Response
Google sends back:
```
HTTP/2 200 OK
Content-Type: text/html; charset=UTF-8
Cache-Control: private
Set-Cookie: ...

<!doctype html><html>...
```

### Step 9: Browser Renders the Page
Browser parses HTML → finds references to CSS, JS, images → makes additional HTTP requests for each → page renders.

The whole process from pressing Enter to first byte of content: typically 50–500ms depending on your location relative to Google's servers.

---

## 24. Quick Reference Cheat Sheet

### IP & Routing
```bash
ip addr show                  # Show IPs
ip route show                 # Show routes
ip route get 8.8.8.8          # Test which route for an IP
arp -n / ip neigh show        # ARP cache
```

### DNS
```bash
dig example.com               # DNS lookup
dig example.com +trace        # Full trace
dig @8.8.8.8 example.com      # Use specific resolver
cat /etc/resolv.conf          # DNS server config
cat /etc/hosts                # Local overrides
```

### Connections & Ports
```bash
ss -tlnp                      # TCP listening with processes
ss -tan                       # All TCP connections with states
lsof -i :80                   # What's on port 80
nc -zv host port              # Test if port is open
```

### Packet Capture
```bash
tcpdump -i eth0 port 80       # Capture HTTP traffic
tcpdump -i eth0 -w file.pcap  # Save to file
```

### HTTP
```bash
curl -v https://example.com   # Full request/response
curl -I https://example.com   # Headers only
curl -k https://example.com   # Skip TLS check
```

### TLS
```bash
openssl s_client -connect host:443   # Test TLS connection
openssl x509 -in cert.pem -text      # Read certificate
openssl x509 -in cert.pem -noout -enddate  # Check expiry
```

### Connectivity Testing
```bash
ping 8.8.8.8                  # Basic connectivity (ICMP)
traceroute google.com         # Trace path
mtr google.com                # Continuous traceroute (best for troubleshooting)
nmap -p 80,443 10.0.0.1      # Port scan
```

### Firewall
```bash
iptables -L -n -v             # View rules
ufw status                    # UFW status
firewall-cmd --list-all       # firewalld status
```

---

## Networking Concepts Map

```
Physical Medium (cables, Wi-Fi, fiber)
              ↓
       Ethernet Frames (MAC addresses — ARP resolves IP→MAC)
              ↓
       IP Packets (source/destination IP, TTL, routing)
              ↓
       TCP/UDP Segments (ports, reliability, sessions)
              ↓
       TLS (encryption layer — wraps HTTP)
              ↓
       HTTP/HTTPS (web), SSH (remote), DNS (name resolution),
       SMTP (email), DHCP (IP assignment), NTP (time)
              ↓
       Your Application

Supporting infrastructure:
  DNS         → translates names to IPs
  Routing     → gets packets from A to B
  NAT         → lets private IPs use the internet
  Firewalls   → controls what traffic is allowed
  Load Balancers → distributes traffic across servers
  CDN         → serves content from nearby edge nodes
```

---

*Document version 1.0 — Networking Fundamentals for DevOps Engineers. Pair this with the OS Fundamentals document for complete foundational understanding.*
