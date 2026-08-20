# Lecture 13: The Complete Internet Journey — From URL to Response
## Student Notes — SST Computer Networks (Term 5)

---

## 🎯 Learning Objectives
- Trace the complete journey of an HTTPS request end-to-end
- Map every phase to the OSI layer and protocol that handles it
- Identify where performance bottlenecks occur and how they are addressed
- Connect all course topics into one unified mental model

---

## The Scenario

```
User types: https://www.example.com/products?id=42

Laptop:     192.168.1.5  (assigned by DHCP)
Gateway:    192.168.1.1  (home router)
Router WAN: 203.0.113.5  (from ISP via DHCP)
Server:     93.184.216.34 (behind load balancer)
```

---

## Phase 1: Browser Cache Check

Before any network traffic, the browser checks its caches:

```
1. Memory cache (same session) → if fresh, return immediately, no network
2. Disk cache (HTTP cache) → if within Cache-Control max-age, serve from disk
   → If stale, send conditional GET: If-None-Match: "etag123"
     → Server replies 304 Not Modified → use cached body, no transfer
3. No cache → full request required
```

**Key headers that control caching:**

| Header | Effect |
|--------|--------|
| `Cache-Control: max-age=300` | Cache for 5 minutes |
| `Cache-Control: no-cache` | Always revalidate with server |
| `ETag: "abc123"` | Fingerprint; used in `If-None-Match` conditional requests |
| `Expires:` | Absolute expiry time (older; prefer Cache-Control) |

---

## Phase 2: DNS Resolution

### Cache Chain (checked in order)

```
1. Browser DNS cache
2. OS resolver cache
3. /etc/hosts file
4. Local DNS resolver (router at 192.168.1.1:53)
5. Recursive resolver (e.g. 8.8.8.8)
6. Root server → TLD server → Authoritative nameserver
```

### Full Resolution Trace

```
OS → Router (forwarder) → 8.8.8.8 (recursive resolver)

8.8.8.8 → Root server: "Who handles .com?"
           Root: "Ask TLD NS at 192.5.6.30"

8.8.8.8 → .com TLD: "Who handles example.com?"
           TLD: "Ask auth NS at 205.251.196.1"

8.8.8.8 → Auth NS: "What is www.example.com?"
           Auth NS: "93.184.216.34 (TTL 300s)"

8.8.8.8 returns 93.184.216.34 to OS
OS caches for 300 seconds
```

### DNS Packet Details

```
Protocol: UDP
Source port: 54321 (ephemeral)
Destination port: 53 (DNS)
Query type: A (IPv4 address)
```

**DNS record types (recap):**

| Type | Meaning | Example |
|------|---------|---------|
| A | IPv4 address | 93.184.216.34 |
| AAAA | IPv6 address | 2001:db8::1 |
| CNAME | Canonical name (alias) | www → example.com |
| MX | Mail server | mail.example.com |
| NS | Authoritative nameservers | ns1.example.com |
| TXT | Arbitrary text (SPF, DKIM) | "v=spf1 ..." |

---

## Phase 3: ARP — Getting the Gateway's MAC

```
Routing decision:
  "Is 93.184.216.34 in 192.168.1.0/24?" → No
  → Send to default gateway: 192.168.1.1

ARP cache lookup:
  → Hit: router MAC = 11:22:33:44:55:66 (cached from boot)
  → Miss: ARP broadcast → ARP reply → cache updated

Ethernet frame built:
  Dst MAC: 11:22:33:44:55:66  (router)
  Src MAC: AA:BB:CC:DD:EE:FF  (laptop)
  IP: Src=192.168.1.5, Dst=93.184.216.34
```

> **Key rule:** MAC addresses change at every hop. IP addresses stay the same across the full internet path.

---

## Phase 4: TCP Three-Way Handshake

Connection to 93.184.216.34:443:

```
Laptop                              Server (Load Balancer)

  │── SYN (seq=x) ─────────────────▶│
  │◀─ SYN-ACK (seq=y, ack=x+1) ────│
  │── ACK (ack=y+1) ───────────────▶│
  
  ══════ ESTABLISHED ═══════
```

**What NAT does during SYN:**

```
Laptop sends:     Src=192.168.1.5:52341, Dst=93.184.216.34:443
Router rewrites:  Src=203.0.113.5:10001, Dst=93.184.216.34:443
NAT table entry:  203.0.113.5:10001 ↔ 192.168.1.5:52341
```

