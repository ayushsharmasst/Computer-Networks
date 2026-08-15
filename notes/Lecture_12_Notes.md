# Lecture 12: Network Troubleshooting
## Student Notes — SST Computer Networks (Term 5)

---

## 🎯 Learning Objectives
- Apply a systematic troubleshooting methodology
- Use ping, traceroute, dig/nslookup, netstat/ss, tcpdump, and Wireshark
- Interpret tool outputs to diagnose connectivity issues
- Recognize and fix common networking problems

---

## 1. Troubleshooting Methodology

Work bottom-up through the OSI layers:

```
Layer 1 – Physical:     Cable plugged in? Wi-Fi associated? LEDs on?
Layer 2 – Data Link:    Interface up? MAC conflict? Switch port active?
Layer 3 – Network:      IP configured? Can ping gateway? Can ping remote IP?
Layer 4 – Transport:    Correct port open? Firewall blocking? Listening?
Layer 7 – Application:  Service running? Correct config? Logs show errors?
```

**Quick checklist:**
1. `ip addr` — interface up, has IP?
2. `ping <gateway>` — local network OK?
3. `ping 8.8.8.8` — internet routing OK?
4. `ping <hostname>` — DNS OK?
5. `ss -tulpn | grep <port>` — service listening?
6. `curl http://localhost:<port>` — service responding?

---

## 2. ping

`ping` sends **ICMP Echo Request** packets and measures round-trip time (RTT).

### Common Options

```bash
# Linux/macOS
ping google.com               # Continuous (Ctrl+C to stop)
ping -c 4 google.com          # Send exactly 4 packets
ping -i 0.5 google.com        # Send every 0.5 seconds
ping -s 1000 google.com       # 1000-byte packets (test MTU)

# Windows
ping google.com               # 4 packets by default
ping -t google.com            # Continuous
ping -n 10 google.com         # 10 packets
```

### Reading Output

```
64 bytes from 142.250.195.46: icmp_seq=1 ttl=115 time=6.82 ms
```

| Field | Meaning |
|-------|---------|
| `64 bytes` | Response size |
| `icmp_seq=1` | Sequence number (gaps = lost packets) |
| `ttl=115` | TTL of reply; estimate hops = (start TTL - received TTL) |
| `time=6.82 ms` | Round-trip time |

```
3 packets transmitted, 3 received, 0% packet loss
rtt min/avg/max/mdev = 6.45/6.79/7.10/0.265 ms
```

| Stat | Meaning |
|------|---------|
| `packet loss` | % of packets that didn't return |
| `min` | Best latency |
| `avg` | Average latency |
| `max` | Worst latency |
| `mdev` | Jitter (variation) — high mdev = unstable link |

### ping Diagnostics

| Result | Likely Issue |
|--------|------------|
| Success | L3 connectivity OK |
| 100% packet loss | Host unreachable, ICMP blocked, or host down |
| Intermittent loss | Congestion, bad cable, Wi-Fi interference |
| High latency | Congestion, long physical path, VPN |
| High mdev | Unstable or congested link |
| `Name not known` | DNS resolution failed |
| `Destination Unreachable` | Router has no route to host |

> **Note:** Some servers block ICMP intentionally. A ping failure doesn't always mean the host is down — try connecting to a port next.

---

## 3. traceroute / tracert

Reveals the path packets take and latency at each hop.

### How It Works

Sends packets with increasing TTL (1, 2, 3…). Each router that drops a TTL=0 packet sends back ICMP Time Exceeded, revealing its IP and latency.

### Commands

```bash
# Linux/macOS
traceroute google.com
traceroute -T -p 80 google.com    # TCP on port 80 (avoids ICMP blocks)
mtr google.com                     # Better: continuous + stats per hop

# Windows
tracert google.com
```

### Reading Output

```
 1  192.168.1.1       1.0 ms    ← your gateway
 2  10.50.0.1         8.2 ms    ← ISP first hop
 3  * * *                       ← ICMP filtered (hop still forwards traffic)
 4  10.60.100.2      15.1 ms
 5  72.14.239.45     18.4 ms    ← Google peering
 6  142.250.195.46   19.9 ms    ← destination reached
```

### Interpreting Latency Jumps

```
Hop 1:   1ms
Hop 2:   8ms   (+7ms — one ISP hop, normal)
Hop 3:   * * * (filtered but traffic passes)
Hop 4: 200ms   (+192ms — congestion or slow link HERE)
Hop 5: 202ms   (+2ms — after the slow link, normal again)
```

