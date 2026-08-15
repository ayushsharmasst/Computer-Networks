# Lecture 3: IP Addresses and Subnetting I
## Instructor Script — SST Computer Networks (Term 5)

---

### 🎯 Learning Objectives
By the end of this session, students will:
- Understand why IP addresses are needed
- Convert between binary and decimal (IPv4)
- Interpret a subnet mask and CIDR notation
- Distinguish public, private, and loopback addresses

**Duration:** ~90 minutes  
**Teaching Style:** Problem-first, lots of binary practice

---

## 📦 OPENING HOOK (5 minutes)

**[INSTRUCTOR: Start with this question on the board:]**

> "Amazon has a billion customers. Google has billions of users. How does a data center in Mumbai know where to send a response back — when millions of people are all asking at the same time?"

Take answers. Then say:

> "Every device connected to the internet has an **IP address**. Think of it like the complete mailing address of your house — country, city, street, house number. Without it, nobody can deliver a letter to you, and you can't address a letter to anyone else. Today we're going to learn exactly how these addresses work."

---

## SECTION 1: Why IP Addresses? (8 minutes)

**Recap from Lecture 2:**
> "Last time we saw that Ethernet frames use MAC addresses. But MAC addresses only work within a local network. They're like knowing someone's desk number in a building — useless if you're trying to mail them something from another city."

**The Need for a Universal Addressing System:**

Write on board:
```
Your Home Network       →     Internet     →    Google's Data Center
(192.168.1.0/24)               (routing)         (142.250.x.x)
```

> "IP addresses solve three problems:
> 1. They're globally meaningful — every IP on the public internet is unique
> 2. They're hierarchical — just like postal codes, they embed location info that helps with routing
> 3. They're assigned logically — unlike MAC addresses, you can change your IP"

**Real example:**
- Your laptop's IP at home: `192.168.1.5` (private, only meaningful locally)
- Google's web server: `142.250.195.46` (public, globally routable)
- Your phone when it goes cellular: `45.123.67.89` (public, assigned by Jio/Airtel)

---

## SECTION 2: IPv4 Addresses (10 minutes)

**Format:**

> "An IPv4 address is a **32-bit number**. We write it as four decimal numbers separated by dots — called **dotted-decimal notation**."

Write:
```
192  .  168  .   1  .   5
 8 bits  8 bits  8 bits  8 bits  = 32 bits total
```

Each section (called an **octet**) can range from 0 to 255.

**Total possible IPv4 addresses:**
> "2^32 = 4,294,967,296 ≈ 4.3 billion. That sounds like a lot, but the world has 8 billion people and everyone has multiple devices. We're actually running out — that's why IPv6 exists. But IPv4 is still dominant, so we need to understand it deeply."

**[INSTRUCTOR: Ask the class — "What's the maximum value one octet can hold?"  
Expected: 255 (which is 11111111 in binary = 2^8 - 1)]**

---

## SECTION 3: Binary Refresher (15 minutes)

**[INSTRUCTOR: This section is critical. Many subnetting errors come from weak binary skills. Do this slowly and let students do the practice at their seats.]**

### Decimal to Binary

> "An octet is 8 bits. Think of 8 on/off switches. Each switch represents a power of 2."

Write the bit weights on board:
```
Bit position:   7    6    5    4    3    2    1    0
Value (2^n):   128   64   32   16    8    4    2    1
```

**Example — Convert 192 to binary:**
```
192 ÷ 2:  Is 128 ≤ 192? YES → bit 7 = 1, remainder = 192-128 = 64
           Is 64 ≤ 64?  YES → bit 6 = 1, remainder = 64-64 = 0
           All remaining bits = 0

192 = 11000000
```

**Example — Convert 168 to binary:**
```
128 → YES → 1, remainder 40
64  → NO  → 0
32  → YES → 1, remainder 8
16  → NO  → 0
8   → YES → 1, remainder 0
4   → 0, 2 → 0, 1 → 0

168 = 10101000
```

**Class practice (2 minutes at seats):**
Convert these to binary:
- 10 → ?  (Answer: 00001010)
- 255 → ? (Answer: 11111111)
- 1 → ?   (Answer: 00000001)

### Binary to Decimal

**Example — Convert 11000000 to decimal:**
```
1×128 + 1×64 + 0×32 + 0×16 + 0×8 + 0×4 + 0×2 + 0×1
= 128 + 64 = 192
```

**Class practice:**
- 00001010 → ? (Answer: 10)
- 11111111 → ? (Answer: 255)
- 10000001 → ? (Answer: 129)

### Full IP Address in Binary

> "Let's write 192.168.1.5 fully in binary:"

```
192  →  11000000
168  →  10101000
  1  →  00000001
  5  →  00000101

Full 32-bit:  11000000.10101000.00000001.00000101
```