**Cost:** 1 full RTT before any data can flow.

---

## Phase 5: TLS Handshake

Runs inside the TCP connection. Establishes an encrypted channel.

### TLS 1.3 Handshake (1-RTT)

```
Client                              Server

  │── ClientHello ─────────────────▶│
  │  TLS version supported: 1.3     │
  │  Cipher suites: [AES-GCM, ...]  │
  │  SNI: "www.example.com"         │
  │  Client random (nonce)          │
  │                                  │
  │◀─ ServerHello ──────────────────│
  │  Chosen cipher: AES-256-GCM     │
  │  Server Certificate             │
  │  Certificate chain (→ root CA)  │
  │  Server random                  │
  │  Server Finished (MAC)          │
  │                                  │
  │── Client Finished ─────────────▶│
  │  Key exchange complete          │
  │                                  │
  ══════ ENCRYPTED CHANNEL ══════════
```

### Key TLS Concepts

| Concept | Explanation |
|---------|-------------|
| **SNI** (Server Name Indication) | Tells server which domain's certificate to present — allows one IP to host multiple TLS domains |
| **Certificate** | Contains server's public key + domain name, signed by a trusted CA |
| **Certificate chain** | Server cert → intermediate CA → root CA (trusted by OS/browser) |
| **Session keys** | Both sides independently compute the same AES key using Diffie-Hellman; the actual key is never transmitted over the wire |
| **Forward secrecy** | A new ephemeral DH key pair is generated per session — even if the server's private key leaks later, past sessions cannot be decrypted |
| **0-RTT resumption** | TLS 1.3 allows sending data on the first packet for returning clients using a pre-shared session ticket (carries replay-attack risk for non-idempotent requests) |

**Why TLS can't replace TCP:** TLS provides confidentiality and integrity, but still relies on TCP for reliable, ordered delivery of its records.

### How Diffie-Hellman Key Exchange Works

The most common student question: "If neither side sends the key, how do both sides end up with the same key?"

**The paint-mixing analogy:**

```
Setup: Everyone knows the starting color = YELLOW

Alice has a secret: RED    (never shown to anyone)
Bob has a secret:   BLUE   (never shown to anyone)

Step 1: Alice mixes Yellow + Red = ORANGE → sends ORANGE to Bob
Step 2: Bob mixes   Yellow + Blue = GREEN → sends GREEN to Alice

An attacker (Eve) sees: Yellow, Orange, Green
Eve cannot determine Red or Blue from these.

Step 3: Alice takes Green + her secret Red  = BROWN (= Y + R + B)
Step 4: Bob takes Orange + his secret Blue  = BROWN (= Y + R + B)

Both have BROWN — the shared secret — without ever transmitting it.
```

**The actual math (ECDH):**

All parties agree on public parameters: prime p, generator g.

```
Alice picks private 'a' (random, never shared)
Alice computes: A = g^a mod p  →  sends A to Bob

Bob picks private 'b' (random, never shared)  
Bob computes:   B = g^b mod p  →  sends B to Alice

Alice computes: B^a mod p = (g^b)^a mod p = g^(ab) mod p
Bob computes:   A^b mod p = (g^a)^b mod p = g^(ab) mod p

Both have: g^(ab) mod p — the shared secret
```

An attacker sees g^a and g^b but cannot compute g^(ab) without solving the **Discrete Logarithm Problem** — no efficient algorithm exists for this with proper parameters.

The shared secret g^(ab) is then passed through a **Key Derivation Function (KDF)** to produce the actual AES-256 session keys used to encrypt the HTTP data.

---

## Phase 6: HTTP Request

The actual application request, sent encrypted over TLS:

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

### Request Header Breakdown

| Header | Purpose |
|--------|---------|
| `Host` | Required in HTTP/1.1; identifies the virtual host |
| `Accept-Encoding: gzip` | Browser can decompress — server should compress response |
| `Cookie` | Browser auto-attaches all cookies for this domain |
| `Connection: keep-alive` | Reuse TCP connection for subsequent requests |

### Full Packet Stack at This Point

```
┌────────────────────────────────────┐
│  HTTP/1.1 GET (plaintext message)  │  Layer 7 — Application
├────────────────────────────────────┤
│  TLS Record (AES-256 encrypted)    │  Layer 6 — Presentation
├────────────────────────────────────┤
│  TCP Segment (Seq/Ack/Port 443)    │  Layer 4 — Transport
├────────────────────────────────────┤
│  IP Packet (192.168.1.5 → 93....)  │  Layer 3 — Network
├────────────────────────────────────┤
│  Ethernet Frame (MAC → MAC)        │  Layer 2 — Data Link
└────────────────────────────────────┘
```

