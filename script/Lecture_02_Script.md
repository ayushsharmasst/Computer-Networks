# Lecture 2: Network Packets and Layered Communication
## Instructor Script — SST Computer Networks (Term 5)

---

### 🎯 Learning Objectives
By the end of this session, students will be able to:
- Explain what a protocol is and why layered architecture exists
- Name all 7 OSI layers and describe what each one does
- Map OSI layers to the TCP/IP model used in practice
- Trace how data is encapsulated as it travels down the network stack
- Distinguish between frames, packets, and segments
- Describe the structure of an Ethernet frame
- Explain MTU and what happens when data exceeds it

**Duration:** ~95 minutes  
**Teaching Style:** Practical-first, analogy-driven  
**Demo Materials:** Wireshark (or screenshots), `ip link show` on any Linux machine

---

## 📦 OPENING HOOK (5 minutes)

**[INSTRUCTOR: Start with this scenario. Write on the board: "You send a WhatsApp message. What actually happens?"]**

Say to students:

> "Yesterday I sent a 47 MB file to a friend on WhatsApp. He's in Bangalore. How did those 47 megabytes physically travel? Was it sent as one giant chunk? Was it broken up? And — critically — who is responsible for what part of that journey?"

Take 2–3 hands. Let students speculate. Then say:

> "The answer is that the data was broken into **thousands of small pieces**. Each piece was wrapped in multiple envelopes. Each envelope was handled by a different 'department' that has its own job and its own rules. On the other end, those departments unwrapped in reverse order. This system of departments is what we're learning today — and it's called a **layered network architecture**."

---

## SECTION 1: What is a Protocol? (8 minutes)

**[INSTRUCTOR: Draw two stick figures on the board — one labelled "Your Device" and one labelled "Server".]**

### Analogy — Language as a Protocol

> "If I call someone and speak in Hindi but they only understand Tamil, what happens?"

**Answer:** Communication fails — not because the phone line is broken, but because the rules of communication don't match.

> "A **protocol** is simply an agreed-upon set of rules for communication. Both sides must follow the same format, the same ordering, and the same actions for every situation."

Write on board:
```
Protocol = Rules that define:
  - FORMAT of messages (what bytes mean what)
  - ORDER of messages (who speaks first, who responds how)
  - ACTIONS on events (what to do when something goes wrong)
```

**Examples of protocols:**
- HTTP: Rules for requesting and responding to web pages
- SMTP: Rules for sending email
- TCP: Rules for reliable, ordered delivery
- IP: Rules for addressing and routing across networks

> "Notice that each of these protocols has a specific, limited job. HTTP doesn't care how it gets delivered reliably — that's TCP's problem. TCP doesn't care which city the server is in — that's IP's problem. This separation of responsibilities is the key idea of **layered architecture**."

**[PAUSE: Ask — "Why do you think we separate responsibilities into layers instead of having one giant protocol that does everything?"  
Take 2–3 answers. Expected: easier to change one piece, easier to debug, can mix and match.]**

**Correct answer to formalize:**
- **Modularity:** Change one layer without touching others. Apple can update Safari's HTTP handling without touching IP.
- **Reuse:** TCP works identically whether the network underneath is Ethernet, Wi-Fi, or 4G. It doesn't know or care.
- **Debugging:** If a web page doesn't load, you can isolate the problem layer by layer: is DNS working? Is TCP connecting? Is HTTP returning an error?

---

## SECTION 2: The OSI Model — The 7-Layer Mental Map (18 minutes)

