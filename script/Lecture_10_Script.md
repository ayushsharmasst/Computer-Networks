# Lecture 10: TCP, UDP and Socket Programming
## Instructor Script — SST Computer Networks (Term 5)

---

### 🎯 Learning Objectives
By the end of this session, students will:
- Explain the transport layer's role (multiplexing, reliability)
- Compare TCP and UDP with concrete use cases
- Trace the TCP three-way handshake and four-way close
- Explain TCP's reliability mechanisms (acknowledgments, retransmission, flow control)
- Write a basic TCP client-server socket program in Python

**Duration:** ~90 minutes  
**Code Demo:** Python socket programming

---

## 📦 OPENING HOOK (5 minutes)

**[INSTRUCTOR: Start with a contrast]**

> "When you download a 2GB movie over HTTP, every single bit must arrive correctly. One flipped bit and your movie could be corrupted. Netflix, YouTube, WhatsApp file transfers — all NEED reliability."

> "But when you're on a live video call, if one frame gets lost — do you want your video frozen for 500ms while it retransmits? Or do you prefer a slightly glitchy frame for a millisecond and the stream to continue? You want the second option."

> "This is the fundamental tradeoff of the transport layer. **TCP** is reliable but heavy. **UDP** is lightweight but unreliable. Both have their place — it depends what your application needs."

---

## SECTION 1: The Transport Layer (8 minutes)

> "We've been thinking about IP, which routes packets from machine to machine. But a single machine runs HUNDREDS of processes — your browser, Spotify, Slack, your IDE. How does the OS know which incoming packet belongs to which application?"

**Answer: Ports**

> "A **port** is a 16-bit number (0–65535) that identifies a specific process/service on a host."

**Full addressing:**
```
IP address = identifies the machine
Port       = identifies the specific process on that machine

Full socket address: 192.168.1.5 : 52341
                     (IP)          (Port)
```

### Well-Known Ports (0–1023)

| Port | Service | Protocol |
|------|---------|---------|
| 20/21 | FTP | TCP |
| 22 | SSH | TCP |
| 25 | SMTP (email) | TCP |
| 53 | DNS | UDP (primarily) |
| 80 | HTTP | TCP |
| 443 | HTTPS | TCP |
| 3306 | MySQL | TCP |
| 5432 | PostgreSQL | TCP |
| 6379 | Redis | TCP |
| 27017 | MongoDB | TCP |

### Transport Layer Functions

> "The transport layer (TCP/UDP) does two critical things:
> 1. **Multiplexing** — combine data from multiple processes into IP packets
> 2. **Demultiplexing** — deliver incoming packets to the correct process using port numbers"

---

## SECTION 2: UDP — User Datagram Protocol (10 minutes)

### What is UDP?

> "UDP is the minimal transport protocol. It takes your data, adds source and destination ports, adds a checksum, and sends it. That's it."

**UDP Header:**
```
┌─────────────┬─────────────┐
│  Src Port   │  Dst Port   │  4 bytes
├─────────────┼─────────────┤
│   Length    │  Checksum   │  4 bytes
├─────────────────────────┤
│          Data            │
└─────────────────────────┘
```

