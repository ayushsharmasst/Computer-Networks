# Lecture 2: Network Packets and Layered Communication
## Instructor Script — SST Computer Networks (Term 5)

---

### 🎯 Learning Objectives
By the end of this session, students will be able to:
- Explain what a protocol is and why layering exists
- Trace how data is encapsulated as it travels down the network stack
- Distinguish between frames, packets, and segments
- Describe the structure of an Ethernet frame
- Explain MTU and what happens when data exceeds it

**Duration:** ~90 minutes  
**Teaching Style:** Practical-first, analogy-driven  
**Demo Materials:** Wireshark (or screenshots), a simple Python socket script (optional)

---

## 📦 OPENING HOOK (5 minutes)

**[INSTRUCTOR: Start with this scenario. Write on the board: "You send a WhatsApp message. What actually happens?"**]

Say to students:

> "Yesterday I sent a file to a friend on WhatsApp. 47 megabytes. He's in Bangalore, I'm here. How did those 47 megabytes physically travel? Was it one giant chunk? Was it broken up? Who handled what?"

Take 2-3 hands. Let students speculate. Then say:

> "The answer is — it was broken into **thousands of small pieces**, each piece was wrapped in multiple envelopes, each envelope handled by a different postal department, and then reassembled on the other side. That's exactly what we're going to learn today — the anatomy of those envelopes and how they're layered."

---

## SECTION 1: What is a Protocol? (10 minutes)

**[INSTRUCTOR: Draw two stick figures on the board — one labelled "You" and one labelled "Friend".]**

### Analogy — Language as a Protocol

> "If you call someone and speak in Hindi but they only understand Tamil, what happens?"

**Answer:** Communication fails.

> "A **protocol** is simply an agreed-upon set of rules for communication. Both sides must speak the same language in the same format, in the same order."

### In Networking:

Write on board:
```
Protocol = Rules that define:
  - FORMAT of messages
  - ORDER of messages
  - Actions to take on sending/receiving
```

**Examples:**
- HTTP: Rules for web requests/responses
- SMTP: Rules for sending email
- TCP: Rules for reliable delivery
- IP: Rules for addressing and routing

**Key point to emphasize:**
> "Protocols exist at EVERY layer. The HTTP layer doesn't care how TCP works. TCP doesn't care how IP works. Each layer has its own job, its own protocol, its own rules. This is called **layered architecture**."

**[PAUSE: Ask students — "Why do you think we break it into layers instead of having one giant protocol?"  
Take 2-3 answers. Expected: easier to change, easier to debug, modularity.]**

**Correct answer to formalize:**
- Modularity: Change one layer without touching others
- Reuse: HTTP can run over TCP, which can run over Wi-Fi OR Ethernet — same TCP either way
- Debugging: If a web page doesn't load, you can isolate — is it DNS? TCP? HTTP?

---

## SECTION 2: Encapsulation and Decapsulation (15 minutes)

**[INSTRUCTOR: This is the core concept of the lecture. Draw the layered stack on the board and build it live.]**

### The Postal Analogy

> "Imagine you're sending a secret letter. You write the letter, put it in an envelope (with your address and their address), put THAT in a shipping box, label the box for the postal service, which then loads it on a truck. Each wrapper adds addressing info for THAT level of delivery."

