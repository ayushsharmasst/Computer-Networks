# Lecture 10: TCP, UDP and Socket Programming
## Student Notes — SST Computer Networks (Term 5)

---

## 🎯 Learning Objectives
- Explain the transport layer's role (ports, multiplexing)
- Compare TCP and UDP with concrete use cases
- Trace TCP three-way handshake and four-way close
- Explain TCP reliability mechanisms
- Write a basic TCP/UDP client-server socket program in Python

---

## 1. Transport Layer Fundamentals

### The Problem: Multiple Applications, One Machine

IP routes packets to a machine, but a machine runs many applications simultaneously (browser, Slack, Spotify, SSH). Ports solve this.

### Ports

A **port** is a 16-bit integer (0–65535) identifying a specific process/service on a host.

```
Full socket address = IP address : Port number
Example:           192.168.1.5 : 52341
```

### Port Ranges

| Range | Name | Use |
|-------|------|-----|
| 0–1023 | Well-known | Standard services (HTTP:80, HTTPS:443) |
| 1024–49151 | Registered | Applications (MySQL:3306, Redis:6379) |
| 49152–65535 | Ephemeral | Client-side temporary ports |

### Common Well-Known Ports

| Port | Service | Protocol |
|------|---------|---------|
| 22 | SSH | TCP |
| 25 | SMTP | TCP |
| 53 | DNS | UDP/TCP |
| 80 | HTTP | TCP |
| 443 | HTTPS | TCP |
| 3306 | MySQL | TCP |
| 5432 | PostgreSQL | TCP |
| 6379 | Redis | TCP |

### Transport Layer Functions

- **Multiplexing**: combine data from multiple processes → IP packets (adding port headers)
- **Demultiplexing**: deliver received packets to correct process (using destination port)

---

## 2. UDP — User Datagram Protocol

### UDP Header (8 bytes total)

```
 0      7 8     15 16    23 24    31
┌────────────────┬────────────────┐
│  Source Port   │  Dest Port     │
├────────────────┼────────────────┤
│    Length      │   Checksum     │
└────────────────┴────────────────┘
│              Data               │
```

### Properties