> Large jump at a hop = congestion or slow physical link at that point.

> `* * *` doesn't mean the path is broken — many routers silently forward without responding to ICMP probes.

---

## 4. nslookup and dig

Tools for querying DNS servers directly.

### nslookup

```bash
nslookup google.com                  # Default DNS
nslookup google.com 8.8.8.8          # Specific DNS server
nslookup -type=MX google.com         # Mail records
nslookup -type=NS google.com         # Nameservers
```

### dig (Preferred — More Detail)

```bash
dig google.com                       # A record (IPv4)
dig google.com AAAA                  # IPv6 record
dig google.com MX                    # Mail server
dig google.com NS                    # Nameservers
dig google.com TXT                   # Text records

dig @8.8.8.8 google.com             # Use specific resolver
dig +short google.com               # Just the IP
dig +trace google.com               # Full resolution trace
dig -x 142.250.195.46               # Reverse lookup (IP → name)
```

### Reading dig Output

```
;; QUESTION SECTION:
;google.com.            IN      A          ← What was asked

;; ANSWER SECTION:
google.com.     300     IN      A       142.250.195.46
                ↑TTL         ↑type   ↑answer

;; Query time: 6 msec
;; SERVER: 192.168.1.1#53                  ← which DNS server responded
```

### DNS Troubleshooting

| Symptom | Diagnosis |
|---------|-----------|
| `NXDOMAIN` | Domain doesn't exist; check spelling |
| `SERVFAIL` | DNS server internal error; try alternate resolver |
| Works with `8.8.8.8` but not ISP DNS | ISP DNS broken → change DNS config |
| Wrong IP returned | DNS poisoning, stale cache |
| Very slow first query | High TTL missed; cold cache |

---

## 5. netstat and ss

Show all active network connections and listening sockets.

### ss (Modern — Preferred)

```bash
ss -t          # All TCP connections
ss -tl         # Listening TCP sockets only
ss -u          # UDP sockets
ss -tulpn      # TCP+UDP, listening+established, with PIDs, numeric

ss -t state established          # Only established connections
ss -t dst :443                   # Connections to port 443
ss -t src 192.168.1.5            # Connections from specific IP
```

### Reading ss Output

```
Netid State   Recv-Q Send-Q   Local Address:Port   Peer Address:Port  Process
tcp   LISTEN  0      128      0.0.0.0:22            0.0.0.0:*          sshd
tcp   LISTEN  0      10       0.0.0.0:8080          0.0.0.0:*          python3
tcp   ESTAB   0      0        192.168.1.5:52341     142.250.195.46:80  curl
udp   UNCONN  0      0        0.0.0.0:53            0.0.0.0:*          dnsmasq
```

**Key fields:**
- `Local Address`: `0.0.0.0:port` = all interfaces; `127.0.0.1:port` = localhost only ⚠️
- `LISTEN`: waiting for connections
- `ESTAB`: active TCP connection
- `Recv-Q > 0`: data received but not read by application (app slow or stuck)

### TCP States Reference

| State | Meaning |
|-------|---------|
| `LISTEN` | Server waiting for incoming connection |
| `SYN_SENT` | Client sent SYN, waiting for SYN-ACK |
| `ESTABLISHED` | Active connection |
| `CLOSE_WAIT` | Remote closed; local app hasn't called close() yet |
| `TIME_WAIT` | Graceful close done; waiting for late packets |
| `FIN_WAIT_1/2` | Local side initiated close |

**Common bugs revealed by states:**
- Many `CLOSE_WAIT` → **application bug**: not closing sockets after use
- Many `TIME_WAIT` → high connection churn; use Keep-Alive
- Stuck in `SYN_SENT` → destination unreachable or firewall dropping SYNs

---

## 6. tcpdump

Raw packet capture — the most powerful network debugging tool.

### Basic Usage

```bash
sudo tcpdump                         # All traffic on default interface
sudo tcpdump -i eth0                 # Specific interface
sudo tcpdump -n                      # Don't resolve hostnames (faster)
sudo tcpdump -w capture.pcap         # Save to file
tcpdump -r capture.pcap             # Read saved file
sudo tcpdump -vv                     # Very verbose (more decode)
sudo tcpdump -A -n port 80          # Show ASCII content (for HTTP)
```

### Filters