**[INSTRUCTOR: Point out — 'This is the actual representation that routers work with. Everything else — CIDR, subnetting — is just manipulating these 32 bits.']**

---

## SECTION 4: Network ID and Host ID (10 minutes)

> "An IP address serves double duty — part of it identifies the NETWORK, and part identifies the specific HOST (device) on that network."

Draw:
```
IP Address:    192.168.1.5
               └─────┘ └─┘
               Network  Host
                Part    Part
```

> "But how do we know where the network part ends and the host part begins? That's what the **subnet mask** tells us."

**Traditional Classes (historical, still useful to know):**

| Class | First Octet Range | Network bits | Host bits | Default Mask |
|-------|-----------------|-------------|----------|-------------|
| A | 1–126 | 8 | 24 | 255.0.0.0 |
| B | 128–191 | 16 | 16 | 255.255.0.0 |
| C | 192–223 | 24 | 8 | 255.255.255.0 |

**Example:**
> "192.168.1.5 starts with 192, which is in Class C range. So:
> - Network: 192.168.1 (first 24 bits)
> - Host: 5 (last 8 bits)
> - Subnet Mask: 255.255.255.0"

**Problem with classful addressing:**
> "If a company needs 500 IP addresses, they'd need a Class B (65,534 hosts). That wastes 65,034 addresses. This is why we moved to CIDR — more flexible subnetting."

---

## SECTION 5: Subnet Mask (12 minutes)

> "A subnet mask is a 32-bit number where ALL the 1s come first, followed by ALL 0s. The 1s mark the network portion, the 0s mark the host portion."

**Example — 255.255.255.0:**
```
255.255.255.0 in binary:
11111111.11111111.11111111.00000000

Network portion: first 24 bits (all 1s)
Host portion:    last 8 bits (all 0s)
```

**How to apply the mask:**
> "To find the network address, we AND the IP address with the subnet mask."

```
IP:    11000000.10101000.00000001.00000101  (192.168.1.5)
Mask:  11111111.11111111.11111111.00000000  (255.255.255.0)
AND:   11000000.10101000.00000001.00000000  = 192.168.1.0 ← Network Address
```

**[INSTRUCTOR: Do this AND operation bit-by-bit at least once on board. Students need to see why it works: 1 AND 1 = 1, anything AND 0 = 0, so host bits are zeroed out.]**

**Example 2 — 255.255.0.0:**
```
IP:    172.16.5.10
Mask:  255.255.0.0 → 11111111.11111111.00000000.00000000
AND:   172.16.0.0 ← Network Address

Network: 172.16.0.0
Host ID: 5.10
```

### Special Addresses in a Subnet

For any subnet, there are two reserved addresses:

**Network Address (host bits all 0):** Identifies the subnet itself, not usable for a device
**Broadcast Address (host bits all 1):** Sends to ALL devices in the subnet

**Example for 192.168.1.0/24:**
```
Network Address:   192.168.1.0   (host bits = 00000000)
Broadcast Address: 192.168.1.255 (host bits = 11111111)
Usable hosts:      192.168.1.1 to 192.168.1.254 → 254 hosts
```

**Formula:** For a /N subnet:
```
Total addresses = 2^(32-N)
Usable hosts    = 2^(32-N) - 2
```

---

## SECTION 6: CIDR Notation (8 minutes)

> "Typing out `255.255.255.0` every time is tedious. CIDR — Classless Inter-Domain Routing — gives us a shorter way: just count how many 1 bits are in the mask."

**CIDR format:** `IP/prefix_length`

**Examples:**
```
192.168.1.5/24   →  subnet mask = 255.255.255.0  (24 ones)
10.0.0.0/8       →  subnet mask = 255.0.0.0      (8 ones)
172.16.0.0/16    →  subnet mask = 255.255.0.0    (16 ones)
192.168.1.128/25 →  subnet mask = 255.255.255.128 (25 ones)
```

**Reading /25:**
```
/25: 11111111.11111111.11111111.10000000 = 255.255.255.128

192.168.1.128/25:
- Network: 192.168.1.128 (first 25 bits)
- Hosts: 192.168.1.129 to 192.168.1.254
- Broadcast: 192.168.1.255
- Usable hosts: 2^7 - 2 = 126
```

**[PAUSE: Ask — "What does /30 mean? How many usable hosts?"**  
Answer: 2^2 - 2 = 2. These /30 subnets are commonly used for point-to-point links between routers.]

**Common CIDR reference:**

| CIDR | Hosts | Common Use |
|------|-------|-----------|
| /8 | 16.7 million | Large ISPs, Class A orgs |
| /16 | 65,534 | Large enterprises |
| /24 | 254 | Home/office networks |
| /28 | 14 | Small subnets |
| /30 | 2 | Router-to-router links |
| /32 | 0 (1 total) | Single host route |

---

## SECTION 7: Public vs. Private IP Addresses (8 minutes)

