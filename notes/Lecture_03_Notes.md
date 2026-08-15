# Lecture 3: IP Addresses and Subnetting I
## Student Notes — SST Computer Networks (Term 5)

---

## 🎯 Learning Objectives
- Understand the need for IP addressing
- Read and write IPv4 addresses in binary and dotted-decimal
- Apply subnet masks and CIDR notation
- Identify network, host, network address, broadcast address
- Distinguish public, private, and loopback addresses

---

## 1. Why IP Addresses?

MAC addresses (from Lecture 2) are local — they only work within a single network segment. To route data across the internet (across thousands of networks), we need a global addressing system.

**IP addresses provide:**
- **Global uniqueness** (for public IPs)
- **Hierarchical structure** — the network portion enables efficient routing
- **Logical assignment** — can be changed, unlike hardware MACs

---

## 2. IPv4 Address Format

An IPv4 address is a **32-bit number**, written as four decimal octets (0–255) separated by dots:

```
192   .   168   .    1   .    5
8 bits    8 bits   8 bits   8 bits  = 32 bits total
```

**Total IPv4 address space:** 2³² = 4,294,967,296 ≈ 4.3 billion addresses

---

## 3. Binary ↔ Decimal Conversion

### Bit Position Values (for one 8-bit octet)
```
Position:  7    6    5    4    3    2    1    0
Value:    128   64   32   16    8    4    2    1
```

### Decimal → Binary
Add bit values from left to right until you reach the target number.

**Example: 192**
```
128 ≤ 192 → bit7 = 1, remainder = 64
 64 ≤ 64  → bit6 = 1, remainder = 0
All others = 0

192 = 1 1 0 0 0 0 0 0 = 11000000
```

**Example: 168**
```
128 → 1, rem=40 | 64 → 0 | 32 → 1, rem=8 | 16 → 0 | 8 → 1, rem=0 | 4,2,1 → 0
168 = 10101000
```

### Binary → Decimal
Sum the values of all positions with a 1.

**Example: 11000000** = 128 + 64 = **192**  
**Example: 10101000** = 128 + 32 + 8 = **168**

### Practice Table

| Decimal | Binary |
|---------|--------|
| 0 | 00000000 |
| 1 | 00000001 |
| 10 | 00001010 |
| 128 | 10000000 |
| 192 | 11000000 |
| 255 | 11111111 |

### Full IP in Binary

```
192.168.1.5:
  192 → 11000000
  168 → 10101000
    1 → 00000001
    5 → 00000101

32-bit: 11000000.10101000.00000001.00000101
```

---

## 4. Network ID and Host ID

Every IP address has two parts:

```
IP Address:  192  .  168  .   1  .   5
             └─── Network Portion ──┘ └─ Host Portion ─┘
```

- **Network Portion:** Identifies which network the device is on (used by routers)
- **Host Portion:** Identifies the specific device within that network

The dividing line is determined by the **subnet mask**.

### Traditional Address Classes (Historical Reference)

| Class | First Octet | Network Bits | Host Bits | Default Mask |
|-------|-------------|-------------|----------|-------------|
| A | 1–126 | 8 | 24 | 255.0.0.0 |
| B | 128–191 | 16 | 16 | 255.255.0.0 |
| C | 192–223 | 24 | 8 | 255.255.255.0 |

> Note: Classful addressing is largely replaced by CIDR, but the ranges are still referenced.

---

## 5. Subnet Mask

A **subnet mask** is a 32-bit number where:
- All **1s** mark the **network** bits
- All **0s** mark the **host** bits
- The 1s are always contiguous (come first)

**Example — 255.255.255.0**
```
Binary: 11111111.11111111.11111111.00000000
        └──── 24 network bits ────┘└─ 8 host bits ─┘
```

### Finding the Network Address

**Network Address = IP address AND Subnet Mask**

The AND operation: `1 AND 1 = 1`, `1 AND 0 = 0`, `0 AND 0 = 0`

```
IP:   192.168.1.5    → 11000000.10101000.00000001.00000101
Mask: 255.255.255.0  → 11111111.11111111.11111111.00000000
AND:                   11000000.10101000.00000001.00000000 = 192.168.1.0
```

**Result:** Network address = `192.168.1.0`

### Special Addresses in a Subnet

For any subnet `X.X.X.X/N`:

| Address | Host bits | Purpose |
|---------|-----------|---------|
| **Network Address** | All 0s | Identifies the subnet (not assignable to a device) |
| **First Host** | 0...01 | First usable device address |
| **Last Host** | 1...10 | Last usable device address |
| **Broadcast Address** | All 1s | Sends to ALL devices in subnet (not assignable) |

**Example — 192.168.1.0/24:**
```
Network Address:   192.168.1.0
First Host:        192.168.1.1
Last Host:         192.168.1.254
Broadcast:         192.168.1.255
Usable Hosts:      254
```

**Formulas:**
```
Total addresses = 2^(32 - N)      where N = prefix length
Usable hosts    = 2^(32 - N) - 2
```

---

## 6. CIDR Notation

**CIDR (Classless Inter-Domain Routing)** uses a `/N` prefix to indicate the number of network bits, replacing the need to write the full subnet mask.

**Format:** `IP_Address / Prefix_Length`