---

## Phase 7: Server-Side Processing

### Load Balancer

```
Receives:    Dst=93.184.216.34:443
Terminates:  TLS (SSL offloading — backend receives plain HTTP)
Routes:      GET /products → backend at 10.10.0.5:8080
Adds header: X-Forwarded-For: 203.0.113.5 (real client IP for logging)
```

**SSL Offloading:** LB handles TLS; backend servers handle only HTTP. Reduces CPU load on backends and simplifies certificate management.

### Web Server Processing Steps

```
1. Parse: GET /products?id=42
2. Route: /products handler
3. Auth: session_id=abc123 → User #789 (authenticated)
4. Query: SELECT * FROM products WHERE id=42
5. Render: HTML template + product data
6. Build: HTTP response with headers
```

### HTTP Response

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Content-Length: 4821
Cache-Control: max-age=300, public
ETag: "a3f8b2c1"
Set-Cookie: last_seen=product_42; Max-Age=86400; Secure; HttpOnly

[gzip-compressed HTML]
```

### Response Header Breakdown

| Header | Purpose |
|--------|---------|
| `Content-Encoding: gzip` | Body is compressed; browser decompresses before rendering |
| `Cache-Control: max-age=300` | Cache for 5 min; avoids repeat requests |
| `ETag: "a3f8b2c1"` | Fingerprint; enables 304 Not Modified on next visit |
| `Set-Cookie: Secure; HttpOnly` | Secure = HTTPS only; HttpOnly = no JS access (XSS protection) |

---

## Phase 8: Return Journey

```
Web Server → Load Balancer
  Plain HTTP response (internal network, no TLS needed)

Load Balancer → Internet
  TLS encrypts response
  TCP segment: Src=93.184.216.34:443, Dst=203.0.113.5:10001

Router NAT reverse lookup:
  Port 10001 → 192.168.1.5:52341
  Rewrites Dst to 192.168.1.5:52341
  ARP for laptop's MAC (cached)
  Delivers on LAN

Laptop:
  TCP reassembles segments in order
  TLS decrypts record
  HTTP parses headers + body
  gzip decompresses body
  Browser renders HTML
```

### Subsequent Resource Requests

```
HTML references → browser makes additional requests:
  GET /static/style.css   → same TCP connection (keep-alive)
  GET /static/app.js      → same connection (pipelined in HTTP/2)
  GET /images/product.jpg → possibly a CDN (different server, closer)
```

---

## Phase 9: Connection Close

```
Laptop                              Server

  │── FIN ───────────────────────▶│
  │◀─ ACK ────────────────────────│
  │◀─ FIN ────────────────────────│
  │── ACK ───────────────────────▶│
  
  Laptop: TIME_WAIT (~60s)
  Router: NAT entry cleaned up after TCP close detected
```

---

## Complete Timeline

```
  0ms   Browser cache miss → begin resolution
  1ms   OS cache miss, /etc/hosts miss
  5ms   DNS query sent to router (UDP:53)
 30ms   Recursive DNS: root query
 50ms   Recursive DNS: TLD query
 70ms   Recursive DNS: authoritative answer → 93.184.216.34
 72ms   ARP check (likely cached → ~0ms delay)
 75ms   TCP SYN sent, NAT creates entry