> "Not all IP addresses are created equal. Some are 'private' — they exist only inside your home or office network and are NOT routable on the internet."

**IANA-reserved private ranges:**

| Range | CIDR | Class | Common Use |
|-------|------|-------|-----------|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | A | Large corporate networks |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | B | Medium networks |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | C | Home/office networks |

> "Your home router hands out 192.168.x.x addresses to your devices. These addresses mean NOTHING outside your home — they can't be routed on the public internet. Your router handles the translation using NAT (Network Address Translation) — more on that in Lecture 11."

**Why private IPs exist:**
> "4.3 billion public IPs aren't enough for every device on earth. Private addresses let thousands of networks reuse the same address ranges internally. Only the router needs a public IP."

**Prove it — Ask students:**
> "How many devices are on the network right now in this room? If we all have 192.168.x.x addresses, are they all the same? Why isn't there a conflict?"

**Answer:** Because they're all on different private networks (or the same one), and private networks are isolated. The conflict only matters within one network.

---

## SECTION 8: Special / Reserved Addresses (5 minutes)

### Loopback Address: 127.0.0.1

> "The entire 127.0.0.0/8 range is reserved for loopback — packets sent to this range never leave the device. They loop back to the same machine."

```bash
ping 127.0.0.1   # Always works — even if you're offline
# Equivalent to "ping myself"
# Used to test if your network stack is working
```

> "When you run a local web server and access `localhost` in your browser, you're accessing `127.0.0.1`."

### Other Reserved Ranges

| Address | Purpose |
|---------|---------|
| 0.0.0.0 | Represents 'any' or 'unspecified' address |
| 127.0.0.0/8 | Loopback |
| 169.254.0.0/16 | APIPA (Automatic Private IP Addressing) — when DHCP fails |
| 255.255.255.255 | Limited broadcast — to all hosts on local subnet |

**APIPA in practice:**
> "If you've ever seen a Windows machine get a 169.254.x.x address, it means DHCP failed — it couldn't get an IP from the router, so it assigned itself one. This is usually a sign of a network problem."

---

## SECTION 9: Putting It Together — Reading Network Info on Your Machine (5 minutes)

**Live demo:**

```bash
# Linux/macOS
ip addr show
# or
ifconfig

# Windows
ipconfig

# Look for:
# inet 192.168.1.5/24 → your IP and CIDR
# brd 192.168.1.255   → broadcast address
```

**What each field tells you:**
- `192.168.1.5` = Your device's IP
- `/24` = 24-bit network, 8-bit host
- Broadcast `192.168.1.255` = Confirmed: /24 means host bits are last 8
- Default gateway (router): typically `.1` — `192.168.1.1`

---

## 📝 CLASS ACTIVITY (5 minutes)

**Quick problems — students solve on paper:**

1. What is the network address of `172.16.45.123/20`?
2. How many usable hosts in a `/26` subnet?
3. Is `10.200.50.6` a public or private IP?
4. What is the broadcast address of `192.168.10.128/25`?

**Answers:**
1. 172.16.32.0 (first 20 bits: 172.16.0010xxxx... → zeroed host = 172.16.32.0)
2. 2^6 - 2 = 62 hosts
3. Private (10.x.x.x range)
4. 192.168.10.255

---

## SUMMARY (3 minutes)

```
✅ IPv4 = 32-bit address in dotted-decimal, e.g. 192.168.1.5
✅ Network ID = routing (where), Host ID = specific device (who)
✅ Subnet Mask: 1s mark network bits, 0s mark host bits
✅ Network Address = IP AND mask (all host bits = 0)
✅ Broadcast Address = all host bits set to 1
✅ CIDR notation: /N = N network bits
✅ Private ranges: 10.x, 172.16-31.x, 192.168.x
✅ Loopback: 127.0.0.1 — always local, never leaves the machine
```

## 🔗 Preview of Next Lecture

> "Next lecture, we go deeper — subnetting problems, splitting a network into multiple pieces, and a preview of IPv6. We'll also explore how NAT hides all your private addresses behind a single public IP."

---

## ❓ Potential Student Questions

**Q: Can two devices on the same network have the same IP?**  
A: This causes an IP conflict. Both devices will have connectivity issues. DHCP prevents this by tracking which IPs it has assigned.

**Q: Is 192.168.0.0 itself a usable IP?**  
A: No — it's the network address. The first usable address would be 192.168.0.1.

**Q: My router is 192.168.1.1 — does the router need two IPs?**  
A: Yes. The router has at least two interfaces: one facing your home network (192.168.1.1, private) and one facing your ISP (a public IP). We'll cover this in NAT.

**Q: What happens if someone accidentally uses a private IP as a destination on the internet?**  
A: Routers on the internet drop packets destined for private addresses. Those packets never reach their destination.

---

*End of Lecture 3 Script*
