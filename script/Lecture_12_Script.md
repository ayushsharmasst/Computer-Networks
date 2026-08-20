# Lecture 12: Network Troubleshooting
## Instructor Script — SST Computer Networks (Term 5)

---

### 🎯 Learning Objectives
By the end of this session, students will:
- Use ping, traceroute, nslookup/dig to diagnose connectivity problems
- Read netstat/ss output to understand socket states
- Capture and interpret packets with tcpdump and Wireshark
- Follow a structured troubleshooting methodology
- Diagnose common networking problems

**Duration:** ~90 minutes  
**Demo:** Heavy — run all tools live

---

## 📦 OPENING HOOK (5 minutes)

**[INSTRUCTOR: Read this scenario aloud]**

> "It's 2am. Your company's main API is down. Customers are angry. Your CEO is calling. You log in to the server — the application is running. You try to curl the endpoint — nothing. You try to ping the server from your laptop — it works. You try to ping port 443 — it doesn't respond. Where is the problem?"

> "This is a real situation every engineer faces. Debugging network problems without tools is guesswork. With tools, it becomes systematic. Today we're going to build your network debugging toolkit — from the simplest `ping` to reading raw packet captures."

---

## SECTION 1: Diagnostic Methodology (5 minutes)

**[INSTRUCTOR: Write the OSI-layer approach on board]**

> "Network problems follow layers. Always start from the bottom and work up."

```
Layer 7 (Application): Is the application responding correctly?
Layer 4 (Transport):   Can I connect on the right port?
Layer 3 (Network):     Can I reach the IP?
Layer 2 (Data Link):   Is my network interface up?
Layer 1 (Physical):    Is the cable plugged in? Is Wi-Fi associated?
```

> "Most engineers make the mistake of jumping straight to application logs. Start physical first — is the interface up? Then check ping. Then check the port. Then check the application."

**Systematic checklist:**
1. Can you ping the machine? (L3 connectivity)
2. Can you reach the specific port? (L4 connectivity)
3. Does the service respond? (L7 application)
4. DNS resolving correctly?
5. No firewall blocking?

---

## SECTION 2: ping (12 minutes)

### What ping Does

> "ping sends **ICMP Echo Request** packets to a host and measures the round-trip time (RTT). The target host responds with ICMP Echo Reply."

```bash
# Basic ping
ping google.com
ping 8.8.8.8

# Ping with count
ping -c 4 google.com    # Send exactly 4 packets (Linux/Mac)
ping -n 4 google.com    # Windows

# Ping with custom size
ping -s 1000 google.com   # Send 1000-byte packets

# Ping with interval
ping -i 0.2 google.com    # Send every 200ms

# Flood ping (test max throughput, needs root)
sudo ping -f google.com
```

### Reading ping Output

```bash
$ ping google.com
PING google.com (142.250.195.46) 56(84) bytes of data.
64 bytes from bom12s14-in-f14.1e100.net (142.250.195.46): icmp_seq=1 ttl=115 time=6.82 ms
64 bytes from bom12s14-in-f14.1e100.net (142.250.195.46): icmp_seq=2 ttl=115 time=6.45 ms
64 bytes from bom12s14-in-f14.1e100.net (142.250.195.46): icmp_seq=3 ttl=115 time=7.10 ms
^C
--- google.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 6.45/6.79/7.10/0.265 ms
```

**Parse each field:**
- `142.250.195.46` — IP that google.com resolved to
- `icmp_seq=1` — sequence number (look for gaps = lost packets)
- `ttl=115` — Time To Live on the response (128 - 13 hops ≈ 13 hops from Google to you)
- `time=6.82 ms` — round trip time
- `0% packet loss` — all packets returned
- `rtt min/avg/max/mdev` — latency statistics; mdev = deviation (jitter)

### Diagnosing from ping Output

| Result | Diagnosis |
|--------|-----------|
| Works fine | L3 connectivity OK |
| 100% loss | Host unreachable, firewall blocking ICMP, or host down |
| Intermittent loss | Network congestion, bad cable, wireless interference |
| High latency (>200ms domestic) | Congestion, routing issue, VPN overhead |
| High mdev (jitter) | Unstable link, congestion |
| `Destination Host Unreachable` | Router can't reach the host |
| `Request Timeout` | Packet sent, no reply — ICMP blocked or host down |
| `Name or service not known` | DNS resolution failure |

**[INSTRUCTOR: Demo each type of failure if possible]**

### ping Limitations

> "ping tests ICMP reachability. Some servers BLOCK ICMP intentionally (for security / to avoid ping floods). A server that doesn't respond to ping may still serve HTTP. Always try port-based tests too."

---