**[INSTRUCTOR: This is the most important section of this lecture. The OSI model is the vocabulary we'll use for the next 11 lectures. Take your time here. Draw the 7-layer stack on the board and fill it in as you explain each layer.]**

### Why Do We Need a Model?

> "In the 1970s and 80s, different companies made networking equipment that couldn't talk to each other. IBM equipment couldn't communicate with DEC equipment. Each vendor had their own proprietary protocol stack."

> "In 1984, the International Organization for Standardization (ISO) published the **OSI model** — Open Systems Interconnection. Its goal was to define a standard framework so that ANY vendor's equipment could interoperate. You can think of it as the 'reference constitution' of networking."

> "Today, the OSI model is not what we actually implement — we use the TCP/IP stack, which we'll see in a moment. But OSI is what we USE TO TALK ABOUT networking. When an engineer says 'that's a Layer 3 problem,' every engineer in the room knows exactly what they mean. You need to know this vocabulary."

### The 7 Layers — Draw on Board from Bottom to Top

Draw this on the board:

```
┌─────────────────────────────────────────────────────────┐
│  7 — Application    │ What the user's software does     │
├─────────────────────┼───────────────────────────────────┤
│  6 — Presentation   │ Data format, compression, encrypt │
├─────────────────────┼───────────────────────────────────┤
│  5 — Session        │ Managing connections/sessions      │
├─────────────────────┼───────────────────────────────────┤
│  4 — Transport      │ Reliable delivery, ports           │
├─────────────────────┼───────────────────────────────────┤
│  3 — Network        │ Routing, IP addresses              │
├─────────────────────┼───────────────────────────────────┤
│  2 — Data Link      │ Local delivery, MAC addresses      │
├─────────────────────┼───────────────────────────────────┤
│  1 — Physical       │ Electrical signals, cables, bits   │
└─────────────────────────────────────────────────────────┘
```

**Walk through each layer now — one at a time:**

---

**Layer 1 — Physical**

> "The physical layer is the most literal. It answers: 'How do bits travel from one device to another?'"

> "A bit needs to be represented physically — as a voltage on a wire, a light pulse in a fiber optic cable, or a radio wave in Wi-Fi. The physical layer defines exactly what a '0' and a '1' look like as physical signals, how fast they travel, what the connector pins do."

> "Examples: Ethernet cables (Cat5/Cat6), fiber optic cables, Wi-Fi radio frequency specs. When your cable is unplugged or your Wi-Fi signal is too weak, you have a Layer 1 problem."

---

**Layer 2 — Data Link**

> "The physical layer sends bits, but bits without structure are just noise. The data link layer organizes those bits into structured units called **frames**, and handles delivery within a single local network."

> "It answers: 'On this specific network segment — this room, this building's Wi-Fi — how do I get data from one device to the specific device I want?' The addressing here is **MAC addresses** — hardware addresses burned into your network card."

> "Protocols at Layer 2: Ethernet, Wi-Fi (802.11), Bluetooth. Devices at Layer 2: switches (they forward frames based on MAC addresses)."

> "Layer 2 problems: wrong VLAN configuration, duplicate MAC addresses, STP issues, 'connected but can't communicate' on the same network."

---

**Layer 3 — Network**

> "Layer 2 handles delivery on ONE network. But the internet connects millions of networks. How does data travel from your home network to a server in Germany?"

> "Layer 3 handles addressing across networks. The addressing scheme here is **IP addresses** — we'll spend all of Lectures 3 and 4 on this. Layer 3 handles **routing** — the process of figuring out which path a packet should take across multiple networks to reach its destination."

> "Protocols at Layer 3: IP (IPv4, IPv6), ICMP. Devices at Layer 3: routers (they forward packets based on IP addresses)."

> "Layer 3 problems: wrong IP address, wrong subnet mask, missing default gateway, routing table issues."

---

**Layer 4 — Transport**

> "Layers 1–3 can get a packet from point A to point B. But there's no guarantee it arrived. It might arrive out of order. Multiple packets might be lost. And your computer is running dozens of apps — email, browser, Slack — all using the network simultaneously. Which app should receive which data?"

> "Layer 4 — the Transport layer — handles all of this. It adds **port numbers** (so data goes to the right app) and manages **reliable delivery** (making sure data arrives complete, in order, without duplicates)."

> "Protocols: **TCP** (Transmission Control Protocol — reliable, ordered) and **UDP** (User Datagram Protocol — fast, no guarantees). We spend all of Lecture 10 on these."

> "Layer 4 problems: port blocked by firewall, connection refused (nothing listening on that port), packet loss causing retransmissions."

---

**Layer 5 — Session**

> "Session layer manages the lifecycle of a communication session — opening, maintaining, and closing a connection. In practice today, this is mostly handled by Layer 7 applications and the TCP layer below, so you won't see 'Session' as a distinct protocol in modern stacks. It exists in the model for historical reasons."

---

**Layer 6 — Presentation**

> "If the sender and receiver use different formats for the same data — different character encodings, different ways of representing numbers, different file compression formats — they won't understand each other even if the bits arrive correctly."

> "The presentation layer handles data **translation**, **compression**, and **encryption**. In practice today, this is also largely collapsed into the application layer. However, **TLS/SSL — the encryption that makes HTTPS work — conceptually lives here.** We'll cover TLS in Lecture 13."

---

**Layer 7 — Application**

> "This is what your users actually interact with. The application layer defines the protocols that applications use to communicate — HTTP for web browsing, SMTP for email, DNS for name resolution, FTP for file transfer."

> "This is NOT the app itself — it's the protocol the app uses. Chrome is not Layer 7; HTTP is Layer 7."

---

**[INSTRUCTOR: Pause and check understanding. Ask: "If your web browser can't load any pages but you can ping Google's IP address — what layer is the problem at?" Expected answer: Layer 7 (HTTP/DNS) — Layer 3 (ping/IP) is working.]**

---

### The TCP/IP Model — What We Actually Use

> "The OSI model is our reference map. But what engineers actually implement is the **TCP/IP model** — also called the Internet model. It was developed by DARPA in the 1970s before OSI existed, and it's what the entire internet runs on."

Draw the comparison:

```
OSI Model                 TCP/IP Model
─────────────             ─────────────────
7 — Application  ─┐
6 — Presentation  ├───►  Application
5 — Session      ─┘
4 — Transport    ─────►  Transport
3 — Network      ─────►  Internet
2 — Data Link    ─┐
1 — Physical     ─┘───►  Network Access
```

> "TCP/IP collapses the top three OSI layers into one (Application), and the bottom two into one (Network Access). In practice:"

| TCP/IP Layer | What it handles | Key protocols |
|---|---|---|
| Application | Everything user-facing | HTTP, DNS, SMTP, SSH |
| Transport | End-to-end delivery, ports | TCP, UDP |
| Internet | Routing across networks | IP, ICMP |
| Network Access | Local delivery + physical | Ethernet, Wi-Fi |

> "For this course, we use OSI layer numbers as vocabulary — 'Layer 3 problem', 'Layer 2 switch' — but we study TCP/IP protocols in practice. This is industry standard."

**Mnemonic for OSI layers bottom to top:**  
**"Please Do Not Throw Sausage Pizza Away"**  
Physical → Data Link → Network → Transport → Session → Presentation → Application

---

## SECTION 3: Encapsulation and Decapsulation (12 minutes)

**[INSTRUCTOR: Now that students know what each layer does, the encapsulation concept will make immediate sense. Draw the layered stack on the board and build it live.]**

### The Postal Analogy

> "You want to send a confidential document internationally. You put the document in an envelope. You put that envelope in a padded box. You hand the box to a courier who puts it in a shipping container. Each wrapper adds addressing information relevant to THAT level of delivery — the envelope has the recipient's name, the box has a building address, the shipping container has a country code."

> "Encapsulation in networking works exactly the same way."

Draw on board, building top to bottom:
```
APPLICATION DATA:     "GET /index.html HTTP/1.1\r\nHost: google.com"
        ↓  [TCP wraps it with port numbers and sequence numbers]
TCP SEGMENT:          [TCP Header: Src Port 52341, Dst Port 80, Seq=1001] + Data
        ↓  [IP wraps it with IP addresses]
IP PACKET:            [IP Header: Src=192.168.1.5, Dst=142.250.195.46] + TCP Segment
        ↓  [Ethernet wraps it with MAC addresses]
ETHERNET FRAME:       [ETH Header: Src MAC AA:BB, Dst MAC 11:22] + IP Packet + [FCS]
        ↓
Bits on the wire ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

**Say:**
> "This process — wrapping data with a header at each layer as it travels DOWN the stack — is called **encapsulation**. Each layer adds its own header and treats everything from the layer above as just 'payload' — just raw data it doesn't need to understand."

> "At the receiving end, the process reverses. Ethernet removes its header and passes the IP packet up. IP removes its header and passes the TCP segment up. TCP removes its header and passes the HTTP message to your browser. This reverse process is called **decapsulation**."

**Critical observation:**
> "Notice that the Ethernet header says 'Dst MAC 11:22.' That's your ROUTER's MAC address — not Google's. Why? Because Ethernet is only for LOCAL delivery. Once the frame reaches your router, the router strips the Ethernet header, reads the IP packet, figures out where to send it next, and puts it in a NEW Ethernet frame addressed to the NEXT hop. This happens at every router along the path. The IP addresses inside stay the same all the way. The Ethernet header is replaced at every hop."

### Live Walkthrough — Sending a GET request

Step through each layer when you visit google.com:

**Layer 7 (Application — HTTP):**
```
Browser constructs:
GET / HTTP/1.1
Host: www.google.com
```

**Layer 4 (Transport — TCP):**
```
OS adds:
  Source Port: 52341  (randomly chosen, 1024–65535)
  Destination Port: 80  (well-known port for HTTP)
  Sequence Number: 1001
  Flags: PSH, ACK
```

**Layer 3 (Network — IP):**
```
OS adds:
  Source IP: 192.168.1.5  (your laptop)
  Destination IP: 142.250.195.46  (Google)
  TTL: 64
```

**Layer 2 (Data Link — Ethernet):**
```
Network driver adds:
  Source MAC: AA:BB:CC:DD:EE:FF  (your laptop's NIC)
  Destination MAC: 11:22:33:44:55:66  (your ROUTER — not Google!)
  EtherType: 0x0800 (IPv4)
```

**[INSTRUCTOR: Stop here. Ask: "Why is the Ethernet destination address your router and not Google?" Let 2–3 students answer. Key insight: MAC addresses only work on the local network. Your laptop doesn't know Google's MAC — and Google is not on your local network. So Ethernet delivers to your router, and the router takes responsibility from there.]**

---

## SECTION 4: Headers and Payload (6 minutes)

Draw a simple box:

```
┌──────────────────────────────────────────────────────────┐
│  HEADER                │  PAYLOAD (DATA)                  │
│  (control information) │  (the content being delivered)   │
└──────────────────────────────────────────────────────────┘
```

**Ask students:** "What kind of information would you expect to find in a header?"

**Build the answer:**
- Source and destination addresses (who sent it, who should get it)
- Length of the payload
- Type of content inside (so the receiver knows how to parse it)
- Error-checking information
- Sequence numbers (for ordering pieces)
- Control flags (like SYN/ACK in TCP)

**Key insight:**
> "The payload of one layer is the ENTIRE message of the layer above it. The IP packet doesn't know or care that its payload is a TCP segment containing HTTP data. To IP, it's just bytes. This is intentional — it's what allows each layer to evolve independently. HTTPS was invented decades after IP, but IP happily carries it because it just sees it as payload."

---

## SECTION 5: Frames vs Packets vs Segments (8 minutes)

**[INSTRUCTOR: Write these three terms on the board with their layer numbers. Students should see that each term has a specific technical meaning — they're not interchangeable.]**

| Term | OSI Layer | Protocol Examples | Addressing Used |
|------|-----------|------------------|----------------|
| Segment | L4 — Transport | TCP, UDP | Port numbers |
| Packet | L3 — Network | IP | IP addresses |
| Frame | L2 — Data Link | Ethernet, Wi-Fi | MAC addresses |

**Walk through the "scope" of each:**

**Segment:** "A TCP segment carries your data with port numbers. Port 80 = HTTP, Port 443 = HTTPS, Port 22 = SSH. The port tells the OS which application on the destination machine should receive this data. Without ports, your browser and your SSH client would both receive everything and wouldn't know what belongs to them."

**Packet:** "An IP packet carries the TCP segment and adds city-level addressing — IP addresses. A packet can travel across many networks: from your laptop to your router, through your ISP, across the internet, to the destination server. The IP addresses don't change along the way."

**Frame:** "A frame is strictly local. It carries the IP packet across ONE network segment — from your laptop to your router, or from a router to the next router. At every hop, the old frame is discarded and a new one is created for the next segment of the journey."

**Demonstration — What changes at each hop?**

```
HOP 1: Your Laptop → Home Router
  Ethernet Frame:  [Src: Laptop MAC] → [Dst: Router LAN MAC]
  IP Packet:       [Src: 192.168.1.5] → [Dst: 142.250.195.46]
  (same all the way)

HOP 2: Home Router → ISP Router
  Ethernet Frame:  [Src: Router WAN MAC] → [Dst: ISP MAC]  ← CHANGED
  IP Packet:       [Src: 192.168.1.5] → [Dst: 142.250.195.46]  ← SAME

HOP 3: ISP Router → Google's Server
  Ethernet Frame:  [Src: ISP MAC] → [Dst: Google Server MAC]  ← CHANGED AGAIN
  IP Packet:       [Src: 192.168.1.5] → [Dst: 142.250.195.46]  ← SAME
```

> "MAC addresses are replaced at every hop. IP addresses stay the same. This is the fundamental contract between Layer 2 and Layer 3."

**[PAUSE: Ask — "If IP is the same all the way, why do we need Ethernet at all? Why not just use IP?"  
Answer: IP doesn't define how to physically send bits across a cable. IP says where to go; Ethernet and Wi-Fi say how to physically get to the next hop.]**

---

## SECTION 6: Ethernet Frame Structure (10 minutes)

**[INSTRUCTOR: Draw the Ethernet frame structure on the board. Go field by field.]**

```
┌──────────┬──────────┬──────────┬──────────────┬───────────────┬─────────┐
│ Preamble │ Dst MAC  │ Src MAC  │  EtherType   │    Payload    │   FCS   │
│  8 bytes │  6 bytes │  6 bytes │   2 bytes    │  46–1500 bytes│ 4 bytes │
│(7+1 SFD) │          │          │              │               │         │
└──────────┴──────────┴──────────┴──────────────┴───────────────┴─────────┘
Total minimum: 64 bytes    Total maximum: 1518 bytes
```

**Walk through each field with an analogy:**

**Preamble (7 bytes) + SFD (1 byte):**
> "Think of the preamble like a ringing bell before an announcement — it synchronizes the receiver's clock so it knows when the actual data starts. The last byte is the Start Frame Delimiter (SFD), which says 'the frame itself starts right after me.'"

**Destination MAC (6 bytes):**
> "A 48-bit hardware address identifying the intended recipient on this local network. Could be a specific device (unicast), all devices on the network (broadcast: FF:FF:FF:FF:FF:FF), or a specific group (multicast)."

**Source MAC (6 bytes):**
> "48-bit hardware address of the sender."

**EtherType (2 bytes):**
> "Tells the receiver what type of data is in the payload, so it knows which protocol to hand it to. Common values:"
```
0x0800 = IPv4 (this is an IP packet)
0x0806 = ARP  (Address Resolution Protocol)
0x86DD = IPv6
0x8100 = VLAN-tagged frame
```
> "This is like writing 'FRAGILE - GLASS' or 'ELECTRONIC COMPONENTS' on a shipping box — the handler knows what it is before opening it."

**Payload (46–1500 bytes):**
> "The IP packet. The minimum of 46 bytes is interesting — Ethernet was designed for shared cables where two devices could transmit simultaneously and cause a collision. For collision detection to work reliably, frames had to be long enough to still be on the wire when the collision signal came back. That required a minimum frame size of 64 bytes — so with 18 bytes of headers/trailer, the payload minimum is 46 bytes. On modern switched networks, collisions don't happen, but the minimum is kept for compatibility."

**FCS — Frame Check Sequence (4 bytes):**
> "A CRC (Cyclic Redundancy Check) computed over the entire frame. The sender computes it and puts it here. The receiver recomputes it from the received data. If they don't match, the frame was corrupted in transit and is **silently discarded**. No error message. No retransmission. Ethernet doesn't retry."

**[PAUSE — Ask: "If Ethernet silently discards corrupted frames, who is responsible for detecting that data was lost and asking for it again?"  
Answer: TCP, at Layer 4. This is a beautiful example of separation of responsibilities — Layer 2 detects and discards corruption; Layer 4 detects and recovers from loss.]**

---

## SECTION 7: MTU and Fragmentation (10 minutes)

### What is MTU?

> "We just saw that the Ethernet payload can be at most **1500 bytes**. This limit is called the **MTU — Maximum Transmission Unit**. It's the largest single chunk of data that a given network technology can carry in one frame."

Different link types, different MTUs:

| Link Type | MTU |
|---|---|
| Ethernet | 1500 bytes |
| Wi-Fi (802.11) | up to 2304 bytes (but usually 1500 for compatibility) |
| Jumbo Frames (data center Ethernet) | up to 9000 bytes |
| IPv6 minimum | 1280 bytes |

### What Happens When Data is Too Big?

> "Suppose your application wants to send 5000 bytes of data. That's too big for a single Ethernet frame. The solution: the IP layer **fragments** the data — breaks it into multiple IP packets, each small enough to fit in a frame."

Draw on board:
```
Original: 5000 bytes of data

Fragment 1: bytes 0–1479     (1480 bytes payload, offset = 0,   MF = 1)
Fragment 2: bytes 1480–2959  (1480 bytes payload, offset = 185, MF = 1)
Fragment 3: bytes 2960–4439  (1480 bytes payload, offset = 370, MF = 1)
Fragment 4: bytes 4440–4999  (560 bytes payload,  offset = 555, MF = 0)
```

**IP Header fields used for fragmentation:**

| Field | Meaning |
|---|---|
| Identification | All fragments of the same original packet share this ID |
| Fragment Offset | Where in the original packet this fragment starts (in 8-byte units) |
| More Fragments (MF) bit | 1 = more pieces coming; 0 = this is the last one |
| Don't Fragment (DF) bit | If set, drop and send ICMP error instead of fragmenting |

**Why offset in 8-byte units?** The offset field is 13 bits. 13 bits × 8 = up to 65,528 bytes of offset, which covers the maximum IP packet size of 65,535 bytes.

**Critical note on reassembly:**
> "IP fragments are reassembled only at the DESTINATION. Intermediate routers do NOT reassemble. If even one fragment is lost anywhere along the path, the destination discards ALL fragments it received and the entire data has to be retransmitted — at the TCP layer."

### Path MTU Discovery (PMTUD)

> "Modern networks avoid fragmentation entirely using **Path MTU Discovery**. Here's how it works:"

```
1. Sender sets DF (Don't Fragment) bit = 1
2. Sends packet at 1500 bytes
3. If a router on the path has a smaller MTU, it drops the packet
   and sends back an ICMP "Fragmentation Needed" message with its MTU
4. Sender learns the smaller MTU and resizes future packets
5. Process repeats until the optimal size is found
```

> "This is why blocking ICMP completely is bad practice — you'll break PMTUD and mysteriously large transfers will fail while small ones succeed."

---

## SECTION 8: MAC Addresses (6 minutes)

### What is a MAC Address?

> "MAC stands for **Media Access Control**. It's a 48-bit address assigned to every network interface — your Wi-Fi card, your Ethernet port, even your Bluetooth adapter all have different MACs."

**Format:**
```
AA:BB:CC:DD:EE:FF
│── OUI (24 bits) ──│── Device ID (24 bits) ──│
```

**OUI = Organizationally Unique Identifier** — IEEE assigns these to manufacturers. The first 3 bytes tell you who made the device:
- `00:1A:2B` → specific manufacturer
- `F4:5C:89` → another manufacturer

**Run on Linux:**
```bash
ip link show
# Look for "link/ether" — that's your MAC address
```

**Key properties:**
1. **Globally unique** (in theory — manufacturers guarantee no two devices ship with the same MAC)
2. **Local scope only** — meaningful only on the current network segment
3. **Replaced at every router hop** — the frame is re-created each hop with new MACs
4. **Can be spoofed** — software can change what MAC your card advertises (privacy feature in modern phones)

### ARP — Connecting IP to MAC

> "Your OS knows the destination IP address (from DNS). But Ethernet needs a MAC address. How do you find the MAC address of the router when you only know its IP (192.168.1.1)?"

> "**ARP — Address Resolution Protocol** — is what bridges IP addresses to MAC addresses on a local network. We'll cover ARP in full in Lecture 11, but here's the preview:"

```
Your PC wants to reach 192.168.1.1 (router):

1. Broadcast on the LAN (to FF:FF:FF:FF:FF:FF):
   "WHO HAS 192.168.1.1? Tell 192.168.1.5"

2. The router (192.168.1.1) hears this broadcast and replies:
   "192.168.1.1 is at 11:22:33:44:55:66"

3. Your PC caches this mapping and now builds valid Ethernet frames
```

---

## SECTION 9: Seeing It All Together — Wireshark (5 minutes)

**[INSTRUCTOR: If possible, do a live Wireshark capture of a simple HTTP request to a non-HTTPS site. If not, show a screenshot. Point out the nested layers.]**

> "Wireshark captures every frame on the network and lets you expand the headers layer by layer. Let's look at what we've been talking about in practice."

Point out in Wireshark:
- **Frame** (Wireshark's own info — arrival time, interface)
- **Ethernet II** — Src/Dst MAC, EtherType 0x0800
- **Internet Protocol Version 4** — Src/Dst IP, TTL, Protocol (6 = TCP)
- **Transmission Control Protocol** — Src/Dst Port, Seq/Ack, Flags
- **Hypertext Transfer Protocol** — the actual HTTP request

> "This nested view IS the encapsulation stack we drew on the board. Every single web request looks like this."

---

## 📝 CLASS DISCUSSION (3 minutes)

> "You're on 4G mobile data instead of home Wi-Fi. Does the HTTP, TCP, and IP you're using change?"

**Expected answer:** HTTP and TCP: no change at all. IP: no change. What changes is Layer 1 and Layer 2 — the physical radio signal and the LTE frame format. But those are transparent to everything above. **This is the power of layering.**

> "Apple didn't need to rewrite Safari when 5G was invented. Google didn't rewrite their servers. Only the radio hardware and the L1/L2 driver changed."

---

## SUMMARY (3 minutes)

Write on board:

```
✅ Protocol = agreed rules for format, order, and actions
✅ OSI Model — 7 layers (Physical → Data Link → Network → Transport → Session → Presentation → Application)
✅ TCP/IP Model — 4 layers (Network Access → Internet → Transport → Application)
✅ Encapsulation = wrapping data with headers going DOWN the stack
✅ Decapsulation = unwrapping headers going UP the stack
✅ Segment (L4, ports) → Packet (L3, IPs) → Frame (L2, MACs)
✅ Ethernet frame: Preamble | Dst MAC | Src MAC | EtherType | Payload (46–1500 B) | FCS
✅ MTU = 1500 bytes for Ethernet; larger data gets fragmented
✅ MAC = 48-bit hardware address, local scope only, changes at every hop
✅ IP = same all the way source to destination
```

---

## 🔗 Preview of Next Lecture

> "Now we know HOW data travels in packets and what the addressing looks like at Layer 2. But IP — the Layer 3 addressing — is the real foundation of the entire internet. Next lecture, we'll learn how IP addresses work: how to read them in binary, what subnets are, and how a router decides which path to send your packet down. Everything in this course from Lecture 3 onwards builds on IP addressing."

---

## ❓ Potential Student Questions

**Q: The OSI model has 7 layers but TCP/IP has 4. Which one do we actually use?**  
A: We implement TCP/IP in practice. We use OSI as the reference vocabulary. When engineers say "Layer 3 problem" or "this is a L2 switch," they're using OSI numbering even though the actual protocols are TCP/IP. You need to know both.

**Q: Does the router read the HTTP headers?**  
A: A basic router at Layer 3 only reads IP headers — it doesn't look inside. But modern firewalls and load balancers can do "deep packet inspection" and read all the way up to application data. This is why HTTPS (encrypted) is important — it prevents middle boxes from reading your HTTP content.

**Q: What if a fragment arrives out of order?**  
A: The destination collects all fragments (identified by the same IP Identification field) and reassembles them using the offset values. Order doesn't matter — offsets tell it exactly where each piece goes.

**Q: Can we change the MTU?**  
A: Yes. `ip link set eth0 mtu 9000` enables "jumbo frames" (up to 9000 bytes) on Ethernet — common in data centers for performance. But all devices on the path must support the larger MTU, or you'll get mysterious failures.

**Q: Why is Wi-Fi's frame format different from Ethernet's?**  
A: Wi-Fi (802.11) needs to handle wireless medium problems that don't exist on Ethernet — acknowledgment at the frame level (because wireless is less reliable), power management for battery devices, and beacon frames for network discovery. Different physical medium, different requirements.

---

*End of Lecture 2 Script*