**Examples:**
```
192.168.1.5/24    → mask = 255.255.255.0   (24 ones)
10.0.0.0/8        → mask = 255.0.0.0       (8 ones)
172.16.0.0/16     → mask = 255.255.0.0     (16 ones)
192.168.1.64/26   → mask = 255.255.255.192 (26 ones)
```

### Reading /26 in detail

```
/26 mask: 11111111.11111111.11111111.11000000 = 255.255.255.192

For 192.168.1.64/26:
- Network:   192.168.1.64
- Hosts:     192.168.1.65 to 192.168.1.126
- Broadcast: 192.168.1.127
- Usable:    2^6 - 2 = 62 hosts
```

### Common CIDR Reference

| CIDR | Subnet Mask | Usable Hosts | Common Use |
|------|-------------|-------------|-----------|
| /8 | 255.0.0.0 | 16,777,214 | Large ISPs |
| /16 | 255.255.0.0 | 65,534 | Large enterprises |
| /24 | 255.255.255.0 | 254 | Home, office networks |
| /25 | 255.255.255.128 | 126 | Medium subnets |
| /26 | 255.255.255.192 | 62 | Small subnets |
| /27 | 255.255.255.224 | 30 | Small offices |
| /28 | 255.255.255.240 | 14 | Very small subnets |
| /30 | 255.255.255.252 | 2 | Point-to-point links |
| /32 | 255.255.255.255 | 0 (single host) | Host routes |

---

## 7. Public vs. Private IP Addresses

### Private IP Ranges

These addresses are NOT routable on the public internet. They can be freely reused inside any private network:

| Range | CIDR | Size | Common Use |
|-------|------|------|-----------|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | 16.7M addresses | Large corps, cloud VPCs |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | 1.04M addresses | Medium networks |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | 65,536 addresses | Home routers, offices |

### Public IP Addresses

All addresses NOT in private ranges (and not in other reserved ranges) are **public** — globally routable on the internet. These are assigned by ISPs and registries (IANA, APNIC, etc.).

### Why Private Addresses Exist

IPv4 has only ~4.3 billion addresses — not enough for every device on earth. Private addressing allows millions of networks to reuse the same address space internally. Only the router/gateway needs a public IP. Translation between private and public is handled by **NAT** (Lecture 11).

---

## 8. Special Addresses

### Loopback — 127.0.0.1 (127.0.0.0/8)

The **loopback address** routes packets back to the same device without sending them over any physical network interface.

```bash
ping 127.0.0.1        # Always works if your network stack is functional
# "localhost" resolves to 127.0.0.1
# Used for local server testing: http://localhost:3000
```

### Other Reserved Ranges

| Address / Range | Purpose |
|----------------|---------|
| 0.0.0.0 | Unspecified / "any" address |
| 127.0.0.0/8 | Loopback (entire /8 reserved) |
| 169.254.0.0/16 | APIPA — automatic self-assigned when DHCP fails |
| 255.255.255.255 | Limited broadcast (all hosts on local subnet) |

> **Tip:** If your device has a `169.254.x.x` address, it means DHCP didn't work — your device couldn't get an IP from the router.

---

## 9. Worked Examples

### Example 1: Analyze 192.168.10.130/25
```
/25 mask: 11111111.11111111.11111111.10000000 = 255.255.255.128

IP in binary: ...10000010 (last octet = 130)
Network bits (25): 192.168.10.1xxxxxxx → 192.168.10.128

Network Address:   192.168.10.128
Broadcast:         192.168.10.255
Hosts:             192.168.10.129 to 192.168.10.254
Usable count:      2^7 - 2 = 126
```

### Example 2: Is 172.20.5.1 private?
```
172.20.x.x falls in range 172.16.0.0 – 172.31.255.255
→ YES, it is private (172.16.0.0/12)
```

### Example 3: Network address of 10.45.60.100/20
```
/20 mask: 11111111.11111111.11110000.00000000 = 255.255.240.0

Third octet of 60: 00111100
Keep first 4 bits: 0011 → then zero rest: 00110000 = 48

Network Address: 10.45.48.0
Broadcast:       10.45.63.255
Usable hosts:    2^12 - 2 = 4094
```

---

## 📌 Key Takeaways

1. IPv4 = 32-bit address in 4 octets (0–255 each)
2. Subnet mask divides address into network (1s) and host (0s) parts
3. **Network Address** = IP AND mask (all host bits = 0)
4. **Broadcast** = all host bits set to 1
5. **Usable hosts** = 2^(host bits) − 2
6. **CIDR /N** = N network bits; total addresses = 2^(32−N)
7. Private ranges: 10.x, 172.16–31.x, 192.168.x — not internet-routable
8. **Loopback** 127.0.0.1 = always local, never leaves the machine

---

## 🧠 Quick Self-Check Questions

1. Convert 172 to binary.
2. What is the network address of `10.20.30.40/16`?
3. How many usable hosts in a `/28` subnet?
4. Is `172.25.10.5` a public or private IP? Which range?
5. What does APIPA (169.254.x.x) mean when you see it on a device?
6. Write `192.168.1.5/24` fully in 32-bit binary.
7. What is the broadcast address of `172.16.0.0/12`?

---

*Lecture 3 of 13 — Computer Networks, Term 5, SST*
