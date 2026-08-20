# Lecture 2: Network Packets and Layered Communication
## Assignment — SST Computer Networks (Term 5)

**Instructions:** Each question may have **one or more correct answers**. Select all that apply.  
Mark the question type for your reference: **(T)** = Theoretical, **(C)** = Conceptual, **(S)** = Scenario-based.

---

### Q1 (T) — What a Protocol Defines

Which of the following are part of what a network protocol defines?

A. The physical material used to manufacture network cables  
B. The format of messages — what fields appear and in what order  
C. The sequence of operations — who communicates first and how responses are structured  
D. The actions to take when a message is lost or corrupted  
E. The brand of hardware that must be used at each end  

**Correct Answers: B, C, D**

*Explanation: A protocol defines format, order, and actions on events. It is hardware and vendor agnostic — this is a core benefit of layered, standardized protocols.*

---

### Q2 (T) — OSI Layer Responsibilities

Match each responsibility to its correct OSI layer. Which of the following statements are true?

A. Layer 1 (Physical) is responsible for routing packets between different networks using IP addresses  
B. Layer 2 (Data Link) handles delivery within a single network segment using MAC addresses  
C. Layer 3 (Network) is responsible for routing packets across multiple networks using IP addresses  
D. Layer 4 (Transport) adds port numbers to identify which application should receive the data  
E. Layer 7 (Application) includes protocols like HTTP, DNS, and SMTP  

**Correct Answers: B, C, D, E**

*Explanation: A is wrong — inter-network routing via IP is Layer 3. All other statements correctly describe OSI layer responsibilities as defined in the lecture.*

---

### Q3 (T) — OSI to TCP/IP Mapping

In the TCP/IP model, which OSI layers are collapsed into the single "Application" layer?

A. Layer 4 — Transport  
B. Layer 5 — Session  
C. Layer 6 — Presentation  
D. Layer 7 — Application  
E. Layer 3 — Network  

**Correct Answers: B, C, D**

*Explanation: The TCP/IP Application layer combines OSI Layers 5 (Session), 6 (Presentation), and 7 (Application). Layer 4 (Transport) and Layer 3 (Network/Internet) are kept as separate layers in TCP/IP.*

---

### Q4 (C) — Benefits of Layered Architecture

Which of the following are genuine benefits of using a layered network architecture?

A. Each layer can be changed independently without affecting other layers  
B. It guarantees that all network connections will be encrypted  
C. TCP works identically whether the underlying network is Ethernet, Wi-Fi, or 4G  
D. Problems can be isolated to a specific layer, making debugging faster  
E. A layered stack requires fewer total protocols than a single monolithic protocol  

**Correct Answers: A, C, D**

*Explanation: Modularity (A) and reusability (C) are core benefits described in the lecture. Isolation for debugging (D) is also explicitly listed. Layered architecture does not guarantee encryption (B) — that's optional at Layer 6. It typically requires MORE distinct protocols (E), not fewer — each layer has its own.*

---

### Q5 (C) — Encapsulation Process

Which of the following correctly describe what happens during encapsulation as data travels down the network stack?

A. Each layer adds its own header to the data passed down from the layer above  
B. The Application layer header contains MAC addresses  
C. The TCP segment's payload is the HTTP message from the Application layer  
D. The IP packet's payload is the complete TCP segment  
E. Each layer reads and modifies the headers added by all layers above it  

**Correct Answers: A, C, D**

*Explanation: A correctly describes encapsulation. C and D correctly describe the nesting — each layer treats the layer above's entire output as its payload. B is wrong — MAC addresses are at Layer 2, not the Application layer. E is wrong — each layer treats the content from layers above it as opaque data; it does not read or modify those inner headers.*

---

### Q6 (C) — Frames, Packets, and Segments

Which of the following correctly distinguish frames, packets, and segments?

A. A frame uses MAC addresses and is valid only within a single network segment (one hop)  
B. A packet uses IP addresses and can travel across multiple networks  
C. A segment uses port numbers to identify which application should receive the data  
D. A packet is re-created at every router hop, just like a frame  
E. A frame is the data unit at Layer 2; a packet at Layer 3; a segment at Layer 4  

