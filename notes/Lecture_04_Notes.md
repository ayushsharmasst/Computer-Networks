# Lecture 4: IP Addresses and Subnetting II
## Student Notes — SST Computer Networks (Term 5)

---

## 🎯 Learning Objectives
- Solve subnetting problems from host requirements
- Apply VLSM to efficiently allocate IP space
- Understand IPv6 format and key differences from IPv4
- Explain default gateway and how a device decides to route locally vs. via gateway
- Describe NAT and why it exists

---

## 1. Subnetting — From Requirements to Subnet

### Step-by-Step Process

1. **Determine** how many hosts you need
2. **Find** the smallest 2^N ≥ (hosts + 2)
3. **Host bits** = N → **Prefix** = 32 − N
4. **Subnet size** (block) = 2^N
5. **Usable hosts** = 2^N − 2

### Quick Reference: Host Count → Prefix

| Hosts Needed | Min 2^N | Prefix | Usable Hosts |
|-------------|---------|--------|-------------|
| 2 | 4 | /30 | 2 |
| 3–6 | 8 | /29 | 6 |
| 7–14 | 16 | /28 | 14 |
| 15–30 | 32 | /27 | 30 |
| 31–62 | 64 | /26 | 62 |
| 63–126 | 128 | /25 | 126 |
| 127–254 | 256 | /24 | 254 |

### Example 1: Need 50 hosts per subnet within 10.0.0.0/24

```
50 + 2 = 52 → next power of 2 = 64 = 2^6
Host bits = 6 → Prefix = /26
Block size = 64

Subnets:
10.0.0.0/26   → Hosts: 10.0.0.1 – 10.0.0.62    Broadcast: 10.0.0.63
10.0.0.64/26  → Hosts: 10.0.0.65 – 10.0.0.126   Broadcast: 10.0.0.127
10.0.0.128/26 → Hosts: 10.0.0.129 – 10.0.0.190  Broadcast: 10.0.0.191
10.0.0.192/26 → Hosts: 10.0.0.193 – 10.0.0.254  Broadcast: 10.0.0.255
```

### Example 2: How many /28 subnets in a /24?

```
Number of subnets = 2^(new prefix − original prefix) = 2^(28−24) = 16 subnets
Hosts per /28 = 2^4 − 2 = 14 usable
```

---

## 2. Variable Length Subnet Masking (VLSM)

**VLSM** allows different subnets to have different prefix lengths, enabling efficient use of IP space.

**Golden rule:** Allocate the **largest subnet first**, then work downward.

### Example: Allocate 192.168.10.0/24 for 4 departments

| Dept | Hosts Needed | Prefix | Block Size |
|------|-------------|--------|-----------|
| Engineering | 60 | /26 | 64 |
| Marketing | 30 | /27 | 32 |
| HR | 10 | /28 | 16 |
| Management | 5 | /29 | 8 |

**Allocation (largest first):**

```
Engineering (60): 192.168.10.0/26   → .0   to .63   (62 usable)
Marketing  (30): 192.168.10.64/27  → .64  to .95   (30 usable)
HR         (10): 192.168.10.96/28  → .96  to .111  (14 usable)
Management  (5): 192.168.10.112/29 → .112 to .119  (6 usable)
Remaining:       192.168.10.120 to 192.168.10.255 (unused)
```

---

## 3. IPv6

### Why IPv6?
- IPv4 exhausted: IANA ran out of free blocks in 2011
- IPv6 provides 2¹²⁸ ≈ **3.4 × 10³⁸** addresses — essentially unlimited

### IPv6 Format
- **128 bits**, written as 8 groups of 4 hex digits separated by colons
- Example: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`

### Compression Rules

**Rule 1:** Omit leading zeros in each group
```
0db8 → db8
0000 → 0
```

**Rule 2:** Replace one sequence of consecutive all-zero groups with `::`
```
2001:0db8:0000:0000:0000:8a2e:0370:7334
→ 2001:db8::8a2e:370:7334
```
> ⚠️ `::` can only be used **once** per address (otherwise ambiguous)

### Common IPv6 Addresses

| Address | Meaning |
|---------|---------|
| `::1` | Loopback (like 127.0.0.1) |
| `::` | Unspecified / any |
| `fe80::/10` | Link-local (auto-configured, not globally routed) |
| `ff00::/8` | Multicast |
| `2000::/3` | Global unicast (public internet) |

### IPv4 vs IPv6 Comparison

| Feature | IPv4 | IPv6 |
|---------|------|------|
| Size | 32 bits | 128 bits |
| Format | Dotted decimal | Colon-hex |
| Total addresses | ~4.3 billion | 3.4 × 10³⁸ |
| Header | Variable (20–60 bytes) | Fixed 40 bytes |
| Broadcast | Yes | No (uses multicast) |
| NAT | Necessary | Not needed |
| ARP | Yes | Replaced by NDP |
| Fragmentation | Routers + hosts | Source host only |

---

## 4. Default Gateway

The **default gateway** is the router's IP address on your local network — the device that forwards traffic to networks outside your subnet.

### How a Device Decides: Direct or Gateway?

When sending a packet to IP `X`:
```
1. Compute: X AND my_subnet_mask
2. Does result equal my network address?
   YES → X is on my subnet → send directly (ARP for MAC)
   NO  → X is on a different network → send to default gateway