**Total header overhead: 8 bytes** (vs TCP's 20+ bytes)

### UDP Properties

| Property | Value |
|---------|-------|
| Connection | Connectionless (no setup) |
| Reliability | None — packets may be lost, duplicated, reordered |
| Ordering | Not guaranteed |
| Error detection | Checksum (detects but doesn't correct) |
| Speed | Fast (no handshake, no ACKs, no retransmission) |
| Overhead | 8-byte header |

### When to Use UDP

| Use Case | Why UDP? |
|---------|---------|
| **DNS** | Single request-response; latency matters; retransmit if no reply |
| **Video streaming** | Old frame is useless; better to skip than to wait |
| **Online gaming** | Position updates are time-sensitive; stale updates are worse than missing |
| **VoIP/Video calls** | Real-time; one lost packet = glitch, not a pause |
| **DHCP** | Broadcast-based; TCP can't do broadcasts |
| **QUIC (HTTP/3)** | Google's new protocol — UDP underneath but with custom reliability |

**[INSTRUCTOR: Ask — "If UDP gives no reliability, why not just write reliability into your application?"  
Answer: Some applications do! QUIC does. But most use TCP because TCP's reliability is battle-tested and complex to reimplement correctly.]**

---

## SECTION 3: TCP — Transmission Control Protocol (20 minutes)

### TCP's Guarantees

> "TCP provides four guarantees:
> 1. **Reliable delivery** — data arrives or the sender knows it didn't
> 2. **In-order delivery** — bytes arrive in the same order they were sent
> 3. **Error checking** — corrupted segments are discarded and retransmitted
> 4. **Flow control** — sender doesn't overwhelm the receiver"

### TCP Header

```
┌────────────────┬────────────────┐
│   Src Port     │   Dst Port     │  4 bytes
├────────────────────────────────┤
│         Sequence Number        │  4 bytes
├────────────────────────────────┤
│        Acknowledgment Number   │  4 bytes
├────┬───────────┬────────────────┤
│HLEN│  Reserved │     Flags      │  2 bytes
│    │           │ URG ACK PSH    │
│    │           │ RST SYN FIN    │
├────────────────┬────────────────┤
│   Window Size  │   Checksum     │  4 bytes
├────────────────┴────────────────┤
│          Urgent Pointer         │  2 bytes
├─────────────────────────────────┤
│   Options (variable, padded)    │
├─────────────────────────────────┤
│         Data (payload)          │
└─────────────────────────────────┘
```

**Key fields:**
- **Sequence Number:** Identifies byte position in the stream
- **Acknowledgment Number:** Next byte the receiver expects
- **Flags:** SYN (establish), ACK (acknowledge), FIN (close), RST (reset/abort), PSH (push data to app)
- **Window Size:** How many bytes the receiver can accept (flow control)

### Three-Way Handshake — Connection Setup

**[INSTRUCTOR: Draw client and server, show the three arrows]**

```
Client                          Server
  |                               |
  |----SYN (seq=x)-------------→|   SYN: "I want to connect; my seq# starts at x"
  |                               |
  |←--SYN-ACK (seq=y, ack=x+1)--|   SYN-ACK: "OK, my seq# starts at y; I got your x"
  |                               |
  |----ACK (seq=x+1, ack=y+1)--→|   ACK: "Got it; connection established"
  |                               |
  |====ESTABLISHED=====================|
```

**Why sequence numbers start random?**
> "Initial sequence numbers (ISN) are random to prevent old packets from a previous connection being accepted as valid for a new connection. If they were always 0, an attacker could inject packets."

**What takes time in the handshake?**
> "Each arrow is a network round trip. On a connection from India to US (RTT ~200ms), the handshake alone takes 200ms before any data flows. This is why HTTPS connection reuse (Keep-Alive) and HTTP/2 multiplexing matter for performance."

### Four-Way Close — Connection Termination

> "TCP close is more complex because each direction of the stream is closed independently (full-duplex)."

```
Client                          Server
  |                               |
  |----FIN (seq=m)------------→|   "I'm done sending"
  |                               |
  |←--ACK (ack=m+1)------------|   "Got it. Server can still send data"
  |                               |
  |  (Server may still send data) |
  |                               |
  |←--FIN (seq=n)--------------|   "I'm also done sending"
  |                               |
  |----ACK (ack=n+1)----------→|   "Got it. Connection fully closed"
  |                               |
```

**TIME_WAIT state:**
> "After sending the final ACK, the client enters TIME_WAIT for 2×MSL (Maximum Segment Lifetime, typically 60s). This ensures:
> 1. The final ACK reached the server (if not, server retransmits FIN and client re-ACKs)
> 2. Any delayed packets from this connection die before a new connection uses the same port numbers"

### TCP Reliability — ACKs and Retransmission

> "TCP uses a byte-stream model. Every byte is numbered. The sender tracks which bytes have been acknowledged."

```
Sender sends:   [Bytes 1-1000]  [Bytes 1001-2000]  [Bytes 2001-3000]
                      ↓                 ↓                   ↓
Receiver gets:  [ACK 1001]       [ACK 2001]          [ACK 3001]

If Bytes 2001-3000 gets lost:
Receiver:       [ACK 2001] (cumulative — still wants byte 2001)
                [ACK 2001] (duplicate ACK — "I got something but still want 2001!")
                [ACK 2001] (duplicate ACK again)

Three duplicate ACKs → sender retransmits from 2001 (Fast Retransmit)
```

**Retransmission triggers:**
1. **RTO timeout:** Receiver never sends ACK → sender retransmits after timeout
2. **Fast Retransmit:** 3 duplicate ACKs received → retransmit immediately (faster than timeout)

### Flow Control — Sliding Window

> "What if the sender sends data faster than the receiver can process it? The receiver's buffer would overflow and data would be lost."

**Solution: Receiver advertises its buffer capacity in the Window Size field**

```
Receiver tells sender:
"My window size is 65535 bytes" → sender can have up to 65535 bytes in-flight (unACKed)

As receiver processes data and frees buffer:
"Window size: 100000 bytes" → sender can send more

If buffer fills:
"Window size: 0" → sender must stop and wait (Zero Window Probe follows)
```

**Conceptually (simplified):**
```
Send buffer: [=======XXXXXXXXXXXXXXXXXX..............                ]
              Acked   In-flight(unACKed) Can send    Can't send yet
              ←———————— Window size ————————→
```

---

## SECTION 4: Socket Programming (20 minutes)

### What is a Socket?

> "A **socket** is an abstraction provided by the OS that lets an application send/receive data over the network. Think of it as an endpoint of a connection — a file-like object you can read and write."

**Socket address:** IP + Port + Protocol (TCP or UDP)

### TCP Socket Programming — Client-Server Model

**[INSTRUCTOR: Live code this step by step]**

**Server Side:**
```python
import socket

# 1. Create a TCP socket (SOCK_STREAM = TCP)
server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# 2. Bind to address and port
server_socket.bind(('0.0.0.0', 8080))  # Listen on all interfaces, port 8080

# 3. Start listening (max 5 pending connections in queue)
server_socket.listen(5)
print("Server listening on port 8080...")

# 4. Accept loop
while True:
    # Blocks until a client connects
    client_socket, client_address = server_socket.accept()
    print(f"Connection from: {client_address}")
    
    # 5. Receive data
    data = client_socket.recv(1024)  # receive up to 1024 bytes
    print(f"Received: {data.decode()}")
    
    # 6. Send response
    response = f"Echo: {data.decode()}"
    client_socket.send(response.encode())
    
    # 7. Close connection
    client_socket.close()
```

**Client Side:**
```python
import socket

# 1. Create a TCP socket
client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# 2. Connect to server (triggers three-way handshake)
client_socket.connect(('127.0.0.1', 8080))

# 3. Send data
message = "Hello, Server!"
client_socket.send(message.encode())

# 4. Receive response
response = client_socket.recv(1024)
print(f"Server says: {response.decode()}")

# 5. Close (triggers four-way close)
client_socket.close()
```

**[INSTRUCTOR: Run server in one terminal, client in another. Show the echo working.]**

### UDP Socket Programming

```python
# UDP Server
import socket
server = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)  # SOCK_DGRAM = UDP
server.bind(('0.0.0.0', 9090))

data, addr = server.recvfrom(1024)  # No accept() — connectionless!
print(f"Got {data.decode()} from {addr}")
server.sendto(f"Got it!".encode(), addr)

# UDP Client
client = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
client.sendto("Hello UDP".encode(), ('127.0.0.1', 9090))
data, server_addr = client.recvfrom(1024)
print(data.decode())
```

**[INSTRUCTOR: Note key difference — no `connect()`, no `accept()`. UDP just sends datagrams.]**

### Handling Multiple Clients — Threading

```python
import socket, threading

def handle_client(client_socket, addr):
    data = client_socket.recv(1024)
    client_socket.send(f"Echo: {data.decode()}".encode())
    client_socket.close()

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(('0.0.0.0', 8080))
server.listen(10)

while True:
    client, addr = server.accept()
    t = threading.Thread(target=handle_client, args=(client, addr))
    t.start()
```

**[INSTRUCTOR: Ask — "What's the problem with spawning a thread per client?"  
Answer: resource exhaustion at high scale. Production servers use async I/O (event loops), thread pools, or non-blocking sockets.]**

---

## SECTION 5: TCP vs UDP — Decision Framework (5 minutes)

| Choose TCP when | Choose UDP when |
|----------------|----------------|
| Data must arrive completely | Timeliness > completeness |
| Order matters | Occasional loss is acceptable |
| Application can tolerate retransmit delay | Low overhead needed |
| Example: HTTP, FTP, SSH, databases | Example: DNS, video call, games, DHCP |

**Modern trend — QUIC:**
> "QUIC (HTTP/3) uses UDP but implements its own reliability at the application layer. Benefits: no head-of-line blocking (individual streams can lose packets without blocking others), faster connection setup (0-RTT resumption), built-in TLS."

---

## SUMMARY (5 minutes)

```
✅ Transport layer: multiplexing (ports), demultiplexing
✅ Ports: 0-1023 well-known, 1024-49151 registered, 49152-65535 ephemeral
✅ UDP: connectionless, 8-byte header, no reliability, no ordering
    → Use for: DNS, video, gaming, real-time
✅ TCP: connection-oriented, reliable, ordered, flow-controlled
    → Three-way handshake: SYN → SYN-ACK → ACK
    → Four-way close: FIN → ACK → FIN → ACK
    → Reliability via ACKs, retransmission, sliding window
✅ Sockets: AF_INET+SOCK_STREAM=TCP; AF_INET+SOCK_DGRAM=UDP
    → Server: socket→bind→listen→accept→recv/send→close
    → Client: socket→connect→send/recv→close
```

---

*End of Lecture 10 Script*