| Property | UDP |
|---------|-----|
| Connection | Connectionless — no setup needed |
| Reliability | None — packets may be lost, duplicated, reordered |
| Ordering | Not guaranteed |
| Error detection | Checksum (detects errors but doesn't correct) |
| Overhead | 8-byte header only |
| Speed | Fast (no handshake, no ACKs, no retransmission) |

### When to Use UDP

| Application | Reason |
|------------|--------|
| DNS | Single request-response; low latency needed |
| Video streaming | Old frame is useless; skip > wait |
| Online gaming | Position updates are time-sensitive |
| VoIP / video calls | Real-time; one lost packet = brief glitch, not pause |
| DHCP | Broadcast-based; TCP can't broadcast |
| QUIC / HTTP/3 | UDP base; custom reliability in the app layer |

---

## 3. TCP — Transmission Control Protocol

### TCP's Four Guarantees

1. **Reliable delivery**: data arrives or sender knows it didn't
2. **In-order delivery**: bytes arrive in the order they were sent
3. **Error checking**: corrupted segments are detected and retransmitted
4. **Flow control**: sender doesn't overwhelm receiver's buffer

### Key TCP Header Fields

| Field | Size | Purpose |
|-------|------|---------|
| Source Port | 16-bit | Sending process port |
| Destination Port | 16-bit | Receiving process port |
| **Sequence Number** | 32-bit | Byte position in data stream |
| **Acknowledgment Number** | 32-bit | Next byte expected |
| **Flags** | 6-bit | SYN, ACK, FIN, RST, PSH, URG |
| **Window Size** | 16-bit | Receiver's available buffer (flow control) |
| Checksum | 16-bit | Error detection |

---

## 4. TCP Three-Way Handshake (Connection Setup)

```
Client                              Server

  │──── SYN (seq=x) ────────────────▶│
  │    "I want to connect;            │
  │     my starting seq# = x"        │
  │                                   │
  │◀─── SYN-ACK (seq=y, ack=x+1) ───│
  │    "OK, my starting seq# = y;    │
  │     got your x, expecting x+1"  │
  │                                   │
  │──── ACK (seq=x+1, ack=y+1) ────▶│
  │    "Got it; connection established│
  │     expecting y+1 from you"      │
  │                                   │
  ╔═══════ ESTABLISHED ═══════════════╗
```

**Why random initial sequence numbers?**  
Prevents old packets from a previous connection being mistaken for new ones.

**RTT impact:**  
Each arrow = one network round trip. US-India RTT ~200ms → handshake alone takes ~200ms before any data.

---

## 5. TCP Four-Way Close (Connection Termination)

```
Client                              Server

  │──── FIN (seq=m) ──────────────▶│
  │    "I'm done sending"           │
  │                                 │
  │◀─── ACK (ack=m+1) ────────────│
  │    "Got your FIN; server may   │
  │     still send data"           │
  │                                 │
  │   (Server finishes sending)    │
  │                                 │
  │◀─── FIN (seq=n) ──────────────│
  │    "I'm also done sending"     │
  │                                 │
  │──── ACK (ack=n+1) ────────────▶│
  │    "Got it; connection done"   │
  │                                 │
  │  [Client enters TIME_WAIT]     │
```

**TIME_WAIT (2×MSL, ~60 seconds):**
- Ensures final ACK reached server (if not, server retransmits FIN)
- Allows delayed packets from this connection to expire

---

## 6. TCP Reliability Mechanisms

### Acknowledgments and Retransmission

TCP numbers every byte. The receiver ACKs the next byte it expects.

```
Sender sends:  [seq 1–1000]  [seq 1001–2000]  [seq 2001–3000]
Receiver ACKs: [ACK 1001]   [ACK 2001]        [ACK 3001]

If seq 2001–3000 is lost:
Receiver keeps getting data after 2000, sends:
  [ACK 2001]  ← "Still waiting for byte 2001"
  [ACK 2001]  ← duplicate
  [ACK 2001]  ← 3rd duplicate ACK

3 duplicate ACKs → sender does Fast Retransmit (retransmit seq 2001 immediately)
```

**Retransmission triggers:**
1. **RTO timeout**: no ACK received after a timeout period → retransmit
2. **Fast Retransmit**: 3 duplicate ACKs received → retransmit immediately (faster than waiting for timeout)

### Flow Control — Sliding Window

Receiver tells sender how much buffer space it has:

```
Window Size in ACK = "You can send up to N bytes without waiting for ACK"

Receiver buffer large:  Window = 65535 → sender can pipeline many segments
Receiver buffer full:   Window = 0    → sender must pause (Zero Window)
```

**Effect:**
```
[Bytes Acknowledged] [Bytes In-Flight] [Bytes Sendable] [Wait]
                     └───── Window ────────────────┘
```

---

## 7. Socket Programming in Python

### What is a Socket?

A **socket** is an OS-level abstraction for a network communication endpoint.

`socket(family, type)`:
- `AF_INET` = IPv4 address family
- `SOCK_STREAM` = TCP (reliable, connected)
- `SOCK_DGRAM` = UDP (unreliable, connectionless)

### TCP Server

```python
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)  # Reuse port

server.bind(('0.0.0.0', 8080))   # Listen on all interfaces, port 8080
server.listen(5)                   # Allow up to 5 pending connections
print("Server listening on port 8080")

while True:
    client_sock, client_addr = server.accept()  # Blocks until client connects
    print(f"Connection from {client_addr}")
    
    data = client_sock.recv(1024)              # Receive up to 1024 bytes
    print(f"Received: {data.decode()}")
    
    client_sock.send(f"Echo: {data.decode()}".encode())
    client_sock.close()
```

### TCP Client

```python
import socket

client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.connect(('127.0.0.1', 8080))   # Three-way handshake happens here

client.send("Hello, Server!".encode())
response = client.recv(1024)
print(f"Server replied: {response.decode()}")

client.close()   # Four-way close happens here
```

### TCP Server/Client Lifecycle

```
SERVER                       CLIENT
socket()                     socket()
bind()                       
listen()                     
accept() ─────────────────── connect()   ← Handshake
recv() ◀──────────────────── send()
send() ──────────────────────▶ recv()
close()                      close()     ← 4-way close
```

### UDP Server

```python
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
server.bind(('0.0.0.0', 9090))

data, addr = server.recvfrom(1024)    # Receive datagram + sender address
print(f"Got '{data.decode()}' from {addr}")
server.sendto("Got it!".encode(), addr)
```

### UDP Client

```python
import socket

client = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
client.sendto("Hello UDP".encode(), ('127.0.0.1', 9090))
data, server = client.recvfrom(1024)
print(data.decode())
```

### Multi-client TCP Server (Threading)

```python
import socket, threading

def handle_client(sock, addr):
    data = sock.recv(1024)
    sock.send(f"Echo: {data.decode()}".encode())
    sock.close()

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(('0.0.0.0', 8080))
server.listen(10)

while True:
    client, addr = server.accept()
    t = threading.Thread(target=handle_client, args=(client, addr))
    t.daemon = True
    t.start()
```

---

## 8. TCP vs UDP Comparison

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented (handshake) | Connectionless |
| Reliability | Guaranteed (ACK + retransmit) | None |
| Ordering | Guaranteed | Not guaranteed |
| Error detection | Checksum + retransmit | Checksum only |
| Header size | 20+ bytes | 8 bytes |
| Speed | Slower (overhead) | Faster |
| Flow control | Yes (window size) | No |
| Use cases | HTTP, FTP, SSH, databases | DNS, video, games, VoIP |

---

## 📌 Key Takeaways

1. **Ports** = process identifiers; transport layer multiplexes/demultiplexes using ports
2. **UDP**: connectionless, 8-byte header, no reliability — use when speed > reliability
3. **TCP**: connection-oriented, reliable, ordered, flow-controlled — use when correctness matters
4. **Three-way handshake**: SYN → SYN-ACK → ACK establishes TCP connection
5. **Four-way close**: FIN → ACK → FIN → ACK gracefully terminates connection
6. **Retransmission**: triggered by timeout OR 3 duplicate ACKs (fast retransmit)
7. **Flow control**: receiver's window size limits how much sender can send unacknowledged
8. **Python sockets**: `SOCK_STREAM` for TCP, `SOCK_DGRAM` for UDP; server binds+listens+accepts; client connects

---

## 🧠 Quick Self-Check Questions

1. What is the purpose of the Window Size field in the TCP header?
2. Why does TCP use random initial sequence numbers?
3. In the three-way handshake, which side initiates? What flags are set in each packet?
4. What triggers TCP's Fast Retransmit? How is it different from timeout retransmission?
5. You're building a live leaderboard for a game that updates every 100ms. TCP or UDP? Why?
6. Write the server-side socket code to echo any received UDP message back to the sender.
7. What is the TIME_WAIT state and why does it exist?
8. If recv() returns 0 bytes on a TCP socket, what does that mean?

---

*Lecture 10 of 13 — Computer Networks, Term 5, SST*
