# Lecture 4: IP Addresses and Subnetting II
## Instructor Script — SST Computer Networks (Term 5)

---

### 🎯 Learning Objectives
By the end of this session, students will:
- Solve subnetting problems (given requirements, design a subnet)
- Understand Variable Length Subnet Masking (VLSM) at a high level
- Know the key differences between IPv4 and IPv6
- Explain what NAT does and why it exists

**Duration:** ~90 minutes  
**Teaching Style:** Problem-solving heavy; real-world scenarios

---

## 📦 OPENING HOOK (5 minutes)

**[INSTRUCTOR: Put this scenario on the board]**

> "You're the network engineer for a startup that just got a block of 256 IP addresses: `192.168.10.0/24`. You have four departments:
> - Engineering: 60 employees
> - Marketing: 30 employees
> - HR: 10 employees
> - Management: 5 employees
>
> How do you split these 256 IPs efficiently across four departments?"

**Take 2-3 responses. Then say:**

> "If you give everyone a /24, Marketing and HR and Management will waste hundreds of IPs. But if everyone gets /30, Engineering won't have enough. The solution is subnetting — carving a large block into smaller pieces of the right size."

---

## SECTION 1: Subnetting Problems — Methodology (15 minutes)

**[INSTRUCTOR: Build a step-by-step process on the board.]**

### The Subnetting Process

**Step 1:** Determine how many hosts you need  
**Step 2:** Find the smallest power of 2 that fits: hosts needed + 2  
**Step 3:** Host bits = log₂(that number), so Prefix = 32 - host bits  
**Step 4:** Calculate the subnet size (block size = 2^host bits)  
**Step 5:** List the subnets

### Problem 1 — You need 50 hosts per subnet

```
Step 1: 50 hosts needed
Step 2: 50 + 2 = 52 → next power of 2 = 64 → 2^6
Step 3: Host bits = 6, Prefix = 32 - 6 = /26
Step 4: Subnet size = 2^6 = 64 addresses per subnet
Step 5: Usable hosts = 64 - 2 = 62
```

**Subnets of 10.0.0.0/26:**
```
Subnet 1: 10.0.0.0   – 10.0.0.63   (hosts: 10.0.0.1 – 10.0.0.62)
Subnet 2: 10.0.0.64  – 10.0.0.127  (hosts: 10.0.0.65 – 10.0.0.126)
Subnet 3: 10.0.0.128 – 10.0.0.191
Subnet 4: 10.0.0.192 – 10.0.0.255
```

**Key insight:** Within a /24, you get four /26 subnets. Makes sense: 2^(26-24) = 4.

### Problem 2 — Given 192.168.10.0/24, how many /28 subnets?

```
/28: 4 host bits → 16 addresses per subnet
Number of /28 subnets in /24: 2^(28-24) = 2^4 = 16 subnets
Usable hosts per subnet: 16 - 2 = 14
```

**List first 4 subnets:**
```
192.168.10.0   – 192.168.10.15   (hosts: .1 to .14)
192.168.10.16  – 192.168.10.31   (hosts: .17 to .30)
192.168.10.32  – 192.168.10.47
192.168.10.48  – 192.168.10.63
... (12 more)
```

### The Startup Problem from the Hook

**Requirements:**
- Engineering: 60 hosts → /26 (62 usable)
- Marketing: 30 hosts → /27 (30 usable) — exact fit!
- HR: 10 hosts → /28 (14 usable)
- Management: 5 hosts → /29 (6 usable)

**Allocate from 192.168.10.0/24:**

```
Engineering (60 hosts):   192.168.10.0/26   → 192.168.10.0 – 192.168.10.63
Marketing   (30 hosts):   192.168.10.64/27  → 192.168.10.64 – 192.168.10.95
HR          (10 hosts):   192.168.10.96/28  → 192.168.10.96 – 192.168.10.111
Management   (5 hosts):   192.168.10.112/29 → 192.168.10.112 – 192.168.10.119
Remaining unused:         192.168.10.120/29 and beyond
```

This is **Variable Length Subnet Masking (VLSM)** — different subnets have different prefix lengths.

---

## SECTION 2: Variable Length Subnetting — VLSM (8 minutes)