```

**Example:**
```
My IP:             192.168.1.5
Subnet mask:       255.255.255.0 (/24)
My network:        192.168.1.0
Default gateway:   192.168.1.1

Destination: 192.168.1.20
  192.168.1.20 AND 255.255.255.0 = 192.168.1.0 ✓ Same network → send direct

Destination: 8.8.8.8
  8.8.8.8 AND 255.255.255.0 = 8.8.8.0 ✗ Different network → send to 192.168.1.1
```

### Checking Gateway on Your Machine

```bash
# Linux
ip route show
# Output: "default via 192.168.1.1 dev eth0"

# Windows
route print
# or ipconfig → "Default Gateway"

# macOS
netstat -rn | grep default
```

---

## 5. Broadcast and Network Addresses — Summary

For any subnet `A.B.C.D/N`:

| Address | Calculation | Example (/24) |
|---------|-------------|--------------|
| **Network** | All host bits = 0 | 192.168.1.0 |
| **First Host** | Network + 1 | 192.168.1.1 |
| **Last Host** | Broadcast - 1 | 192.168.1.254 |
| **Broadcast** | All host bits = 1 | 192.168.1.255 |

### Types of Broadcast

| Type | Address | Scope |
|------|---------|-------|
| **Limited Broadcast** | 255.255.255.255 | Local subnet only; never forwarded |
| **Directed Broadcast** | All host bits = 1 (e.g., 192.168.1.255) | A specific subnet's broadcast |

---

## 6. NAT — Network Address Translation

### The Problem

Your ISP gives you **one public IP**. You have **many devices**. NAT lets all devices share that one IP.

### How NAT Works (PAT / Masquerade)

The router maintains a **NAT translation table** mapping:

`(private IP : private port) ↔ (public IP : translated port)`

**Outgoing packet:**
```
Laptop:  192.168.1.5:52341  →  Google: 142.250.195.46:80
Router translates source:
         203.0.113.5:10001  →  Google: 142.250.195.46:80
Table: port 10001 ↔ 192.168.1.5:52341
```

**Incoming reply:**
```
Google:  142.250.195.46:80  →  203.0.113.5:10001
Router looks up table: 10001 → 192.168.1.5:52341
Forwards:  142.250.195.46:80 → 192.168.1.5:52341
```

### NAT Variants

| Type | Direction | Use Case |
|------|-----------|---------|
| **SNAT** (Source NAT) | Outgoing | Home internet sharing |
| **DNAT** (Destination NAT) | Incoming | Port forwarding, load balancing |
| **PAT** (Port Address Translation) | Both | Many devices share one IP |

### Port Forwarding (DNAT Example)

```
Scenario: Game server at 192.168.1.10:27015

Router rule: external port 27015 → 192.168.1.10:27015

Friend connects to: 203.0.113.5:27015 (your public IP)
Router translates: → 192.168.1.10:27015 internally
```

### NAT Limitations

1. **Breaks end-to-end connectivity** — devices behind NAT can't receive unsolicited inbound connections without port forwarding
2. **Stateful** — router must track all active connections; reboot breaks sessions
3. **Complicates some protocols** — FTP, VoIP/SIP, gaming embed IP addresses in payload

---

## 7. Worked Examples

### Full Subnetting Example

**Given:** `172.16.0.0/16`. Need 500 subnets. How many hosts per subnet?

```
Available bits: 32 - 16 = 16 host bits
Need 500 subnets → 2^N ≥ 500 → 2^9 = 512, so N = 9 bits for subnets
Remaining host bits: 16 - 9 = 7 → 2^7 - 2 = 126 hosts per subnet
New prefix: /16 + 9 = /25
```

### IPv6 Compression

```
2001:0000:0000:0db8:0000:0000:0000:0001
= 2001:0:0:db8::1
(first two zero groups NOT merged with the three; use :: for the longer run)
```

---

## 📌 Key Takeaways

1. **Subnetting process**: requirements → 2^N ≥ hosts+2 → prefix = 32−N
2. **VLSM**: allocate largest subnet first; no overlap
3. **IPv6**: 128-bit, colon-hex, `::` for zero compression (once only)
4. **Default gateway**: router's local IP — used when destination is off-subnet
5. **NAT**: maps `(private IP:port) → (public IP:port)` — enables IP sharing
6. **Port forwarding**: DNAT — incoming public port maps to private device

---

## 🧠 Quick Self-Check Questions

1. You need 100 hosts per subnet. What prefix length should you use?
2. How many subnets can you create from `10.0.0.0/16` if each subnet is `/20`?
3. Compress `2001:0000:0000:0000:0001:0000:0000:0001` to its shortest form.
4. Your IP is `10.1.1.50/24` and you're sending to `10.1.2.5`. Does it go to the gateway?
5. In NAT, who initiates the connection tracking — the router or the device?
6. What problem does NAT solve? What problem does it create?
7. A company gets a /22 block. How many usable host addresses total?

---

*Lecture 4 of 13 — Computer Networks, Term 5, SST*