```bash
# By host
sudo tcpdump host 8.8.8.8

# By port
sudo tcpdump port 443
sudo tcpdump not port 22            # Exclude SSH

# By protocol
sudo tcpdump icmp
sudo tcpdump udp
sudo tcpdump tcp

# Combined
sudo tcpdump host 8.8.8.8 and port 80
sudo tcpdump 'host 8.8.8.8 or host 1.1.1.1'

# TCP flags (SYN = new connections)
sudo tcpdump 'tcp[tcpflags] & tcp-syn != 0'
```

### Reading tcpdump Output

```
12:34:56.123 IP 192.168.1.5.52341 > 142.250.195.46.80: Flags [S], seq 1001, ...
                └──src ip.port──┘   └──dst ip.port──┘  └flag┘
```

**TCP Flags:**
- `[S]` = SYN (connection initiation)
- `[S.]` = SYN-ACK
- `[.]` = ACK
- `[P.]` = PSH-ACK (data push)
- `[F.]` = FIN-ACK (graceful close)
- `[R]` = RST (connection reset/abort)

### Practical tcpdump Scenarios

```bash
# Is my web server receiving connections?
sudo tcpdump -n port 8080

# Is my app connecting to the database?
sudo tcpdump -n host 10.0.0.5 port 5432

# Is DNS working?
sudo tcpdump -n port 53

# Watch the TCP handshake
sudo tcpdump -n 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0' port 80
```

---

## 7. Wireshark

GUI packet analyzer — deep protocol parsing, easy filtering, statistics.

### Key Features

| Feature | How |
|---------|-----|
| **Capture filter** | Set before capturing (like tcpdump syntax) |
| **Display filter** | Applied post-capture to narrow view |
| **Follow Stream** | Right-click → Follow → TCP Stream → see full conversation |
| **Protocol hierarchy** | Statistics → Protocol Hierarchy |
| **IO Graphs** | Statistics → IO Graph → bandwidth over time |

### Useful Display Filters

```
ip.addr == 8.8.8.8          Filter by IP
tcp.port == 80               Filter by port
dns                          Show DNS packets only
http                         Show HTTP packets
http.request.method == "GET" Only GET requests
tcp.flags.syn == 1           Only SYN packets (new connections)
tcp.analysis.retransmission  Show retransmissions (packet loss)
```

---

## 8. Common Problems and Solutions

| Problem | Tools | Likely Cause | Fix |
|---------|-------|-------------|-----|
| Can't reach internet | ping 8.8.8.8 | ISP outage / router WAN | Check router, call ISP |
| DNS not working | dig, nslookup | ISP DNS down | Change to 8.8.8.8 |
| Port not reachable | ss -tulpn | Service not listening on 0.0.0.0 | Fix bind address in config |
| Intermittent connection drops | mtr, ping -c 100 | Packet loss at a hop | Contact ISP; check cables |
| Slow website | dig +trace, Wireshark | High TTFB (server slow) | Check server logs, DB |
| High `CLOSE_WAIT` count | ss | App not closing sockets | Fix socket.close() in code |
| Service works locally, not externally | iptables -L, ss | Firewall or bind to 127.0.0.1 | Fix firewall / bind address |

---

## 📌 Key Takeaways

1. **Methodology**: start at physical layer, work up; don't guess, use tools
2. **ping**: ICMP echo test; diagnoses L3 reachability, loss, and latency
3. **traceroute**: reveals hop-by-hop path; identifies where latency appears
4. **dig**: DNS lookup with detail; `+trace` shows full resolution chain
5. **ss -tulpn**: shows all sockets, listening ports, and processes
6. **TCP states**: CLOSE_WAIT = app bug; many TIME_WAIT = high churn
7. **tcpdump**: raw packet capture with filters; save to .pcap for Wireshark
8. **Wireshark**: GUI analysis; Follow TCP Stream; protocol statistics

---

## 🧠 Quick Self-Check Questions

1. You `ping 8.8.8.8` and it works, but `ping google.com` fails. What is the most likely cause?
2. A `traceroute` shows `* * *` at hop 4 but hop 5 responds normally. Is hop 4 down?
3. Your server is running but nobody can connect externally. You run `ss -tulpn` and see `127.0.0.1:8080`. What's wrong?
4. What does `Recv-Q: 500` in `ss` output indicate?
5. Write a `tcpdump` command to capture only DNS traffic, without resolving hostnames, saving to a file.
6. In Wireshark you see many red packets labeled `[TCP Retransmission]`. What does this indicate?
7. How would you determine if packet loss is at hop 5 or merely reported by hop 5's ICMP rate limiting?
8. Your app shows many `CLOSE_WAIT` connections. What is the likely bug and how do you fix it?

---

*Lecture 12 of 13 — Computer Networks, Term 5, SST*
