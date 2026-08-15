# Lecture 2: Network Packets and Layered Communication
## Student Notes — SST Computer Networks (Term 5)

---

## 🎯 Learning Objectives

By the end of this lecture, you should be able to:
- Define protocol and explain why layered communication exists
- Trace the encapsulation and decapsulation process
- Distinguish between frames, packets, and segments
- Read and interpret an Ethernet frame structure
- Explain MTU and what fragmentation is
- Describe MAC addresses and their scope

---

## 1. What is a Protocol?

A **protocol** is a set of agreed-upon rules that govern communication between two or more parties. Both sender and receiver must understand and follow the same protocol for communication to succeed.

In networking, a protocol defines:
- The **format** of messages (what fields, what order)
- The **sequence** of operations (who speaks first, how to respond)
- The **actions** to take on certain events (what to do if a message is lost)

### Why Layered Protocols?

Rather than one massive protocol handling everything, networking uses a **layered architecture**:

| Layer | Concern | Example Protocol |
|-------|---------|-----------------|
| Application | What data means | HTTP, SMTP, DNS |
| Transport | Reliable delivery, ports | TCP, UDP |
| Network | Cross-network routing | IP |
| Data Link | Local network delivery | Ethernet, Wi-Fi |
| Physical | Bits on the wire | Electrical signals |

**Benefits of layering:**
- **Modularity**: Each layer can be changed independently
- **Reusability**: HTTP works over TCP regardless of whether TCP runs over Ethernet or Wi-Fi
- **Isolation**: Easier to debug — you can test each layer independently

---

## 2. Encapsulation and Decapsulation

### Encapsulation (Sender Side — going DOWN the stack)

As data travels from the application down to the physical medium, each layer **adds its own header** (and sometimes a trailer) to the data. This wrapping process is called **encapsulation**.

```
Application Data:   "GET / HTTP/1.1\r\nHost: google.com\r\n"
        ↓ TCP wraps it
TCP Segment:        [TCP Header][HTTP Data]
        ↓ IP wraps it
IP Packet:          [IP Header][TCP Header][HTTP Data]
        ↓ Ethernet wraps it
Ethernet Frame:     [ETH Header][IP Packet][ETH Trailer]
        ↓
Bits transmitted on wire
```

### Decapsulation (Receiver Side — going UP the stack)

Each layer **strips its own header** and passes the payload up:

```
Ethernet Frame received
  → ETH layer strips ETH header/trailer → IP Packet extracted
  → IP layer strips IP header → TCP Segment extracted
  → TCP strips TCP header → HTTP Data extracted
  → Application reads the HTTP request
```

**Key Insight:** Each layer treats the entire message from the layer above as just "data" (payload). It doesn't look inside — it just wraps or unwraps its own piece.

---

## 3. Headers and Payload

Every unit of data in networking has two parts:

```
┌──────────────────────────────────────────────────────┐
│       HEADER          │         PAYLOAD               │
│  (control information)│  (the actual data being sent) │
└──────────────────────────────────────────────────────┘
```

**Typical header contents:**
- Source and destination addresses
- Length of the payload
- Type/protocol of the payload
- Sequence numbers (for ordering)
- Error-checking values (checksums)
- Control flags

**The nesting relationship:**

```
Ethernet Payload = IP Packet
IP Payload       = TCP Segment
TCP Payload      = HTTP Message
HTTP Payload     = Your webpage HTML
```

---

## 4. Frames vs. Packets vs. Segments

These three terms describe data units at different layers:

| Term | Layer | Protocol | Address Type | Scope |
|------|-------|----------|-------------|-------|
| **Segment** | Transport (L4) | TCP / UDP | Port numbers | End-to-end (process to process) |
| **Packet** | Network (L3) | IP | IP addresses | Host-to-host (across networks) |
| **Frame** | Data Link (L2) | Ethernet / Wi-Fi | MAC addresses | Local network (one hop) |

### Critical Observation — Frame Changes at Every Hop

