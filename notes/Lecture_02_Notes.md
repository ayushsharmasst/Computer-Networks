# Lecture 2: Network Packets and Layered Communication
## Student Notes — SST Computer Networks (Term 5)

---

## 🎯 Learning Objectives

By the end of this lecture, you should be able to:
- Define protocol and explain why layered architecture exists
- Name all 7 OSI layers and describe what each one is responsible for
- Map OSI layers to the TCP/IP model used in practice
- Trace the encapsulation and decapsulation process
- Distinguish between frames, packets, and segments
- Read and interpret an Ethernet frame structure
- Explain MTU and what fragmentation is
- Describe MAC addresses and their scope

---

## 1. What is a Protocol?

A **protocol** is a set of agreed-upon rules that govern communication between two or more parties. Both sender and receiver must follow the same protocol for communication to succeed.

In networking, a protocol defines:
- The **format** of messages (which fields appear, in what order, what each byte means)
- The **sequence** of operations (who initiates, who responds, in what order)
- The **actions** on specific events (what to do if a message is lost, corrupted, or arrives out of order)

**Why Layered Protocols?**

Rather than one massive protocol handling everything, networking uses a **layered architecture**. Each layer has one job; it relies on the layer below to handle the lower-level details and provides a service to the layer above.

Benefits:
- **Modularity**: Change one layer without touching others. Apple can update Safari's HTTP engine without modifying how TCP works.
- **Reusability**: TCP works identically whether it runs over Ethernet, Wi-Fi, or 4G.
- **Isolation**: Easier to diagnose problems. If TCP connects but the page doesn't load, the problem is in the application layer, not the network.

---

## 2. The OSI Model — The Universal Reference

In the 1980s, different networking vendors (IBM, DEC, etc.) had incompatible systems. The **ISO** (International Organization for Standardization) created the **OSI (Open Systems Interconnection) model** in 1984 as a universal reference framework.

The OSI model has **7 layers**. Every network function can be classified into one of these layers:

```
┌───┬──────────────┬──────────────────────────────────────────────────────┐
│ 7 │ Application  │ The rules for what applications communicate          │
│   │              │ Examples: HTTP (web), SMTP (email), DNS, SSH         │
├───┼──────────────┼──────────────────────────────────────────────────────┤
│ 6 │ Presentation │ Data formatting, compression, encryption             │
│   │              │ Examples: TLS/SSL, JPEG, gzip                        │
├───┼──────────────┼──────────────────────────────────────────────────────┤
│ 5 │ Session      │ Managing the lifecycle of a communication session    │
│   │              │ (largely absorbed into application/TCP layers today) │
├───┼──────────────┼──────────────────────────────────────────────────────┤
│ 4 │ Transport    │ End-to-end delivery, port numbers, reliability       │
│   │              │ Examples: TCP (reliable), UDP (fast, unreliable)     │
├───┼──────────────┼──────────────────────────────────────────────────────┤
│ 3 │ Network      │ Routing across multiple networks, IP addressing      │
│   │              │ Examples: IP (IPv4, IPv6), ICMP                      │
├───┼──────────────┼──────────────────────────────────────────────────────┤
│ 2 │ Data Link    │ Local delivery within one network, MAC addressing    │
│   │              │ Examples: Ethernet, Wi-Fi (802.11)                   │
├───┼──────────────┼──────────────────────────────────────────────────────┤
│ 1 │ Physical     │ Bits on the wire: voltages, radio waves, fiber light │
│   │              │ Examples: Cat6 cable, 802.11 radio specs             │
└───┴──────────────┴──────────────────────────────────────────────────────┘
```

**Mnemonic (bottom to top):** "**P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way"  
Physical → Data Link → Network → Transport → Session → Presentation → Application

### Each Layer Explained

**Layer 1 — Physical:** How bits travel physically. Voltages on copper wire, light pulses in fiber, radio waves for Wi-Fi. If your cable is unplugged: Layer 1 problem.

**Layer 2 — Data Link:** Organizes bits into **frames** and handles delivery on a single network. Addressing uses **MAC addresses** (hardware addresses). If your device is connected to the network but can't communicate with other devices on the same subnet: Layer 2 problem.