Draw on board:
```
APPLICATION DATA:    "Hello, Ayush!"
         ↓ [HTTP wraps it]
HTTP MESSAGE:        [HTTP Header] + "Hello, Ayush!"
         ↓ [TCP wraps it]
TCP SEGMENT:         [TCP Header] + [HTTP Header] + "Hello, Ayush!"
         ↓ [IP wraps it]
IP PACKET:           [IP Header] + [TCP Header] + [HTTP Header] + "Hello, Ayush!"
         ↓ [Ethernet wraps it]
ETHERNET FRAME:      [ETH Header] + [IP Packet] + [ETH Trailer]
         ↓
Bits on the wire ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

**Say:**
> "This process of wrapping data with headers at each layer as it goes DOWN the stack is called **encapsulation**."

> "On the receiving side, each layer UNWRAPS its own header and passes the rest up — this is called **decapsulation**."

### Live Walkthrough — Sending an HTTP Request

Walk through step by step:

**Step 1 — Application Layer:**
```
You type google.com → Browser creates:
GET / HTTP/1.1
Host: google.com
```

**Step 2 — Transport Layer (TCP):**
```
TCP adds:
- Source Port: 52341 (random)
- Destination Port: 80
- Sequence Number: 1001
- Checksum
```

**Step 3 — Network Layer (IP):**
```
IP adds:
- Source IP: 192.168.1.5 (your laptop)
- Destination IP: 142.250.195.46 (Google's server)
- TTL: 64
```

**Step 4 — Data Link Layer (Ethernet):**
```
Ethernet adds:
- Source MAC: AA:BB:CC:DD:EE:FF
- Destination MAC: 11:22:33:44:55:66 (your router's MAC)
```

**[INSTRUCTOR NOTE: Point out that the MAC destination is the ROUTER, not Google. Why? Because MAC is only for local delivery. Once it crosses the router, Ethernet frame changes. IP packet stays the same all the way.]**

---

## SECTION 3: Headers and Payload (8 minutes)

Draw a simple box:

```
┌──────────────────────────────────────────────────┐
│  HEADER          │  PAYLOAD (DATA)                │
│ (metadata/control│  (actual content being sent)   │
│  information)    │                                │
└──────────────────────────────────────────────────┘
```

**Ask students:** "What kind of information would you expect in a header?"

**Build the answer:**
- Source and destination addresses (who sent it, who should receive it)
- Length of the payload
- Type of data inside
- Error-checking information
- Sequence numbers (for ordering)
- Flags (SYN, ACK, FIN in TCP)

**Key insight:**
> "The payload of one layer is the ENTIRE message of the layer above it. So the IP packet's payload is the TCP segment. The TCP segment's payload is the HTTP message. This is called **data encapsulation** — each layer treats everything above it as just 'data'."

---

## SECTION 4: Frames vs Packets vs Segments (10 minutes)

**[INSTRUCTOR: Write these three terms on the board with their layer.]**

| Term | Layer | Protocol | Addresses Used |
|------|-------|----------|---------------|
| Frame | Data Link (L2) | Ethernet, Wi-Fi | MAC Address |
| Packet | Network (L3) | IP | IP Address |
| Segment | Transport (L4) | TCP | Port Numbers |

**Explain each:**

### Segment (TCP/UDP)
> "A TCP segment is the transport layer's unit. It carries your application's data with port numbers attached. Think of this as — 'For which door of the building is this data meant? Door 80 = HTTP, Door 443 = HTTPS, Door 22 = SSH.'"

### Packet (IP)
> "An IP packet adds addressing at the city level — IP addresses tell us which building in which city this data is going to. A packet can travel across multiple networks — from your home to Jio's network to Google's network."

### Frame (Ethernet)
> "A frame is local — it only makes sense on YOUR network segment. Like a name on a box being delivered within a single building floor. Once it crosses a router, the old frame is discarded and a new frame is created for the next hop."

**Demonstration — Frame changes at each hop:**

```
Your Laptop → Router (Frame 1: Your MAC → Router's LAN-side MAC)
Router → ISP Router (Frame 2: Router's WAN-side MAC → ISP Router's MAC)
ISP Router → Google (Frame 3: Different MACs again)

BUT: The IP packet and TCP segment inside stay the SAME all the way!
```

**[PAUSE — Ask: "So if the frame changes at every hop, but IP stays the same — what does that tell us about the scope of each layer?"**]

---

## SECTION 5: Ethernet Frame Structure (10 minutes)

**[INSTRUCTOR: Draw the Ethernet frame structure on the board or project it.]**

```
┌──────────┬──────────┬────────┬──────────────────────┬─────────┐
│ Preamble │  Dest    │  Src   │  EtherType / Length  │ Payload │  FCS   │
│ 7 bytes  │  MAC     │  MAC   │  2 bytes             │ 46-1500 │ 4 bytes│
│ + 1 SFD  │  6 bytes │ 6 bytes│                      │  bytes  │        │
└──────────┴──────────┴────────┴──────────────────────┴─────────┴────────┘
```

**Walk through each field:**

**Preamble (7 bytes) + SFD (1 byte):**
> "The preamble is like a car horn before you speak — it wakes up the receiver and tells it 'data is coming.' The SFD (Start Frame Delimiter) says 'NOW it starts.'"

**Destination MAC (6 bytes):**
> "48-bit MAC address of who should receive this frame on the LOCAL network. Could be a unicast address (specific device), broadcast (FF:FF:FF:FF:FF:FF — everyone), or multicast."

**Source MAC (6 bytes):**
> "48-bit MAC of the sender."

**EtherType (2 bytes):**
> "This tells us what's inside the payload. 0x0800 = IPv4. 0x0806 = ARP. 0x86DD = IPv6. It's like writing 'FRAGILE' or 'ELECTRONICS' on a shipping box."

**Payload (46–1500 bytes):**
> "The actual IP packet. Notice the minimum is 46 bytes — frames must be at least 64 bytes total for collision detection to work on older networks. If your payload is small, it gets padded."

**FCS — Frame Check Sequence (4 bytes):**
> "A CRC checksum. The receiver recomputes this from the data. If it doesn't match, the frame is silently DISCARDED. No retransmission at this level — that's TCP's job."

**Quick Question:** "If the FCS check fails, who is responsible for asking for a re-send?"
**Answer:** TCP (at the transport layer) — because Ethernet just drops the frame silently.

---

## SECTION 6: MTU and Fragmentation (12 minutes)

### What is MTU?

**Say:**
> "We just saw that the Ethernet payload can be at most **1500 bytes**. This limit is called the **MTU — Maximum Transmission Unit**."

**Ask:** "What happens if you try to send more than 1500 bytes in one go?"

**Answer:** It gets broken up — **fragmented**.

### Why 1500 bytes?
> "This is a historical limit from the original Ethernet spec. Different link types have different MTUs — Wi-Fi can be up to 2304 bytes. But Ethernet (which is everywhere) caps at 1500, so the entire internet effectively uses 1500 as the standard."

### Fragmentation — Practical Example

> "You download a 100KB image. That's 102,400 bytes. With an MTU of 1500 bytes, that image gets broken into approximately **68 IP fragments**. Each fragment is an independent IP packet."

Draw on board:
```
Original IP Packet: 5000 bytes payload

→ Fragment 1: bytes 0-1479    (offset=0,   MF=1)
→ Fragment 2: bytes 1480-2959 (offset=185, MF=1)
→ Fragment 3: bytes 2960-4439 (offset=370, MF=1)
→ Fragment 4: bytes 4440-4999 (offset=555, MF=0) ← last fragment
```

**IP Header fields involved in fragmentation:**
- **Identification**: All fragments of the same original packet share the same ID
- **Fragment Offset**: Where in the original packet does this fragment start (in 8-byte units)
- **More Fragments (MF) bit**: 1 = more fragments coming, 0 = this is the last one

**Important note:**
> "Reassembly happens at the DESTINATION only — not at intermediate routers. If even one fragment is lost, the ENTIRE original packet must be retransmitted by TCP."

**Modern practice — Path MTU Discovery:**
> "Modern systems try to avoid fragmentation using PMTUD — they probe the path to find the smallest MTU along the route, then set their packet size accordingly. You'll see this in action with the 'Don't Fragment' bit in IP headers."

---

## SECTION 7: MAC Addresses (8 minutes)

### What is a MAC Address?

> "MAC stands for **Media Access Control**. It's a **physical** address burned into every network interface card (NIC) at the factory."

**Format:**
```
AA:BB:CC:DD:EE:FF
│  OUI (first 24 bits)  │  Device-specific (last 24 bits) │
```

**OUI = Organizationally Unique Identifier** — assigned by IEEE to manufacturers.
- First 3 bytes identify the manufacturer (e.g., Apple, Intel, Qualcomm)
- Last 3 bytes are unique to the device

**Run this command (instructor demo):**
```bash
ip link show        # Linux/Mac
ipconfig /all       # Windows
# Look for "ether" or "Physical Address"
```

**Key Properties:**
1. **Global uniqueness** (in theory): No two devices should have the same MAC
2. **Layer 2 scope**: Only meaningful within a single network segment
3. **Changes at each router hop** (the frame is re-created)
4. **Can be spoofed** (software-level MAC spoofing is common in privacy tools)

**Fun fact:**
> "When you connect to a café Wi-Fi, your phone's MAC address is visible to the router. This is why modern phones use 'randomized MAC addresses' — so the café can't track your device across visits."

### ARP — Connecting IP to MAC

> "We know IP addresses, but Ethernet needs MAC addresses. How do we convert an IP to a MAC on a local network? We use **ARP** — Address Resolution Protocol. We'll cover it in detail in Lecture 11, but here's the preview:"

```
Your PC (192.168.1.5) wants to reach 192.168.1.1 (router)
→ Broadcasts on LAN: "WHO HAS 192.168.1.1? Tell 192.168.1.5"
→ Router replies: "192.168.1.1 is at 11:22:33:44:55:66"
→ Your PC caches this and uses that MAC in its Ethernet frames
```

---

## SECTION 8: Quick Review and Real-World Context (5 minutes)

**[INSTRUCTOR: Do a quick Wireshark demo if available, OR show a screenshot of a captured frame.]**

Show or describe a Wireshark capture of a simple HTTP request. Point out:
- Ethernet frame at the bottom
- IP packet inside it
- TCP segment inside that
- HTTP data at the top

**Say:**
> "Every time you open a web page, this exact nesting happens hundreds of times per second. Your browser creates the HTTP message, your OS adds TCP + IP, your network driver adds the Ethernet frame, and your NIC shoots it out as electrical signals or Wi-Fi radio waves."

---

## 📝 CLASS DISCUSSION QUESTION (5 minutes)

> "If you're on 4G mobile internet instead of Wi-Fi, will you still see Ethernet frames? Why or why not?"

**Expected answer:** No — mobile uses different L2 protocols (like LTE's radio frames). But the IP packet inside is the same. This is the power of layering — the same IP packet can travel over Ethernet, Wi-Fi, or 4G, and TCP and HTTP above it don't need to change at all.

---

## SUMMARY (3 minutes)

Write on board / summarize verbally:

```
✅ Protocol = agreed-upon rules for communication
✅ Encapsulation = wrapping data with headers as it goes down the stack
✅ Decapsulation = unwrapping headers as it goes up the stack
✅ Segment (L4) → Packet (L3) → Frame (L2)
✅ Ethernet Frame: Preamble | Dest MAC | Src MAC | EtherType | Payload | FCS
✅ MTU = 1500 bytes max for Ethernet; larger data gets fragmented
✅ MAC = physical address, unique per NIC, local scope only
```

---

## 🔗 Preview of Next Lecture

> "Now that we know HOW data travels in packets, we need to know WHERE to send them. That's the job of IP addresses — the postal codes of the internet. Next lecture, we'll learn how IP addressing works, how to read a subnet, and how to split a network into smaller pieces."

---

## ❓ Potential Student Questions

**Q: Does the router read the HTTP headers?**  
A: A basic router at Layer 3 only reads IP headers. It doesn't look inside. But some modern routers/firewalls do "deep packet inspection" and can read all the way up to application data.

**Q: What if a fragment arrives out of order?**  
A: The destination collects all fragments (identified by the same IP ID field) and reassembles them using the offset values — order doesn't matter.

**Q: Is 1500 bytes the MTU for Wi-Fi too?**  
A: Wi-Fi's 802.11 standard allows larger frames (up to 2304 bytes payload), but in practice Wi-Fi networks that connect to Ethernet respect the 1500 byte Ethernet MTU to avoid fragmentation.

**Q: Can we change the MTU?**  
A: Yes! You can set the MTU on a network interface. `ip link set eth0 mtu 9000` enables "jumbo frames" on Ethernet (up to 9000 bytes) — common in data centers for performance.

---

*End of Lecture 2 Script*
