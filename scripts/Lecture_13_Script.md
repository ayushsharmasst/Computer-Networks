# Lecture 13: The Complete Internet Journey — From URL to Response
## Instructor Script — SST Computer Networks (Term 5)

---

### 🎯 Learning Objectives
By the end of this session, students will:
- Trace the complete journey of a web request end-to-end at every protocol layer
- Connect all 12 previous lectures into one unified mental model
- Identify which layer/protocol handles each phase of a request
- Understand where performance problems originate

**Duration:** ~90 minutes

---

## 📦 OPENING HOOK (5 minutes)

**[INSTRUCTOR: Open a browser. Type a URL. Press Enter. Ask:]**

> "Between pressing Enter and seeing this page load — what just happened? How many protocols ran? How many machines did your request touch? How many layers did it traverse? In this final lecture, we're going to answer that completely. Every step. Every header. Every packet."

> "Think of this lecture as a director's cut — you've seen every scene individually over the last 12 lectures. Today we're watching the whole film."

**[INSTRUCTOR: Draw a blank horizontal timeline on the board. You'll fill it in over the course of the lecture.]**

```
[Browser] ──?──?──?──?──?──?──?──?──?──[Web Server]
                            TIME →
```

> "By the end of this lecture, you will be able to fill every gap in this timeline."

---

## SETUP: THE SCENARIO (3 minutes)

**[INSTRUCTOR: Write this on the board at the start and keep it visible throughout]**

**Scenario:**
```
User opens laptop on home Wi-Fi. Types in browser:
  https://www.example.com/products?id=42

Laptop IP:     192.168.1.5 (from DHCP)
Gateway IP:    192.168.1.1 (home router)
Router WAN IP: 203.0.113.5 (from ISP DHCP)

www.example.com is hosted at 93.184.216.34
  - Behind a load balancer
  - Web server at 10.10.0.5 (internal)
  - TLS-enabled (HTTPS)
```

**What happens between pressing Enter and the page appearing?**

---

## PHASE 1: Browser Cache Check (2 minutes)

> "Before any network communication happens at all, the browser checks if it already has what it needs."

### What the Browser Checks

```
1. Browser Memory Cache:
   "Have I loaded example.com/products in this session?
    Is the cached response still within its max-age?"
   → Cache hit → return immediately (0ms, no network)
   → Cache miss → continue

2. Browser Disk Cache (HTTP cache):
   "Do I have a cached response from a previous session?"
   → If fresh (within Cache-Control: max-age) → serve from disk
   → If stale → issue conditional GET (If-None-Match: "etag123")
     → Server replies 304 Not Modified → use cached body

3. First visit → no cache → must make full request
```

> "Cache hits are invisible to users and free in terms of network cost. A well-configured CDN and cache policy means most returning users never hit your server. For our scenario: first visit, no cache. Onward."

**[INSTRUCTOR: Mark "Browser Cache Check" on the timeline]**

---

## PHASE 2: DNS Resolution (10 minutes)

> "The user typed www.example.com. The network layer needs an IP address. Time for DNS."

### Step 1: OS Checks Local Caches

```
Browser DNS Cache:
  "Have I resolved www.example.com recently?"
  → Not cached → ask OS

OS Resolver Cache (systemd-resolved / dnsmasq):
  "Is it in /etc/hosts?"
  → No
  "Is it cached from a previous query?"
  → No → proceed to recursive resolution
```

### Step 2: Query to DNS Resolver

```
OS sends DNS query:
  Source:      192.168.1.5:54321  (ephemeral port)
  Destination: 192.168.1.1:53    (router = DNS forwarder)
  Protocol:    UDP
  Question:    www.example.com IN A?
```

> "Your router's DNS is typically a forwarder — it forwards your query to a real recursive resolver (like 8.8.8.8 or your ISP's DNS)."

### Step 3: Recursive Resolution

**[INSTRUCTOR: Draw the DNS hierarchy as you narrate each step]**

```
Recursive Resolver (8.8.8.8):

  Step 1 — Query Root Server:
    "Who handles .com?"
    Root server replies: "Ask a .com TLD nameserver: 192.5.6.30"

  Step 2 — Query TLD Server (.com):
    "Who handles example.com?"
    TLD replies: "Ask authoritative NS: ns1.example.com at 205.251.196.1"

  Step 3 — Query Authoritative Nameserver:
    "What is www.example.com?"
    Auth NS replies:
      www.example.com.  300  IN  A  93.184.216.34
                        ↑TTL       ↑IP address

  Resolver caches this for 300 seconds.
  Returns 93.184.216.34 to your OS.
```

### Step 4: OS Has the IP

```
www.example.com → 93.184.216.34
Cached in OS resolver for 300 seconds.
```

**[INSTRUCTOR: Add "DNS Resolution" to the timeline. Ask: "What protocol carries DNS? How many round trips? Which machines were involved?"]**

> "That entire DNS exchange was 3 queries, each ~30ms. So ~90ms before a single byte of HTTP has been sent."

---

## PHASE 3: ARP — Finding the Gateway's MAC (5 minutes)

> "Now we have the destination IP: 93.184.216.34. Is it on our local subnet (192.168.1.0/24)? No — so we send to the default gateway. But Ethernet needs a MAC address."

### ARP for Gateway

```
OS routing decision:
  "Is 93.184.216.34 in 192.168.1.0/24?"
  → No → send to gateway 192.168.1.1

ARP cache check:
  "Do I have a MAC for 192.168.1.1?"
  → Cache hit (likely cached from DHCP) → router MAC = 11:22:33:44:55:66
  → Cache miss → ARP broadcast:
      "WHO HAS 192.168.1.1? TELL 192.168.1.5"
      Router replies: "192.168.1.1 IS AT 11:22:33:44:55:66"
      Cache updated.
```

> "ARP operates at Layer 2 — it translates L3 IP to L2 MAC. Once we have the MAC, we can build the Ethernet frame."

**[INSTRUCTOR: Add "ARP" to timeline. Point out: "Layer 2 problem solved by Layer 2 protocol. The IP header doesn't change here — only the Ethernet header."]**

---

## PHASE 4: TCP Three-Way Handshake (10 minutes)

> "Before a single byte of application data moves, TCP must establish a connection. This is the cost of reliability."

### TCP Connection to 93.184.216.34:443

```
Laptop                              93.184.216.34 (Load Balancer)

  │── SYN (seq=1000) ─────────────▶│
  │   Src: 192.168.1.5:52341        │  (ephemeral port)
  │   Dst: 93.184.216.34:443        │
  │   Flags: [S], seq=1000          │
  │                                  │
  │◀─ SYN-ACK (seq=5000, ack=1001) ─│
  │   Flags: [S.], seq=5000         │
  │          ack=1001               │
  │                                  │
  │── ACK (ack=5001) ─────────────▶│
  │   Flags: [.]                    │
  │                                  │
  ╔════════ ESTABLISHED ════════════╗
```

**[INSTRUCTOR: Mark the RTT on the diagram]**

> "Each arrow is one RTT. If the server is 100ms away, this handshake alone costs 200ms — just to say hello. This is why HTTP/2 and QUIC try to reduce or eliminate this overhead. QUIC (HTTP/3) does 0-RTT on repeat connections."

### What Happens in the Network During SYN

```
Laptop builds packet:
  Ethernet: Dst=11:22:33:44:55:66, Src=AA:BB:CC:DD:EE:FF
  IP:       Src=192.168.1.5, Dst=93.184.216.34
  TCP:      Src=52341, Dst=443, Flags=SYN

Router receives on LAN:
  → Strips Ethernet frame
  → NAT: Src 192.168.1.5:52341 → 203.0.113.5:10001
  → Creates NAT table entry
  → New Ethernet frame for WAN link
  → Forwards to ISP

Packet traverses internet (~10-12 hops) to reach 93.184.216.34
```

**[INSTRUCTOR: Draw packet layers on board — Ethernet | IP | TCP | empty payload. Show how NAT changes only the IP/port, not TCP payload.]**

---

## PHASE 5: TLS Handshake (10 minutes)

> "The URL starts with https — so we need TLS encryption BEFORE sending the HTTP request. The TLS handshake happens INSIDE the TCP connection."

### TLS 1.3 Handshake (Simplified)

```
Laptop                              Server (Load Balancer)

  │── ClientHello ─────────────────▶│
  │  TLS version: 1.3               │
  │  Cipher suites: [AES-GCM, ...]  │
  │  Random bytes (client nonce)    │
  │  Server Name Indication (SNI):  │
  │    "www.example.com"            │
  │                                  │
  │◀─ ServerHello ──────────────────│
  │  Chosen cipher: AES-256-GCM     │
  │  Random bytes (server nonce)    │
  │  Server Certificate             │
  │    (contains public key)        │
  │  Certificate Verified           │
  │  Server Finished                │
  │                                  │
  │── Client Finished ─────────────▶│
  │  Key exchange complete           │
  │  (Session keys derived)         │
  │                                  │
  ╔══════ ENCRYPTED CHANNEL ════════╗
```

**[INSTRUCTOR: Explain each step:]**

> "**SNI (Server Name Indication):** The server's IP might host multiple domains. SNI tells the server which certificate to present — like a virtual host name at the TLS layer."

> "**Certificate Verification:** Browser checks the certificate chain up to a trusted root CA. If valid, it proceeds. If expired or self-signed, you see the 'Your connection is not private' warning."

> "**Session Keys:** Both sides derive the same symmetric encryption key (AES-256) using the Diffie-Hellman key exchange embedded in the hellos. No key was ever transmitted — both sides compute it independently from the exchanged values."

> "**TLS 1.3 advantage:** The handshake is 1-RTT (vs 2-RTT in TLS 1.2). On repeat connections, TLS 1.3 supports 0-RTT (session resumption)."

**[INSTRUCTOR: Add "TLS Handshake" to timeline. Ask: "How many total round trips have we spent before sending our actual HTTP request?" Answer: 1 DNS + 1 TCP + 1 TLS = ~3 RTTs minimum.]**

---

## PHASE 6: HTTP Request (5 minutes)

> "Finally. After DNS, ARP, TCP, and TLS — we send the actual HTTP request. It's one packet."

### The HTTP GET Request

```
GET /products?id=42 HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0 (...)
Accept: text/html,application/xhtml+xml
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Cookie: session_id=abc123; theme=dark
Connection: keep-alive
```

**[INSTRUCTOR: Annotate each header:]**

> "**Host**: required in HTTP/1.1 — tells the server which virtual host to serve (the IP might host many domains)."

> "**Accept-Encoding: gzip**: browser tells server it can decompress compressed responses — server will gzip the response body, often reducing size by 70-80%."

> "**Cookie**: browser automatically attaches cookies for this domain. The server uses session_id to identify this user's session."

> "**Connection: keep-alive**: 'Don't close the TCP connection after this request — I'll send more.' This avoids repeated TCP + TLS handshakes for each resource."

### Packet Stack at This Moment

```
┌─────────────────────────────────┐
│  HTTP Request (plaintext)       │  ← Application Layer
├─────────────────────────────────┤
│  TLS Record (encrypted)         │  ← Presentation Layer
├─────────────────────────────────┤
│  TCP (Seq, Ack, Port 52341→443) │  ← Transport Layer
├─────────────────────────────────┤
│  IP (192.168.1.5 → 93.184.216.34│  ← Network Layer
├─────────────────────────────────┤
│  Ethernet Frame                 │  ← Data Link Layer
└─────────────────────────────────┘
```

**[INSTRUCTOR: This is the full encapsulation stack. Every layer is visible here. Point back to Lecture 2.]**

---

## PHASE 7: Server-Side Processing (8 minutes)

> "The packet reaches the load balancer at 93.184.216.34. Now the request enters the server infrastructure."

### Load Balancer

```
Load Balancer receives packet:
  Dst IP: 93.184.216.34:443 → this is me
  TLS decryption: LB terminates TLS (SSL offloading)
  
  HTTP routing decision:
    Request: GET /products?id=42
    Host: www.example.com
    Session/least-connections: route to Backend 10.10.0.5

  Forward to 10.10.0.5:8080
    (New TCP connection from LB to backend)
    (X-Forwarded-For: 203.0.113.5 header added — actual client IP)
```

> "**SSL Offloading:** The load balancer terminates TLS. This means the backend servers don't need SSL certificates or the CPU cost of encryption — they receive plain HTTP. The connection between LB and backend is on a trusted internal network."

> "**X-Forwarded-For:** Because the LB makes a new TCP connection to the backend, the backend sees the LB's IP as the source. X-Forwarded-For passes the real client IP for logging, rate limiting, and geolocation."

### Web Server (10.10.0.5)

```
Web Server receives HTTP request:
  1. Parse URL: GET /products?id=42
  2. Route to /products handler
  3. Query parameter: id=42
  4. Check session cookie: session_id=abc123 → authenticated user #789
  5. Database query: SELECT * FROM products WHERE id=42
  6. Template rendering: generate HTML response
  7. Build HTTP response
```

> "The web server is a standard HTTP application. At this point we're in pure application logic — but notice how every detail needed to get here was handled by the network protocols we studied."

### HTTP Response

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Content-Length: 4821
Cache-Control: max-age=300, public
ETag: "a3f8b2c1"
Set-Cookie: last_seen=product_42; Max-Age=86400; Secure; HttpOnly
X-Request-ID: req-abc-123

[gzip-compressed HTML body]
```

**[INSTRUCTOR: Annotate key response headers:]**

> "**Content-Encoding: gzip**: response body is compressed. Browser will decompress before rendering."

> "**Cache-Control: max-age=300**: browser can cache this response for 5 minutes — next visit within 5 min won't need a network request."

> "**ETag**: a fingerprint of the content. Next time browser can send If-None-Match to check if content changed — if not, server sends 304 instead of the full response."

> "**Set-Cookie: Secure; HttpOnly**: Secure means only send over HTTPS; HttpOnly means JavaScript cannot access this cookie (XSS protection)."

---

## PHASE 8: Response Journey Back (5 minutes)

> "Now the response travels back through the network, reversing all the layers."

### The Return Path

```
Web Server → Load Balancer:
  HTTP response (plaintext on internal network)

Load Balancer → Router:
  TLS encrypts response
  New TCP segment: Src=93.184.216.34:443, Dst=203.0.113.5:10001

Router (NAT translation):
  Dst lookup: port 10001 → 192.168.1.5:52341
  Rewrite: Dst=192.168.1.5:52341
  ARP for laptop's MAC
  Forward on LAN

Laptop receives TCP segment:
  TLS: decrypt
  HTTP: parse response headers + body (decompress gzip)
  Browser: render HTML
```

### Rendering and Subsequent Requests

> "The HTML contains references to other resources — CSS, JavaScript, images. The browser makes additional requests for each:"

```
GET /static/style.css     → another HTTP request (may hit cache)
GET /static/app.js        → another HTTP request
GET /images/product_42.jpg → another HTTP request (may hit CDN)
```

> "With HTTP/1.1 keep-alive, all these requests reuse the same TCP connection — no new handshakes. HTTP/2 goes further with multiplexing — all requests in parallel on one connection."

---

## PHASE 9: Connection Teardown (3 minutes)

> "Eventually, the browser decides it's done with this connection."

### TCP Four-Way Close

```
Laptop                              Server

  │── FIN ───────────────────────▶│
  │◀─ ACK ────────────────────────│
  │◀─ FIN ────────────────────────│
  │── ACK ───────────────────────▶│
  
  [Laptop enters TIME_WAIT: ~60 seconds]
  [NAT entry expires: cleaned up by router]
```

> "TIME_WAIT exists to absorb any delayed packets from this connection. The NAT entry in the router is eventually cleaned up after the TCP close is detected."

---

## COMPLETE TIMELINE SUMMARY (10 minutes)

**[INSTRUCTOR: Fill in the complete timeline now. This is the payoff moment.]**

```
Time 0ms:    User presses Enter
             Browser cache check → miss

~1ms:        OS checks /etc/hosts, local DNS cache → miss
~5ms:        DNS query to router (192.168.1.1:53) via UDP
~20ms:       Router forwards to 8.8.8.8
~30ms:       Root server query → TLD referral
~50ms:       TLD query → authoritative nameserver
~70ms:       Authoritative reply: www.example.com = 93.184.216.34
             DNS cached (TTL 300s)

~72ms:       ARP for gateway MAC (likely cached → ~0ms)
             Ethernet frame built

~75ms:       TCP SYN sent (Src: 192.168.1.5:52341, Dst: 93.184.216.34:443)
             NAT: 203.0.113.5:10001 → 93.184.216.34:443

~175ms:      SYN-ACK received
~177ms:      ACK sent — TCP ESTABLISHED

~180ms:      TLS ClientHello sent
~280ms:      TLS ServerHello + Certificate received
~283ms:      TLS Finished — ENCRYPTED CHANNEL ESTABLISHED

~285ms:      HTTP GET /products?id=42 sent (encrypted)

~400ms:      Server processes request, queries DB, renders HTML

~500ms:      HTTP 200 OK response received
             TLS decrypt, gzip decompress, HTML parse begins
             Additional resource requests (CSS/JS/images)

~600-800ms:  Page fully rendered
```

> "That's your page load. Roughly 0.6 to 0.8 seconds — and each step is a protocol you now know deeply."

---

## PROTOCOL LAYER MAPPING (5 minutes)

**[INSTRUCTOR: Draw the full mapping table. This is the consolidation of all 13 lectures.]**

```
┌──────────────────────────────────────────────────────────────────┐
│ OSI Layer       │ What Ran                      │ Lecture        │
├─────────────────┼───────────────────────────────┼────────────────┤
│ Layer 7 - App   │ DNS, HTTP, TLS, cookies       │ L9, L13        │
│ Layer 6 - Pres  │ TLS encryption, gzip          │ L13            │
│ Layer 5 - Sess  │ TCP session management        │ L10            │
│ Layer 4 - Trans │ TCP (SYN/ACK/FIN), UDP (DNS)  │ L10            │
│ Layer 3 - Net   │ IP routing, NAT, TTL          │ L3, L4, L7, L8 │
│ Layer 2 - Link  │ Ethernet frames, ARP, MAC     │ L2, L11        │
│ Layer 1 - Phys  │ Wi-Fi 802.11, cables, signals │ L2             │
└─────────────────┴───────────────────────────────┴────────────────┘
```

```
┌──────────────────────────────────────────────────────────────────┐
│ Protocol        │ Purpose in this journey                        │
├─────────────────┼─────────────────────────────────────────────────┤
│ DHCP            │ Laptop got 192.168.1.5 when it joined Wi-Fi    │
│ ARP             │ Resolved gateway IP to MAC for Ethernet frame  │
│ DNS (UDP)       │ Resolved www.example.com to 93.184.216.34      │
│ IP              │ Routed packet across ~12 hops of the internet  │
│ NAT/PAT         │ Router translated 192.168.1.5:52341 outbound   │
│ TCP             │ Reliable, ordered delivery; handshake + close  │
│ TLS             │ Encrypted the HTTP request/response            │
│ HTTP/1.1        │ Carried the actual web request and response    │
│ BGP (hidden)    │ ISP routers agreed on how to route to the IP   │
│ OSPF (hidden)   │ Routing within ISP backbone                    │
└─────────────────┴─────────────────────────────────────────────────┘
```

> "Every lecture was one row in this table. Every protocol was solving a specific, concrete problem."

---

## WHERE PERFORMANCE COMES FROM (5 minutes)

> "Now that you understand every step, you can reason about performance. Where are the bottlenecks?"

```
Phase              Time    Optimization
─────────────────────────────────────────────────────────
Browser cache hit  0ms     Cache-Control, ETags
DNS resolution     70ms    TTL caching, DNS prefetch
TCP handshake      100ms   QUIC (0-RTT), keep-alive, pooling
TLS handshake      100ms   TLS 1.3 (1-RTT), session resumption
Network RTT        100ms   CDN (serve from closer server)
Server processing  150ms   Efficient DB queries, caching
Response transfer  100ms   gzip, HTTP/2 multiplexing, CDN
─────────────────────────────────────────────────────────
Total              ~620ms  Minimize each phase independently
```

> "When a tech lead says 'we need to reduce TTFB' (Time to First Byte), they mean the server processing + network phase. When they say 'add a CDN,' they mean reduce the RTT phase by serving from a geographically closer server. You now have the vocabulary and mental model to participate in these conversations."

---

## REVISION: COURSE SUMMARY (5 minutes)

**[INSTRUCTOR: Rapid-fire review. Point at each lecture.]**

| # | Topic | Core Concept |
|---|-------|-------------|
| L2 | Packets & Layers | Encapsulation: each layer adds a header; data link needs MAC |
| L3 | IP Addressing I | Binary → decimal; subnet mask ANDs; CIDR notation |
| L4 | IP Addressing II | VLSM; IPv6 compression; NAT intro; default gateway logic |
| L5 | Graph Algorithms I | BFS=min hops; Dijkstra=min cost; these ARE routing algorithms |
| L6 | Graph Algorithms II | Bellman-Ford=negative edges; Kruskal=MST; distance vs link-state |
| L7 | Routing I | LPM; routing table; static routes; TTL; MAC changes per hop |
| L8 | Routing II | RIP convergence; OSPF=link state; BGP=path vector; AS paths |
| L9 | DNS & HTTP | 13 root servers; DNS hierarchy; URL anatomy; HTTP methods/codes |
| L10 | TCP & UDP | SYN→SYN-ACK→ACK; flow control; fast retransmit; socket API |
| L11 | NAT, DHCP, ARP | DORA; NAT state table; home router = 5 devices in one |
| L12 | Troubleshooting | OSI-layer methodology; ping/traceroute/dig/ss/tcpdump/Wireshark |
| L13 | Complete Journey | Everything above runs in sequence for a single HTTPS request |

---

## CLOSING (3 minutes)

**[INSTRUCTOR: Return to the blank timeline drawn at the start. It should now be fully labeled.]**

> "You started this course with a question: 'How does the internet work?' You now have a complete answer — not a hand-wavy one, but a precise, layer-by-layer, protocol-by-protocol answer."

> "You understand why IP needs ARP to work. Why TCP needs to handshake. Why DNS exists. Why routers run BGP. Why NAT blocks peer-to-peer. Why DHCP uses broadcast. These aren't separate trivia — they're one interconnected system, and you understand the whole thing."

> "When you debug a network issue, you'll work bottom-up: check the cable, check the IP, check the port, check the application. When you design a system, you'll know which layer to optimize. When someone asks you how the internet works, you can actually explain it."

> "That's computer networks. Good luck on the exam."

---

## 📌 Class Discussion Questions

1. If the browser cache had a fresh response, how many of the 9 phases could be skipped entirely?
2. Why does TLS happen after TCP? Could it be combined?
3. What would happen if the NAT entry expired while you were in the middle of streaming a video?
4. If a BGP misconfiguration causes traffic to 93.184.216.34 to route to the wrong country, at which phase does the failure appear?
5. A user complains a website is slow. Walk through each phase and describe what tool you'd use to determine which phase is slow.

---

*End of Lecture 13 Script — Final Lecture, Computer Networks, Term 5, SST*