**Layer 3 — Network:** Handles routing **across networks**. Addressing uses **IP addresses**. When your packet needs to travel from your home network to a server in Germany, Layer 3 is what makes that possible. If you can reach your router but not the internet: Layer 3 problem.

**Layer 4 — Transport:** Adds **port numbers** (so data goes to the right application) and manages **reliability** (making sure data arrives complete and in order). Two main protocols: TCP (reliable) and UDP (fast, no guarantees). If everything connects but data is corrupted or lost: Layer 4/higher problem.

**Layer 5 — Session:** Manages connection sessions (opening, maintaining, closing). In modern systems, this is handled by the application or TCP. Exists in the model for historical completeness.

**Layer 6 — Presentation:** Data format translation, compression, and encryption. **TLS (HTTPS encryption) conceptually lives here.** If you can connect but communication is garbled or decryption fails: Layer 6 problem.

**Layer 7 — Application:** The protocols that user-facing applications use. HTTP for browsers, SMTP for email, DNS for name resolution. If the network is fine but the server returns an error: Layer 7 problem.

---

## 3. The TCP/IP Model — What We Actually Implement

The OSI model is a reference. What the internet actually uses is the **TCP/IP model**, developed by DARPA in the 1970s. It has 4 layers:

```
OSI Model                  TCP/IP Model
──────────────             ────────────────────────────────────────
7 — Application  ─┐
6 — Presentation  ├──────►  Application   (HTTP, DNS, SMTP, SSH...)
5 — Session      ─┘
4 — Transport    ────────►  Transport     (TCP, UDP)
3 — Network      ────────►  Internet      (IP, ICMP)
2 — Data Link    ─┐
1 — Physical     ─┘──────►  Network Access (Ethernet, Wi-Fi, LTE...)
```

**How we use these models together:**
- We **implement** TCP/IP protocols
- We **talk about** them using OSI layer numbers ("Layer 3 problem," "L2 switch")
- This course does the same: we study TCP/IP protocols, but use OSI numbers as vocabulary

---

## 4. Encapsulation and Decapsulation

### Encapsulation (Sender — going DOWN the stack)

As data travels from the application down to the physical medium, each layer **adds its own header** to the data. This is called **encapsulation**.

```
Application Data:   "GET / HTTP/1.1\r\nHost: google.com"
        ↓  [Layer 4 — TCP adds port numbers and sequence info]
TCP Segment:        [TCP Header: SrcPort=52341, DstPort=80, Seq=1001] + Data
        ↓  [Layer 3 — IP adds source and destination IP addresses]
IP Packet:          [IP Header: SrcIP=192.168.1.5, DstIP=142.250.195.46] + TCP Segment
        ↓  [Layer 2 — Ethernet adds MAC addresses for local delivery]
Ethernet Frame:     [ETH Header: DstMAC=Router's MAC, SrcMAC=Laptop's MAC] + IP Packet + [FCS]
        ↓
Bits transmitted on the wire / over Wi-Fi radio
```

### Decapsulation (Receiver — going UP the stack)

Each layer strips its own header and passes the payload up:

```
Ethernet Frame arrives
  → Layer 2 strips Ethernet header → extracts IP Packet
  → Layer 3 strips IP header → extracts TCP Segment
  → Layer 4 strips TCP header → extracts HTTP Data
  → Layer 7 reads the HTTP request
```

**Key insight:** Each layer treats everything from the layer above as opaque "payload" — just bytes to carry. It doesn't look inside. This is what allows HTTP to be updated without changing TCP, and TCP to be unchanged whether running over Wi-Fi or fiber.

**Why is the Ethernet destination the router, not Google?**

Notice in the encapsulation above: the Ethernet Dst MAC is the **router's MAC**, not Google's. This is because Ethernet only works on the local network. Your laptop doesn't know Google's MAC address — and even if it did, Google is not on your local network. So Ethernet delivers to your router. Your router then forwards the IP packet to the next hop, rebuilding a completely new Ethernet frame for that segment of the journey.