**Correct Answers: A, B, C, E**

*Explanation: A, B, C, and E are all correct definitions from the lecture. D is wrong — IP packets are NOT re-created at every hop; they remain unchanged (absent NAT). Frames ARE re-created at every hop.*

---

### Q7 (S) — Following an HTTP Request

Your laptop (192.168.1.5) sends an HTTP GET request to google.com (142.250.195.46). Which of the following are true at the moment the packet leaves your laptop?

A. The Ethernet destination MAC is your home router's MAC, not Google's MAC  
B. The IP destination address is 142.250.195.46 (Google's server)  
C. The TCP destination port is 80 (HTTP)  
D. The Ethernet destination MAC is Google's server MAC  
E. The IP source address is 192.168.1.5 (your laptop)  

**Correct Answers: A, B, C, E**

*Explanation: The Ethernet frame is addressed to the router (A) because Ethernet only works for local delivery — your laptop can't put Google's MAC in the frame because Google is not on your local network. The IP addresses (B, E) and TCP port (C) are set for the end-to-end path. D is wrong.*

---

### Q8 (C) — What Changes at Each Router Hop

A packet travels from your laptop through three routers to reach a server. Which of the following change at EVERY router hop along the path?

A. The IP source address  
B. The Ethernet source MAC address  
C. The Ethernet destination MAC address  
D. The IP destination address  
E. The TCP sequence number  

**Correct Answers: B, C**

*Explanation: Ethernet frames (and thus both Ethernet MAC addresses) are destroyed and re-created at every router hop. IP addresses (A, D) stay the same end-to-end (absent NAT). TCP sequence numbers (E) are not changed by routers — they are end-to-end.*

---

### Q9 (T) — Ethernet Frame Structure

Which of the following correctly describe the Ethernet frame structure?

A. The Preamble and SFD together occupy 8 bytes and are used to synchronize the receiver  
B. The EtherType field identifies the protocol carried in the payload  
C. The payload can be anywhere from 46 to 1500 bytes in size  
D. The FCS field retransmits the frame if a CRC error is detected  
E. The Destination MAC field can be FF:FF:FF:FF:FF:FF to address all devices on the LAN  

**Correct Answers: A, B, C, E**

*Explanation: A (Preamble 7B + SFD 1B = 8B total), B (EtherType identifies payload protocol), C (46–1500 byte payload), and E (broadcast MAC = all Fs) are all correct. D is wrong — if the FCS check fails, the frame is silently discarded. Retransmission is the responsibility of TCP at Layer 4, not Ethernet.*

---

### Q10 (T) — EtherType Field Values

An Ethernet frame has EtherType value `0x0800`. Which of the following are true about this frame?

A. The payload contains an IPv4 packet  
B. The payload contains an ARP message  
C. The frame is a broadcast frame  
D. A device receiving this frame should pass the payload to its IP protocol handler  
E. The EtherType value `0x0806` would indicate an ARP payload instead  

**Correct Answers: A, D, E**

*Explanation: 0x0800 = IPv4 (A). The receiver's IP handler processes the payload (D). 0x0806 = ARP (E). B is wrong — ARP is 0x0806, not 0x0800. C is wrong — broadcast is determined by the Destination MAC (FF:FF:FF:FF:FF:FF), not the EtherType.*

---

### Q11 (S) — MTU and Fragmentation

An application sends 4,000 bytes of data over an Ethernet network with an MTU of 1500 bytes. Which of the following are true about how this data is transmitted?

A. The IP layer fragments the data into multiple smaller packets  
B. The data is transmitted as a single 4,000-byte Ethernet frame  
C. All fragments of the same original packet share the same IP Identification field value  
D. Fragments are reassembled at every intermediate router along the path  
E. If one fragment is lost, the destination discards all received fragments and TCP must retransmit  

**Correct Answers: A, C, E**

*Explanation: Data exceeding MTU is fragmented by IP (A). All fragments of the same packet share the same Identification field (C) so the destination can reassemble them. If any fragment is lost, the whole original segment must be retransmitted (E). B is wrong — a 4,000-byte payload cannot fit in a single 1500-byte MTU frame. D is wrong — reassembly happens ONLY at the final destination, not at intermediate routers.*

---

### Q12 (T) — IP Fragmentation Header Fields

Which IP header fields are directly involved in the fragmentation and reassembly process?

A. Identification field — all fragments of the same original packet share the same value  
B. Fragment Offset — indicates where in the original data this fragment belongs, in 8-byte units  
C. More Fragments (MF) bit — set to 1 on all fragments except the last  
D. Don't Fragment (DF) bit — when set, instructs routers to drop rather than fragment the packet  
E. TTL field — ensures fragments don't loop forever  

**Correct Answers: A, B, C, D**

*Explanation: Identification (A), Fragment Offset (B), MF bit (C), and DF bit (D) are all directly used in IP fragmentation. The TTL field (E) is not specifically related to fragmentation — it decrements at every hop regardless and prevents routing loops in general, not specifically fragmentation loops.*

---

### Q13 (T) — MAC Address Properties

Which of the following correctly describe MAC addresses?

A. A MAC address is 48 bits (6 bytes) long  
B. The first 3 bytes of a MAC address identify the manufacturer (OUI)  
C. MAC addresses are globally routable — a router uses them to forward packets across the internet  
D. A MAC address is replaced with a new one at every router hop  
E. Modern smartphones randomize their MAC address when connecting to new networks for privacy  

**Correct Answers: A, B, D, E**

*Explanation: A (48-bit), B (OUI = first 3 bytes = manufacturer), D (frames re-created at each hop = new MAC addresses), and E (MAC randomization for privacy) are all correct. C is wrong — MAC addresses are local scope only, not globally routable. IP addresses handle inter-network routing.*

---

### Q14 (S) — Diagnosing by OSI Layer

A user reports: "My laptop shows 'connected' on Wi-Fi. I can reach my home router. But I cannot load any website and I cannot ping 8.8.8.8 (Google's public IP)." Which OSI layers are most likely to have a problem?

A. Layer 1 (Physical) — because the Wi-Fi connection shows as active  
B. Layer 2 (Data Link) — because the user can reach the home router  
C. Layer 3 (Network) — because the user cannot ping an external IP address  
D. Layer 4 (Transport) — because TCP connections are failing  
E. Layer 7 (Application) — because websites won't load  

**Correct Answers: C**

*Explanation: Layer 1 is working (Wi-Fi is connected, A is confirmed working). Layer 2 is working (can reach the router, B is confirmed working). The inability to ping 8.8.8.8 — an IP-level test — points to Layer 3 as the problem (routing, gateway config, or ISP issue). The failure at Layer 7 (websites) is a consequence of the Layer 3 failure, not an independent problem. TCP (D) cannot even attempt to connect without IP routing working.*

---

### Q15 (S) — Path MTU Discovery

A network engineer configures a server to send packets with the Don't Fragment (DF) bit set in the IP header. Along the path, one router has a smaller MTU than the packet size. Which of the following correctly describe what happens?

A. The router fragments the packet into smaller pieces and forwards them  
B. The router drops the packet and sends an ICMP "Fragmentation Needed" message back to the source  
C. The source receives the ICMP message and reduces its packet size for future sends  
D. This process — DF bit + ICMP feedback + size adjustment — is called Path MTU Discovery (PMTUD)  
E. Blocking ICMP entirely on a network would cause PMTUD to work faster  

**Correct Answers: B, C, D**

*Explanation: With DF=1, the router cannot fragment — it must drop and return an ICMP error (B). The source uses this feedback to reduce packet size (C). This three-step process is PMTUD (D). A is wrong — the DF bit explicitly forbids fragmentation. E is wrong — blocking ICMP breaks PMTUD entirely, causing large transfers to silently fail.*

---

*Lecture 2 Assignment — 15 Questions | Computer Networks, Term 5, SST*