```
Laptop ────────────────→ Home Router ─────────────────→ ISP Router → ... → Google
[Frame A: Laptop→Router]  [Frame B: Router→ISP]  [Frame C: ISP→Google]
          [     Same IP Packet throughout               ]
                      [  Same TCP Segment throughout    ]
```

- The **Ethernet frame** is destroyed and recreated at each router
- The **IP packet** survives unchanged across all routers
- The **TCP segment** stays intact all the way to the destination server

This is why MAC addresses are called "local" — they're only meaningful for one hop.

---

## 5. Ethernet Frame Structure

```
┌──────────┬──────────┬──────────┬────────────┬──────────────────┬────────┐
│ Preamble │ Dest MAC │ Src MAC  │ EtherType  │    Payload       │  FCS   │
│ 8 bytes  │  6 bytes │  6 bytes │  2 bytes   │  46–1500 bytes   │ 4 bytes│
└──────────┴──────────┴──────────┴────────────┴──────────────────┴────────┘
```

### Field Breakdown

**Preamble (7 bytes) + SFD (1 byte):**  
Synchronization signal — alerts the receiver that a frame is arriving. SFD (Start Frame Delimiter) marks the exact start of the frame.

**Destination MAC (6 bytes):**  
48-bit address of the intended recipient on the local network. Can be:
- **Unicast**: specific device (e.g., `AA:BB:CC:11:22:33`)
- **Broadcast**: all devices (`FF:FF:FF:FF:FF:FF`)
- **Multicast**: a group of devices

**Source MAC (6 bytes):**  
48-bit address of the sender.

**EtherType (2 bytes):**  
Identifies the protocol of the payload:
- `0x0800` = IPv4
- `0x0806` = ARP
- `0x86DD` = IPv6

**Payload (46–1500 bytes):**  
The actual data (usually an IP packet). Minimum 46 bytes required; smaller payloads are padded with zeros.

**FCS — Frame Check Sequence (4 bytes):**  
A CRC (Cyclic Redundancy Check) computed over the frame. If the receiver's computed CRC doesn't match, the frame is **silently discarded**. Error recovery is handled by higher layers (TCP).

---

## 6. MTU and Fragmentation

### Maximum Transmission Unit (MTU)

**MTU** is the largest payload size that a network link can carry in a single frame.