---

## 5. Headers and Payload

Every unit of data in networking has two parts:

```
┌──────────────────────────────────────────────────────┐
│       HEADER          │         PAYLOAD               │
│  (control information)│  (the actual data being sent) │
└──────────────────────────────────────────────────────┘
```

Typical header contents:
- Source and destination addresses (at that layer's scope)
- Length of the payload
- Type/protocol of the payload (so the receiver knows how to parse it)
- Sequence numbers (for ordering)
- Error-checking values
- Control flags (e.g., SYN, ACK in TCP)

**The nesting relationship:**
```
Ethernet Payload = IP Packet
IP Payload       = TCP Segment
TCP Payload      = HTTP Message
HTTP Payload     = the webpage HTML
```

---

## 6. Frames vs. Packets vs. Segments

These three terms describe data units at different OSI layers:

| Term | OSI Layer | Protocol | Address Type | Scope |
|------|-----------|----------|-------------|-------|
| **Segment** | L4 — Transport | TCP / UDP | Port numbers | End-to-end (process to process) |
| **Packet** | L3 — Network | IP | IP addresses | Host-to-host (across all networks) |
| **Frame** | L2 — Data Link | Ethernet / Wi-Fi | MAC addresses | Local network (one single hop) |

**What changes at each router hop:**

```
HOP 1: Your Laptop → Home Router
  Ethernet Frame:  [Src: Laptop MAC    → Dst: Router LAN MAC]
  IP Packet:       [Src: 192.168.1.5   → Dst: 142.250.195.46]  ← same

HOP 2: Home Router → ISP Router
  Ethernet Frame:  [Src: Router WAN MAC → Dst: ISP Router MAC]  ← REPLACED
  IP Packet:       [Src: 192.168.1.5   → Dst: 142.250.195.46]  ← still same

HOP 3: ISP Router → Google
  Ethernet Frame:  [Src: ISP MAC       → Dst: Google MAC]       ← REPLACED AGAIN
  IP Packet:       [Src: 192.168.1.5   → Dst: 142.250.195.46]  ← still same
```

**Rule:** MAC addresses change at every router hop. IP addresses stay the same all the way from source to destination (unless NAT is involved — covered in Lecture 11).

---

## 7. Ethernet Frame Structure

```
┌──────────┬──────────┬──────────┬────────────┬──────────────────┬────────┐
│ Preamble │ Dest MAC │ Src MAC  │ EtherType  │    Payload       │  FCS   │
│ 8 bytes  │  6 bytes │  6 bytes │  2 bytes   │  46–1500 bytes   │ 4 bytes│
└──────────┴──────────┴──────────┴────────────┴──────────────────┴────────┘
Min frame size: 64 bytes    Max frame size: 1518 bytes
```

**Field breakdown:**

**Preamble (7 bytes) + SFD (1 byte):** Synchronizes the receiver's clock and marks the exact start of the frame. The SFD (Start Frame Delimiter) byte signals "the real data begins right after me."

**Destination MAC (6 bytes):** 48-bit address of the intended recipient on this local network. Types:
- **Unicast**: one specific device (e.g., `AA:BB:CC:11:22:33`)
- **Broadcast**: all devices on the LAN (`FF:FF:FF:FF:FF:FF`)
- **Multicast**: a specific group of devices

**Source MAC (6 bytes):** 48-bit address of the sender.

**EtherType (2 bytes):** Identifies what protocol is in the payload — so the receiver knows which upper-layer protocol to hand the data to:

| Value | Payload type |
|-------|-------------|
| `0x0800` | IPv4 |
| `0x0806` | ARP |
| `0x86DD` | IPv6 |
| `0x8100` | VLAN-tagged frame |

**Payload (46–1500 bytes):** The data being carried (usually an IP packet). Minimum 46 bytes — if the actual payload is smaller, zeros are added as padding. (The 46-byte minimum comes from the need for frames to be at least 64 bytes total for collision detection on older shared Ethernet networks.)

**FCS — Frame Check Sequence (4 bytes):** A CRC (Cyclic Redundancy Check) over the entire frame. If the receiver's computed CRC doesn't match, the frame is **silently discarded** — no error message, no retransmission. Recovery from data loss is handled by Layer 4 (TCP).

---

## 8. MTU and Fragmentation

### Maximum Transmission Unit (MTU)

**MTU** is the largest payload that can be carried in a single frame on a given link type.

| Link Type | MTU |
|-----------|-----|
| Ethernet | 1500 bytes |
| Wi-Fi (in practice) | 1500 bytes (matches Ethernet for compatibility) |
| Jumbo frames (data center) | Up to 9000 bytes |
| Loopback interface | 65536 bytes |

### IP Fragmentation

When an IP packet is larger than the MTU of a link, the IP layer **fragments** it:

**Example: 4500-byte IP payload over Ethernet (MTU 1500)**

```
Fragment 1: [IP Hdr] [bytes    0–1479]  offset=0,   MF=1
Fragment 2: [IP Hdr] [bytes 1480–2959]  offset=185, MF=1
Fragment 3: [IP Hdr] [bytes 2960–4439]  offset=370, MF=1
Fragment 4: [IP Hdr] [bytes 4440–4499]  offset=555, MF=0  ← last
```

**IP header fields used for fragmentation:**

| Field | Purpose |
|-------|---------|
| **Identification** | All fragments of the same original packet share this 16-bit ID |
| **Fragment Offset** | Where this fragment starts in the original data (in 8-byte units) |
| **More Fragments (MF) bit** | 1 = more pieces coming; 0 = this is the last fragment |
| **Don't Fragment (DF) bit** | 1 = router must drop (not fragment) if too large; used in PMTUD |

**Why offset is measured in 8-byte units:** The offset field is 13 bits. 13 bits can hold values 0–8191. Multiplied by 8, that covers up to 65,528 bytes — which handles the maximum IP packet size of 65,535 bytes.

**Important rules:**
- Fragmentation can occur at any router along the path where the MTU is smaller
- **Reassembly happens ONLY at the final destination** — intermediate routers do not reassemble
- If even one fragment is lost, the destination discards all received fragments and the TCP layer (at the source) must retransmit the entire original data

### Path MTU Discovery (PMTUD)

Modern systems avoid fragmentation using PMTUD:
1. Send packets with DF = 1 (don't fragment)
2. If any router on the path has a smaller MTU, it drops the packet and sends back an ICMP "Fragmentation Needed, MTU = X" error
3. The sender adjusts packet size and retries
4. Repeat until optimal size is found for the full path

> Important: Blocking ICMP entirely breaks PMTUD. Large transfers mysteriously fail while small ones succeed — a common misconfiguration.

---

## 9. MAC Addresses

### What is a MAC Address?

A **MAC (Media Access Control) address** is a 48-bit hardware identifier assigned to every Network Interface Card (NIC).

**Format:** 6 bytes written as colon-separated hexadecimal pairs:
```
AA:BB:CC:DD:EE:FF
└─── OUI ────┘└── Device ID ────┘
  (3 bytes,      (3 bytes,
   manufacturer)  unique per device)
```

**OUI (Organizationally Unique Identifier):** First 3 bytes, assigned by IEEE to the manufacturer. You can look up any OUI to see which company made the device.

**How to find your MAC address:**
```bash
# Linux/macOS
ip link show           # look for "link/ether"

# Windows
ipconfig /all          # look for "Physical Address"
```

**Properties of MAC addresses:**

| Property | Detail |
|----------|--------|
| **Scope** | Local only — meaningful within one network segment (one hop) |
| **Uniqueness** | Globally unique in theory (assigned by manufacturer) |
| **Spoofing** | Can be changed in software — modern phones randomize MAC for privacy |
| **Purpose** | Identify devices for Layer 2 delivery on a local network |

### ARP — Connecting IP to MAC

IP addresses identify destinations globally. But Ethernet frames need MAC addresses for local delivery. **ARP (Address Resolution Protocol)** resolves this:

```
Your PC (192.168.1.5) needs to send to router (192.168.1.1):

Step 1: Broadcast on LAN (Dst MAC = FF:FF:FF:FF:FF:FF):
        "WHO HAS 192.168.1.1? Tell 192.168.1.5"

Step 2: Router hears the broadcast and replies (unicast):
        "192.168.1.1 is at 11:22:33:44:55:66"

Step 3: Your PC stores this in its ARP cache and uses
        11:22:33:44:55:66 as the Dst MAC in future frames
```

We cover ARP in full detail in **Lecture 11**.

---

## 10. A Complete Example — Sending a Web Request

**Scenario:** Your laptop at 192.168.1.5 sends a GET request to google.com (142.250.195.46).

```
Step 1 — Layer 7 (Application):
  Browser creates: GET / HTTP/1.1\r\nHost: google.com\r\n

Step 2 — Layer 4 (Transport):
  TCP adds: [SrcPort:52341 | DstPort:80 | Seq:1001 | ACK]

Step 3 — Layer 3 (Network):
  IP adds:  [SrcIP:192.168.1.5 | DstIP:142.250.195.46 | TTL:64]

Step 4 — Layer 2 (Data Link):
  Ethernet: [DstMAC:Router's MAC | SrcMAC:Laptop's MAC | EtherType:0x0800 | IP Packet | FCS]

Step 5 — Layer 1 (Physical):
  NIC transmits frame as electrical signals on cable (or radio waves over Wi-Fi)

Step 6 — At router (Layer 2 and 3):
  → Strips Ethernet header (it was only for local delivery)
  → Reads IP header: destination is 142.250.195.46
  → Looks up routing table → forward via WAN interface
  → Creates new Ethernet frame with new Src/Dst MACs for next hop

...multiple router hops later...

Step 7 — At Google's server (decapsulation):
  → Ethernet stripped → IP stripped → TCP stripped → HTTP request read
  → Server generates HTTP response
  → Response travels back the same way in reverse
```

---

## 📌 Key Takeaways

1. A **protocol** defines the rules; **layering** keeps rules modular and independently evolvable
2. **OSI model**: 7 layers — Physical, Data Link, Network, Transport, Session, Presentation, Application
3. **TCP/IP model**: 4 layers — Network Access, Internet, Transport, Application — this is what the internet actually uses
4. OSI layer numbers are used as vocabulary ("Layer 3 problem") even when implementing TCP/IP
5. **Encapsulation**: headers added going down the stack; **Decapsulation**: headers stripped going up
6. **Segment** (L4, ports) → **Packet** (L3, IPs) → **Frame** (L2, MACs)
7. **Ethernet frame** = Preamble (8B) + Dst MAC (6B) + Src MAC (6B) + EtherType (2B) + Payload (46–1500B) + FCS (4B)
8. **MTU** = 1500 bytes for Ethernet; larger IP packets are fragmented; reassembly at destination only
9. **MAC addresses** are 48-bit, local scope only — replaced at every router hop
10. **IP addresses** stay the same end-to-end (unless NAT); **MACs** change at every hop

---

## 🧠 Quick Self-Check Questions

1. What are the 7 OSI layers in order from bottom to top? What is the role of each?
2. How does the TCP/IP model map to the OSI model? Which OSI layers get collapsed into one TCP/IP layer?
3. Your laptop can ping the router (192.168.1.1) but cannot reach any website. Which OSI layer or layers are still working? Which might have a problem?
4. When your HTTP request travels from your laptop to Google, how many times does the Ethernet frame get replaced? Does the IP packet ever get replaced?
5. What is the EtherType field? What does the value `0x0806` mean?
6. An IP packet with 4500 bytes of payload must travel over an Ethernet link (MTU 1500). How many fragments will it be divided into? What are the offsets of each fragment?
7. Why does Ethernet require a minimum payload of 46 bytes? What happens if the actual data is smaller?
8. What is the difference between a Layer 2 switch and a Layer 3 router in terms of what addresses they look at to forward traffic?

---

*Lecture 2 of 13 — Computer Networks, Term 5, SST*