## SECTION 3: traceroute / tracert (10 minutes)

### What traceroute Does

> "traceroute maps the network path to a destination — showing each router (hop) along the way and the latency to each one."

**Mechanism (Linux `traceroute`):**
1. Send packet with TTL=1 → first router decrements to 0 → sends ICMP Time Exceeded → reveals first hop
2. TTL=2 → second router reveals itself
3. Continue until destination is reached

```bash
# Linux/Mac
traceroute google.com

# Windows
tracert google.com

# Using TCP (avoids ICMP-blocking firewalls)
traceroute -T -p 80 google.com   # TCP traceroute on port 80

# Using UDP (default on Linux)
traceroute google.com   # Linux uses UDP by default

# mtr — continuous traceroute with stats (better tool)
mtr google.com
```

### Reading traceroute Output

```bash
$ traceroute google.com
traceroute to google.com (142.250.195.46), 30 hops max, 60 byte packets
 1  _gateway (192.168.1.1)  1.024 ms  0.893 ms  0.756 ms
 2  broadband.isp.in (10.50.0.1)  8.234 ms  7.956 ms  8.012 ms
 3  * * *
 4  10.60.100.2  15.123 ms  14.987 ms  15.200 ms
 5  72.14.239.45  18.456 ms  18.123 ms  18.234 ms
 6  bom12s14-in-f14.1e100.net (142.250.195.46)  19.876 ms  19.345 ms  19.678 ms
```

**Parse:**
- Hop 1 = your default gateway (192.168.1.1)
- Hop 2 = first ISP router
- Hop 3 = `* * *` → router doesn't respond to ICMP (not necessarily broken — just silently forwards)
- Hops 4-5 = ISP backbone
- Hop 6 = destination reached

**Interpreting latency:**
```
Hop 1: 1ms    (your LAN → router — should be fast)
Hop 2: 8ms    (+7ms — reasonable for ISP hop)
Hop 3: * * *  (ICMP filtered)
Hop 4: 15ms   (backbone router)
Hop 5: 18ms   (international leg — slight jump, normal)
Hop 6: 20ms   (destination — stable)
```

> "Big latency jump at a hop (e.g., hop 4 jumps from 5ms to 200ms) means congestion or a slow physical link at that point."

**[INSTRUCTOR: Run live traceroute and interpret together with class]**

---

## SECTION 4: nslookup and dig (8 minutes)

### nslookup (Basic)

```bash
nslookup google.com             # Basic lookup
nslookup google.com 8.8.8.8     # Use specific DNS server
nslookup -type=MX google.com    # MX records
```

### dig (Advanced)

```bash
# Basic A record lookup
dig google.com

# Specific record type
dig google.com MX
dig google.com AAAA
dig google.com NS
dig google.com TXT

# Use specific DNS server
dig @8.8.8.8 google.com

# Reverse DNS lookup
dig -x 142.250.195.46

# Show only the answer (terse output)
dig +short google.com

# Trace full resolution path (sees every server consulted)
dig +trace google.com

# Check DNSSEC
dig +dnssec google.com
```

### Reading dig Output

```bash
$ dig google.com

; <<>> DiG 9.18 <<>> google.com
;; QUESTION SECTION:
;google.com.                    IN      A

;; ANSWER SECTION:
google.com.             300     IN      A       142.250.195.46

;; Query time: 6 msec
;; SERVER: 192.168.1.1#53(192.168.1.1)
```