| Network Type | MTU |
|-------------|-----|
| Ethernet | 1500 bytes |
| Wi-Fi (802.11) | 2304 bytes (but typically matches Ethernet's 1500) |
| Loopback (lo) | 65536 bytes |
| Typical internet path | 1500 bytes |

### Fragmentation

When an IP packet is larger than the MTU of the next link, the router **fragments** it into smaller pieces:

**Example: 4500-byte IP payload over Ethernet (MTU 1500)**

```
Original:     [IP Header][4500 bytes of data]

Fragment 1:   [IP Header][bytes 0–1479]    (offset=0,   MF=1)
Fragment 2:   [IP Header][bytes 1480–2959] (offset=185, MF=1)
Fragment 3:   [IP Header][bytes 2960–4439] (offset=370, MF=1)
Fragment 4:   [IP Header][bytes 4440–4499] (offset=555, MF=0)
```

**Key IP header fields for fragmentation:**

| Field | Purpose |
|-------|---------|
| Identification | All fragments of the same packet share the same ID |
| Fragment Offset | Position of this fragment in the original data (in 8-byte units) |
| More Fragments (MF) bit | 1 = more fragments to come; 0 = last fragment |
| Don't Fragment (DF) bit | 1 = do not fragment this packet (used in PMTUD) |

**Rules of fragmentation:**
- Fragmentation can happen at any router along the path
- **Reassembly happens ONLY at the destination** — no intermediate router reassembles
- If even one fragment is lost, TCP times out and the entire original segment must be retransmitted

### Path MTU Discovery (PMTUD)

Modern systems use **PMTUD** to avoid fragmentation:
1. Send packets with the DF bit set
2. If a router can't forward the packet (too large), it sends back an ICMP "Fragmentation Needed" message
3. The sender reduces packet size accordingly

---

## 7. MAC Addresses

### What is a MAC Address?

A **MAC (Media Access Control) address** is a hardware identifier assigned to every Network Interface Card (NIC) at manufacture time.

**Format:** 6 bytes (48 bits), written as colon-separated hex:
```
AA:BB:CC:DD:EE:FF
└── OUI ──┘└── Device ID ──┘
```

**OUI (Organizationally Unique Identifier):** First 3 bytes, assigned by IEEE to the manufacturer.
**Device ID:** Last 3 bytes, assigned by the manufacturer to uniquely identify the device.

### Properties of MAC Addresses

| Property | Detail |
|---------|--------|
| **Scope** | Local — only meaningful within one network segment |
| **Uniqueness** | Globally unique (in theory; assigned by manufacturer) |
| **Persistence** | Burned into hardware, but can be software-spoofed |
| **Purpose** | Identify devices on a local Ethernet/Wi-Fi network |

### How to find your MAC address

```bash
# Linux/macOS
ip link show
# or
ifconfig

# Windows
ipconfig /all
# Look for "Physical Address"
```

### MAC ↔ IP Mapping: A Preview of ARP

IP addresses are used for routing across networks, but Ethernet frames need MAC addresses. **ARP (Address Resolution Protocol)** bridges this gap:

```
Device A (192.168.1.5) wants to reach Device B (192.168.1.10):

1. A broadcasts: "Who has 192.168.1.10? Tell 192.168.1.5"
   (Destination MAC = FF:FF:FF:FF:FF:FF)

2. B replies: "192.168.1.10 is at CC:DD:EE:FF:00:11"

3. A caches this mapping in its ARP table
   and uses CC:DD:EE:FF:00:11 in future frames to B
```

---

## 8. Putting It All Together — A Complete Example

**Scenario:** Your laptop sends an HTTP request to google.com.

```
Step 1: Browser creates HTTP request
        GET / HTTP/1.1\r\nHost: google.com\r\n

Step 2: OS wraps in TCP segment
        [SrcPort:52341 | DstPort:80 | Seq:1001 | HTTP data]

Step 3: OS wraps in IP packet
        [SrcIP:192.168.1.5 | DstIP:142.250.195.46 | TCP segment]

Step 4: NIC driver wraps in Ethernet frame
        [DstMAC:Router's MAC | SrcMAC:Your NIC's MAC | IP packet | FCS]

Step 5: NIC sends frame as electrical signals / radio waves

Step 6: Router receives frame, strips Ethernet header
        → Reads IP packet → Looks up routing table
        → Creates new Ethernet frame for next hop
        → Forwards toward Google

...many hops later...

Step 7: Google's server receives the frame
        → Strips Ethernet → Strips IP → Strips TCP → Reads HTTP
        → Sends HTTP response back
```

---

## 📌 Key Takeaways

1. **Protocols** define rules for communication; **layering** keeps those rules modular and reusable
2. **Encapsulation** adds headers going down the stack; **decapsulation** removes them going up
3. **Segment (L4) → Packet (L3) → Frame (L2)** — each has its own addressing scope
4. **Ethernet frame** = Preamble + Dest MAC + Src MAC + EtherType + Payload (46–1500 bytes) + FCS
5. **MTU** = 1500 bytes for Ethernet; larger data gets **fragmented** by IP
6. **MAC addresses** are 48-bit hardware addresses, local in scope, discovered via ARP
7. Frames change at every router hop; IP packets survive the entire journey

---

## 🧠 Quick Self-Check Questions

1. A TCP segment is the payload of which layer's data unit?
2. What does the EtherType field value `0x0800` indicate?
3. Why is the minimum Ethernet frame payload 46 bytes (not 0)?
4. What does the MF bit in an IP header mean?
5. Your application sends 3000 bytes. How many Ethernet frames will this require, approximately?
6. If a router drops a fragment, does Ethernet re-send it? Explain what actually happens.
7. What's the scope of a MAC address? Can you use a MAC address to route to a server in another country?

---

*Lecture 2 of 13 — Computer Networks, Term 5, SST*