**[INSTRUCTOR: This is marked Optional in the syllabus but is worth 5 minutes since we've already touched it.]**

> "In the old days (classful routing), every subnet in a network had the same mask. If you picked /26, ALL your subnets were /26. This was wasteful."

> "VLSM allows different subnets to have different sizes. Modern routing protocols (OSPF, RIP v2, BGP) all support VLSM."

**Golden rule for VLSM:**
> "Always allocate the LARGEST subnet first. Work from the biggest requirement down."

**Why?** Subnets must not overlap. Starting with the largest prevents awkward gaps.

**Example — Wrong order (don't do this):**
```
❌ HR first (192.168.10.0/28 = .0 to .15)
❌ Engineering next needs 64 addresses starting at .16
   But .16 to .79 works... only if we haven't fragmented the space
```

**Correct order:**
```
✅ Engineering first: .0/26 (takes .0 to .63)
✅ Marketing next:    .64/27 (takes .64 to .95)
✅ HR next:           .96/28 (takes .96 to .111)
✅ Management last:   .112/29 (takes .112 to .119)
✅ No overlap, no waste
```

---

## SECTION 3: IPv6 — Why It Exists and How It Works (15 minutes)

**[INSTRUCTOR: Draw the contrast on board: IPv4 vs IPv6 address size]**

> "We said IPv4 has about 4.3 billion addresses. We've essentially run out — IANA exhausted its free pool in 2011, regional registries ran out between 2012–2020. The internet survives because of NAT (we'll get to that), but the real long-term solution is IPv6."

### IPv6 Basics

**Size:** 128 bits (vs IPv4's 32 bits)

**Total addresses:** 2¹²⁸ = 3.4 × 10³⁸ — 340 undecillion addresses
> "That's more than 50 octillion addresses per person on Earth — effectively unlimited."

**Format:** 8 groups of 4 hexadecimal digits, separated by colons:
```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
```

### IPv6 Shortening Rules

**Rule 1:** Leading zeros in a group can be omitted
```
0db8 → db8
0000 → 0
```

**Rule 2:** One run of consecutive all-zero groups can be replaced with `::`
```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
→ 2001:db8:85a3::8a2e:370:7334
```

**[PAUSE: Ask — "Can you use :: twice in the same address?"  
Answer: NO — ambiguous, not allowed. Only once per address.]**

**More examples:**
```
::1                           → loopback (equivalent of 127.0.0.1)
fe80::1                       → link-local address
2001:db8::/32                 → documentation/example prefix
```

### IPv6 Address Types

| Type | Prefix | Description |
|------|--------|-------------|
| Global Unicast | 2000::/3 | Public routable address |
| Link-Local | fe80::/10 | Local segment only (like 169.254.x.x) |
| Loopback | ::1/128 | Loopback (like 127.0.0.1) |
| Unspecified | ::/128 | No address assigned |
| Multicast | ff00::/8 | Send to a group |

### Key Differences: IPv4 vs IPv6

| Feature | IPv4 | IPv6 |
|---------|------|------|
| Address size | 32 bits | 128 bits |
| Total addresses | 4.3 billion | 3.4 × 10³⁸ |
| Notation | Dotted decimal | Colon-separated hex |
| Header complexity | Variable (options) | Fixed 40-byte header |
| Fragmentation | By routers and hosts | Only by source host |
| Broadcast | Yes | No (multicast instead) |
| NAT required? | Yes (due to shortage) | No |
| ARP | Yes | No (uses NDP instead) |

### IPv6 in Practice

> "IPv6 adoption has been slow. Most of the internet still runs on IPv4 with NAT. As of 2024, about 45% of Google traffic is over IPv6. Your phones on 4G/5G likely have both an IPv4 and IPv6 address."

```bash
# Check if your machine has an IPv6 address
ip -6 addr show
# Look for global unicast addresses (2xxx:...)
```

---

## SECTION 4: Default Gateway (8 minutes)

> "We now know about IP addresses and subnets. But when you send data to an IP that's OUTSIDE your subnet, how does your machine know where to start?"

**The answer: Default Gateway**

> "The **default gateway** is the IP address of your router — the first device that can forward packets outside your local network."

**Analogy:**
> "Imagine your office building. To send an email to someone inside the building, you walk down the hall. But to send a letter to someone in another city, you go to the building's mailroom — that's the default gateway."

**How it works:**

When your computer wants to send a packet:
1. Check: Is the destination IP in my subnet? (IP AND mask = my network address?)
2. **YES** → Send directly using ARP to find MAC address
3. **NO** → Send to the **default gateway** (the router)

```
Your IP:          192.168.1.5/24
Default gateway:  192.168.1.1

Sending to 192.168.1.20 → same /24 → send directly
Sending to 8.8.8.8     → different network → send to 192.168.1.1 first
```

**[INSTRUCTOR: Draw a diagram on board showing the router connecting the home network to the internet]**

**Checking your gateway:**
```bash
ip route show          # Linux — look for "default via X.X.X.X"
route print            # Windows
netstat -rn            # macOS/Linux
```

---

## SECTION 5: Broadcast and Network Addresses Revisited (5 minutes)

**Quick review — three special addresses you must know:**

| Address | Description | Example (/24) |
|---------|-------------|--------------|
| Network Address | First address, identifies the subnet | 192.168.1.0 |
| Broadcast Address | Last address, reaches all hosts in subnet | 192.168.1.255 |
| Default Gateway | Usually .1, the router | 192.168.1.1 |

**Why does broadcast exist?**
> "Some protocols need to talk to everyone on the local network at once — ARP, DHCP. They send to the broadcast address and every device processes the message."

**Directed broadcast:** Send to the broadcast address of another subnet (e.g., 192.168.2.255 from outside that subnet). Most routers block this for security.

**Limited broadcast:** 255.255.255.255 — only reaches devices on the same subnet segment. Never forwarded by routers.

---

## SECTION 6: Introduction to NAT (12 minutes)

### The Problem NAT Solves

> "Your home has 5 devices: laptop, phone, TV, tablet, smart speaker. But your ISP gives you only ONE public IP address. How do all 5 devices access the internet simultaneously?"

**Answer: NAT — Network Address Translation**

### How NAT Works

> "Your router maintains a **translation table** that maps (private IP, port) ↔ (public IP, port)."

**Setup:**
```
Home network: 192.168.1.0/24
Public IP:    203.0.113.5 (given by ISP to router)
```

**Outgoing packet from your laptop:**
```
Laptop sends:
  Src: 192.168.1.5:52341 → Dst: 142.250.195.46:80

Router translates:
  Src: 203.0.113.5:10001 → Dst: 142.250.195.46:80  ← replaces private IP
  Stores in table: 10001 ↔ 192.168.1.5:52341
```

**Response comes back:**
```
Google sends back:
  Src: 142.250.195.46:80 → Dst: 203.0.113.5:10001

Router looks up translation table:
  10001 → 192.168.1.5:52341
  Forwards to laptop as:
  Src: 142.250.195.46:80 → Dst: 192.168.1.5:52341
```

**[INSTRUCTOR: Draw this on board — two columns: Internet-facing and LAN-facing]**

### NAT Variants

| Type | Full Name | Description |
|------|-----------|-------------|
| **SNAT** | Source NAT | Changes source address (outgoing) |
| **DNAT** | Destination NAT | Changes destination (incoming — port forwarding) |
| **PAT/Masquerade** | Port Address Translation | Many-to-one; uses port numbers to distinguish |

**Port Forwarding (DNAT example):**
> "You're running a game server at home (192.168.1.10:27015). How do friends connect from outside? You set up port forwarding: public port 27015 → 192.168.1.10:27015. The router translates incoming packets to your private server."

### Limitations of NAT

1. **Breaks end-to-end connectivity:** The internet was designed so any device can talk to any other directly. NAT breaks this — devices behind NAT can't be reached directly from outside.
2. **Stateful:** The router must maintain state (the translation table). If it reboots, all existing connections break.
3. **Complex for some protocols:** FTP, SIP, some gaming protocols embed IP addresses in their payload — NAT has to inspect and rewrite those (Application Layer Gateways).

---

## SECTION 7: Practical — Reading Your Network Configuration (5 minutes)

**[INSTRUCTOR: Do live or show results]**

```bash
# Linux
ip addr show        → your IPs and subnet masks
ip route show       → routing table; "default via" = your gateway
cat /etc/resolv.conf → DNS servers

# Windows
ipconfig /all       → IPs, masks, gateway, DNS
```

**Example output:**
```
$ ip addr show eth0
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.1.5/24 brd 192.168.1.255 scope global eth0

$ ip route show
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 scope link
```

**Explain:**
- `inet 192.168.1.5/24` — my IP, /24 mask
- `brd 192.168.1.255` — broadcast (confirms /24)
- `default via 192.168.1.1` — my default gateway

---

## 📝 CLASS PROBLEMS (10 minutes)

**Solve at seats, then review as class:**

1. You need 3 subnets from 10.10.10.0/24: one with 100 hosts, one with 50 hosts, one with 20 hosts. Use VLSM to allocate.

2. Convert `2001:0db8:0000:0000:0000:0000:0000:0001` to shorthand IPv6.

3. A packet from 192.168.5.10 is going to 192.168.5.200. The mask is /24. Does it go to the gateway or direct?

4. A packet from 192.168.5.10 is going to 10.0.0.1. Mask is /24. Direct or gateway?

**Answers:**
1. 100 hosts → /25 (10.10.10.0/25, .0–.127), 50 hosts → /26 (10.10.10.128/26, .128–.191), 20 hosts → /27 (10.10.10.192/27, .192–.223)
2. `2001:db8::1`
3. Same /24 → direct (192.168.5.200 is in 192.168.5.0/24)
4. Different network → send to default gateway

---

## SUMMARY (3 minutes)

```
✅ Subnetting: Find host bits needed (power of 2), derive prefix
✅ VLSM: Different subnet sizes from same block — allocate largest first
✅ IPv6: 128 bits, hex, :: for zero compression, no NAT needed
✅ Default Gateway: Router's local IP — used when destination is outside your subnet
✅ Broadcast: 255 in all host bits — sends to all devices on subnet
✅ NAT: Translates private IPs to public IP using port mapping table
```

## 🔗 Preview of Next Lecture

> "We now know how addressing works. Next, we shift to routing — HOW does a packet actually find its way from your laptop to Google? That involves graph algorithms — Dijkstra, BFS, Bellman-Ford — and we'll see how routers use them to find the best path."

---

*End of Lecture 4 Script*