175ms   SYN-ACK received
177ms   ACK sent — ESTABLISHED
180ms   TLS ClientHello sent
280ms   TLS ServerHello + cert received
283ms   TLS ESTABLISHED (encrypted)
285ms   HTTP GET /products?id=42 sent
400ms   Server processes, DB query, render
500ms   HTTP 200 + HTML received, decrypted, decompressed
600–800ms  CSS/JS/images loaded, page fully rendered
```

---

## Protocol-to-Layer Mapping

| Phase | Protocol | OSI Layer | Lecture |
|-------|---------|-----------|---------|
| IP address assignment | DHCP | L7/L3 | L11 |
| Gateway MAC discovery | ARP | L2 | L11 |
| Name resolution | DNS (UDP) | L7 | L9 |
| Reliable transport | TCP | L4 | L10 |
| NAT address translation | NAT/PAT | L3 | L4, L11 |
| Encryption | TLS | L6 | L13 |
| Web request | HTTP/1.1 | L7 | L9 |
| Internet routing | BGP | L3 | L8 |
| ISP internal routing | OSPF | L3 | L8 |
| Packet delivery | IP | L3 | L3, L7 |
| Frame delivery | Ethernet | L2 | L2 |

---

## Performance Phases and Optimizations

| Phase | Typical Cost | Optimization |
|-------|-------------|-------------|
| Browser cache | 0ms (hit) | `Cache-Control`, ETags |
| DNS resolution | 50–100ms | Low TTL → faster propagation; high TTL → fewer queries |
| TCP handshake | 1 RTT | QUIC/HTTP/3 (0-RTT); connection pooling |
| TLS handshake | 1 RTT (TLS 1.3) | Session resumption; TLS 1.3 vs 1.2 |
| Network RTT | ~100ms cross-continent | CDN (serve from edge servers) |
| Server processing | 50–200ms | Efficient DB queries, in-memory cache |
| Response transfer | varies | gzip, HTTP/2 multiplexing |

**TTFB (Time to First Byte):** Time from sending the request to receiving the first byte of the response. Includes network RTT + server processing. A common performance metric.

---

## Where Things Go Wrong — Troubleshooting Map

| Symptom | Likely Phase | Tool | Fix |
|---------|-------------|------|-----|
| ERR_NAME_NOT_RESOLVED | DNS | `dig`, `nslookup` | Fix DNS config, flush cache |
| ERR_CONNECTION_REFUSED | TCP/Port | `ss -tulpn` | Service not running or wrong port |
| SSL_ERROR | TLS | Browser devtools, `openssl s_client` | Expired cert, wrong hostname |
| 404 Not Found | HTTP/App | Browser devtools | Wrong URL, path not registered |
| 403 Forbidden | HTTP/Auth | Server logs | Missing permissions/auth |
| 500 Internal Server Error | App | Server/application logs | Bug in server code |
| Slow TTFB | Server processing | Monitoring, DB query analysis | Optimize queries, add caching |
| Slow page load | Multiple | Waterfall in devtools | Cache, CDN, compression |
| Intermittent drops | Network | `mtr`, `ping -c 100` | Packet loss in path |

---

## Course Summary

| Lecture | Topic | Core Concept |
|---------|-------|-------------|
| L2 | Packets & Layers | Encapsulation; each layer adds a header; Ethernet frame |
| L3 | IP Addressing I | Binary; subnet mask AND; CIDR; private ranges |
| L4 | IP Addressing II | VLSM; IPv6 compression; default gateway; NAT intro |
| L5 | Graph Algorithms I | BFS=min hops, Dijkstra=min cost; both ARE routing algorithms |
| L6 | Graph Algorithms II | Bellman-Ford=negative edges; Kruskal=MST; DV vs LS routing |
| L7 | Routing I | LPM; routing table; static routes; TTL; MAC changes per hop |
| L8 | Routing II | RIP; count-to-infinity; OSPF (link state); BGP (path vector) |
| L9 | DNS & HTTP | DNS hierarchy; record types; URL anatomy; HTTP methods/codes |
| L10 | TCP & UDP | Handshake; flow control; fast retransmit; socket programming |
| L11 | NAT, DHCP, ARP | DORA; NAT state table; home router = 5 devices in one |
| L12 | Troubleshooting | OSI methodology; ping/traceroute/dig/ss/tcpdump/Wireshark |
| L13 | Complete Journey | All above protocols run in sequence for one HTTPS request |

---

## 🧠 Quick Self-Check Questions

1. A browser loads a page in 50ms on second visit but 650ms on first visit. Which phases are skipped on the second visit?
2. Why does the TLS handshake happen after the TCP handshake, not before?
3. You see a 304 Not Modified response. What did the browser send in its request, and what does this mean for the response body?
4. The load balancer adds an `X-Forwarded-For` header. Why is this needed? What does the backend see without it?
5. If the NAT state table entry for a TCP connection expires mid-stream, what happens to the connection?
6. Map each phase of the HTTPS request to its OSI layer. Which layer does TLS operate at?
7. A website has high TTFB (800ms) but fast DNS and TCP. Which phase is slow, and what tool would you use to confirm?
8. Why do subsequent CSS/JS requests on the same page not require new TCP handshakes?

---

*Lecture 13 of 13 — Computer Networks, Term 5, SST*