**Key fields:**
- `QUESTION SECTION` — what was asked
- `ANSWER SECTION` — the response; `300` = TTL in seconds
- `Query time: 6 msec` — how long DNS took
- `SERVER: 192.168.1.1` — which DNS server answered (your router's forwarder)

### Common DNS Troubleshooting

| Symptom | Diagnosis | Fix |
|---------|-----------|-----|
| `dig` works but browser doesn't | DNS poisoning in browser cache | Clear browser DNS cache |
| `NXDOMAIN` (no such domain) | Domain doesn't exist | Check spelling |
| `SERVFAIL` | DNS server issue | Try different DNS (8.8.8.8) |
| Slow first load | High DNS TTL after change | Lower TTL before change |
| Works with 8.8.8.8 but not ISP DNS | ISP DNS broken | Change DNS to 8.8.8.8 |

---

## SECTION 5: netstat and ss (10 minutes)

### What netstat/ss Does

> "netstat and ss show you all **active and listening network connections** on your machine. Think of it as looking at all the sockets the OS has open."

### ss — Socket Statistics (Modern, Preferred)

```bash
# Show all TCP connections
ss -t

# Show all listening TCP sockets
ss -tl

# Show all UDP
ss -u

# Show ALL sockets with PID and process name
ss -tulpn

# Show connections to/from specific port
ss -t dst :443

# Show established connections only
ss -t state established
```

### Reading ss Output

```bash
$ ss -tulpn
Netid  State    Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
tcp    LISTEN   0       128     0.0.0.0:22           0.0.0.0:*          users:(("sshd"))
tcp    LISTEN   0       10      0.0.0.0:8080         0.0.0.0:*          users:(("python3"))
tcp    ESTAB    0       0       192.168.1.5:52341    142.250.195.46:80  users:(("curl"))
udp    UNCONN   0       0       0.0.0.0:53           0.0.0.0:*          users:(("dnsmasq"))
```

**Parse:**
- `LISTEN 0.0.0.0:22` — SSH server listening on all interfaces, port 22
- `LISTEN 0.0.0.0:8080` — your Python app listening
- `ESTAB 52341 → 80` — active HTTP connection (established = TCP handshake done)
- `UNCONN UDP:53` — DNS server

**[INSTRUCTOR: Run this on the class server — students immediately see their SSH connections in ESTAB state]**

### TCP Connection States

| State | Meaning |
|-------|---------|
| LISTEN | Waiting for incoming connection |
| SYN_SENT | SYN sent, waiting for SYN-ACK |
| SYN_RECV | SYN received, SYN-ACK sent |
| ESTABLISHED | Connection active |
| FIN_WAIT_1 | FIN sent, waiting for ACK |
| FIN_WAIT_2 | Got ACK of FIN, waiting for remote FIN |
| CLOSE_WAIT | Remote sent FIN; waiting for local close |
| TIME_WAIT | Both FINs done; waiting for delayed packets |
| CLOSED | No connection |

**Diagnosing with states:**
- Many `TIME_WAIT` → high connection turnover; consider keep-alive
- Many `CLOSE_WAIT` → application not closing sockets properly (bug!)
- `SYN_RECV` but never `ESTABLISHED` → SYN flood attack or firewall issue

---

## SECTION 6: tcpdump (12 minutes)

### What tcpdump Does

> "tcpdump is a packet capture tool — it shows you RAW packets as they flow in and out of your network interface. This is the most powerful network debugging tool you'll ever use."

### Basic Usage

```bash
# Capture on default interface
sudo tcpdump

# Specify interface
sudo tcpdump -i eth0

# Capture with verbosity (shows more detail)
sudo tcpdump -v
sudo tcpdump -vv

# Don't resolve hostnames (faster)
sudo tcpdump -n

# Save capture to file
sudo tcpdump -w capture.pcap

# Read saved capture
tcpdump -r capture.pcap
```

### Filters — The Most Important Part

```bash
# Filter by host
sudo tcpdump host 8.8.8.8

# Filter by port
sudo tcpdump port 80
sudo tcpdump port 443

# Filter by protocol
sudo tcpdump icmp
sudo tcpdump udp
sudo tcpdump tcp

# Combine filters
sudo tcpdump host 8.8.8.8 and port 80

# Capture DNS only
sudo tcpdump -n port 53

# Capture HTTP (show content)
sudo tcpdump -A -n port 80

# Capture TCP SYN packets only (new connections)
sudo tcpdump 'tcp[tcpflags] & tcp-syn != 0'

# Capture everything EXCEPT SSH (to avoid flooding output)
sudo tcpdump -i eth0 not port 22
```

### Reading tcpdump Output

```bash
$ sudo tcpdump -n host 8.8.8.8 port 53

12:34:56.123456 IP 192.168.1.5.52000 > 8.8.8.8.53: 12345+ A? google.com. (28)
12:34:56.130123 IP 8.8.8.8.53 > 192.168.1.5.52000: 12345 1/0/0 A 142.250.195.46 (44)
```

**Parse:**
- `12:34:56.123456` — timestamp (microsecond precision)
- `IP 192.168.1.5.52000` — source IP.port
- `> 8.8.8.8.53` — destination IP.port
- `12345+ A? google.com.` — DNS query ID=12345, type A, for google.com
- `12345 1/0/0 A 142.250.195.46` — DNS response: 1 answer, 0 authority, 0 additional; A record = 142.250.195.46

### Practical tcpdump Scenarios

```bash
# Is my API receiving requests?
sudo tcpdump -n port 8080

# Is the DB getting connections from the app?
sudo tcpdump -n host 10.0.0.5 port 5432

# Is DNS failing?
sudo tcpdump -n port 53

# TCP handshake debugging (show SYN/SYN-ACK/ACK)
sudo tcpdump -n 'tcp[13] & 2 != 0'   # SYN flag

# Find who's making connections to port 80
sudo tcpdump -n 'tcp[tcpflags] == tcp-syn and port 80'
```

---

## SECTION 7: Wireshark (10 minutes)

> "Wireshark is the GUI version of tcpdump with deep protocol parsing, color coding, and filtering. For complex investigations, it's far more powerful than reading tcpdump output."

**[INSTRUCTOR: Open Wireshark and demonstrate the following]**

### Key Wireshark Features

**Capture Filter (before capture starts):**
```
port 80
host 8.8.8.8
icmp
tcp port 443
```

**Display Filter (after capture, refinement):**
```
ip.addr == 8.8.8.8
tcp.port == 80
dns
http
http.request.method == "GET"
tcp.flags.syn == 1
```

### Reading Wireshark

**Panel layout:**
1. **Packet list** (top): each row = one packet; colored by protocol
2. **Packet details** (middle): hierarchical decode — Ethernet → IP → TCP → HTTP
3. **Packet bytes** (bottom): raw hex

**Follow TCP stream:** Right-click any TCP packet → Follow → TCP Stream  
Shows the entire HTTP conversation as readable text.

**Statistics menu:**
- Protocol Hierarchy: what protocols are in the capture
- IO Graph: bandwidth over time
- TCP Stream Graph: throughput, round trip time, sequence number plot

### Analyzing a Packet Capture

**Scenario: Website loading slowly — is it DNS or server response time?**

```
Filter: dns
→ See DNS query at 0.000s → DNS response at 0.006s (6ms — fine)

Filter: tcp.stream eq 0 (first TCP connection)
→ SYN at 0.007s → SYN-ACK at 0.020s (13ms RTT — fine)
→ First HTTP request at 0.021s → HTTP response at 1.500s (1.479 seconds!)

Diagnosis: Server is slow to respond (TTFB = 1.5s). Not DNS, not network.
```

---

## SECTION 8: Common Networking Problems (8 minutes)

**[INSTRUCTOR: Run through these quickly as case studies]**

### Problem 1: "I can't access the internet"

```
Diagnosis path:
1. ip addr → is interface up? got an IP?
2. ping 192.168.1.1 → can I reach my gateway?
   - Fails → LAN issue, cable, Wi-Fi
3. ping 8.8.8.8 → is ISP routing working?
   - Fails → ISP outage, router WAN issue
4. ping google.com → is DNS working?
   - Fails but 8.8.8.8 works → DNS failure
   - Try: dig @8.8.8.8 google.com
5. Try accessing http://142.250.195.46 → is it just DNS?
```

### Problem 2: "My server is running but nobody can connect"

```
1. ss -tulpn → is the service actually listening on the right port/interface?
   - "0.0.0.0:8080" = all interfaces ✓
   - "127.0.0.1:8080" = localhost only ← common bug!
2. iptables -L → firewall rules blocking?
3. curl localhost:8080 → does it work from server itself?
4. tcpdump -n port 8080 → are connections reaching the server?
   - Packets arriving → service issue
   - No packets → firewall or routing issue
```

### Problem 3: "Intermittent packet loss"

```
1. mtr <destination> → continuous traceroute, shows % loss at each hop
2. ping -c 100 <destination> → get statistics over many packets
3. If loss at hop 3 but NOT hop 4 → hop 3 de-prioritizes ICMP (false positive)
4. If loss at hop 3 AND all subsequent → real loss point identified
5. ethtool eth0 → check interface for errors
```

### Problem 4: "DNS resolves differently from different machines"

```
1. dig google.com from each machine
2. dig @8.8.8.8 google.com → test public resolver
3. Check /etc/resolv.conf → which DNS server is configured?
4. Compare TTLs → stale cache?
5. Flush DNS: sudo systemd-resolve --flush-caches (Linux)
               ipconfig /flushdns (Windows)
```

---

## SUMMARY (5 minutes)

```
✅ Methodology: Physical → Data Link → Network → Transport → Application
✅ ping: ICMP echo; tests L3 reachability; check loss and RTT
✅ traceroute: traces each hop; identify where latency appears
✅ dig/nslookup: DNS lookup tool; +trace for full resolution path
✅ ss -tulpn: all listening/active sockets with PIDs
✅ TCP states: LISTEN, ESTABLISHED, TIME_WAIT, CLOSE_WAIT — each tells a story
✅ tcpdump: raw packet capture; powerful filters; save to .pcap
✅ Wireshark: GUI packet analyzer; follow streams; statistics
✅ Common bugs:
    - Service bound to 127.0.0.1 instead of 0.0.0.0
    - DNS resolution failure
    - Firewall blocking (check iptables)
    - Sockets in CLOSE_WAIT (app bug)
```

---

*End of Lecture 12 Script*
