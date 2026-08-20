# Lecture 3: IP Addresses and Subnetting I
## Assignment — SST Computer Networks (Term 5)

**Instructions:** Each question may have **one or more correct answers**. Select all that apply.  
Mark the question type for your reference: **(T)** = Theoretical, **(C)** = Conceptual, **(S)** = Scenario-based.

---

### Q1 (T) — Properties of IPv4 Addresses

Which of the following correctly describe IPv4 addresses?

A. An IPv4 address is a 32-bit number  
B. IPv4 uses dotted-decimal notation, with four 8-bit sections called octets  
C. Each octet in an IPv4 address can range from 0 to 255  
D. The total number of possible IPv4 addresses is approximately 4.3 billion (2^32)  
E. IPv4 addresses are 64-bit numbers to accommodate more devices  

**Correct Answers: A, B, C, D**

*Explanation: IPv4 is 32 bits (A), written as four octets in dotted-decimal (B), each 0–255 (C), giving 2^32 ≈ 4.3 billion possible addresses (D). E is wrong — 128-bit addressing is IPv6, not IPv4.*

---

### Q2 (T) — Advantages of IP Addresses Over MAC Addresses

According to the lecture, IP addresses solve problems that MAC addresses cannot. Which of the following correctly describe why IP addresses are necessary?

A. IP addresses are globally meaningful — every public IP on the internet is unique  
B. IP addresses are hierarchical, embedding location information that helps with routing  
C. IP addresses are assigned logically and can be changed, unlike MAC addresses burned into hardware  
D. IP addresses work inside a single building, just like MAC addresses  
E. IP addresses eliminate the need for MAC addresses entirely  

**Correct Answers: A, B, C**

*Explanation: The three problems IP solves stated in the lecture: global uniqueness (A), hierarchical structure for routing (B), and logical (changeable) assignment (C). D is wrong — the point is that IP works *across* networks, not just locally like MAC. E is wrong — both address types co-exist and serve different layers.*

---

### Q3 (T) — Binary Conversion

Which of the following binary-to-decimal or decimal-to-binary conversions are correct?

A. 192 in binary = 11000000  
B. 168 in binary = 10101000  
C. 255 in binary = 11111111  
D. 10 in binary = 00001010  
E. 11000000 in decimal = 128  

**Correct Answers: A, B, C, D**

*Explanation: 192 = 128+64 = 11000000 (A). 168 = 128+32+8 = 10101000 (B). 255 = all bits set = 11111111 (C). 10 = 8+2 = 00001010 (D). E is wrong — 11000000 = 128+64 = 192, not 128.*

---

### Q4 (C) — Subnet Mask Purpose

Which of the following correctly describe what a subnet mask does?

A. It is a 32-bit number where all the 1s come first, followed by all 0s  
B. The 1-bits in the mask mark the network portion of an IP address  
C. The 0-bits in the mask mark the host portion of an IP address  
D. To find the network address, you perform a bitwise AND between the IP and the subnet mask  
E. A subnet mask determines which domain name corresponds to an IP address  

**Correct Answers: A, B, C, D**

*Explanation: A, B, C, and D all correctly describe subnet mask structure and usage. E is wrong — domain name to IP mapping is done by DNS, not the subnet mask.*

---

### Q5 (T) — AND Operation to Find the Network Address

A device has the IP address 172.16.5.10 and subnet mask 255.255.0.0. Which of the following are correct outcomes of applying the subnet mask?

A. The network address is 172.16.0.0  
B. The AND operation zeroes out the host bits while preserving the network bits  
C. The network address is 172.16.5.0  
D. The host portion of 172.16.5.10 with this mask is 5.10 (the last two octets)  
E. 1 AND 0 = 1 in the bitwise AND operation used here  

**Correct Answers: A, B, D**

*Explanation: 172.16.5.10 AND 255.255.0.0 → zeroes out the last 16 bits → 172.16.0.0 (A). The AND operation preserves bits where the mask is 1 and zeros where mask is 0 (B). The host portion is the last two octets: 5.10 (D). C is wrong — /16 mask zeroes the last 16 bits, not just the last 8. E is wrong — 1 AND 0 = 0 (not 1). That's the whole point: 0s in the mask zero out the host bits.*

---

### Q6 (C) — Network Address and Broadcast Address

For the subnet 192.168.10.0/24, which of the following are correct?

A. The network address is 192.168.10.0 (host bits all set to 0)  
B. The broadcast address is 192.168.10.255 (host bits all set to 1)  
C. The number of usable host addresses is 254  
D. The address 192.168.10.0 can be assigned to a device as its IP  
E. The address 192.168.10.255 can be assigned to a printer on this subnet  

**Correct Answers: A, B, C**

*Explanation: Network address = 192.168.10.0 with all host bits 0 (A). Broadcast = 192.168.10.255 with all host bits 1 (B). Usable = 2^8 - 2 = 254 (C). D is wrong — the network address is reserved and cannot be assigned to a device. E is wrong — the broadcast address is also reserved; assigning it to a device would cause network problems.*

---

### Q7 (T) — CIDR Notation and Host Count

Which of the following CIDR prefix lengths and their host counts are correctly paired?

A. /24 → 254 usable hosts  
B. /30 → 2 usable hosts  
C. /28 → 14 usable hosts  
D. /16 → 65,534 usable hosts  
E. /32 → 254 usable hosts  

**Correct Answers: A, B, C, D**

*Explanation: /24: 2^8-2=254 (A). /30: 2^2-2=2 (B). /28: 2^4-2=14 (C). /16: 2^16-2=65,534 (D). E is wrong — /32 means 32 network bits, 0 host bits, giving 2^0-2 = -1, i.e., zero usable hosts; /32 is used to represent a single specific host route, not a range.*

---

### Q8 (T) — Classful Addressing

Which of the following correctly describe the traditional (classful) IP address classes?

A. Class A addresses have first octets ranging from 1 to 126, with 8 network bits  
B. Class B addresses have first octets ranging from 128 to 191, with 16 network bits  
C. Class C addresses have first octets ranging from 192 to 223, with 24 network bits  
D. A Class B subnet supports up to 65,534 usable host addresses  
E. Class A subnets have fewer host bits than Class C, so they support fewer devices  

**Correct Answers: A, B, C, D**

*Explanation: A, B, C are the classful ranges from the lecture table. D is correct — Class B has 16 host bits → 2^16-2=65,534. E is wrong — it's the opposite: Class A has 24 host bits (more hosts), Class C has only 8 host bits (fewer hosts).*

---

### Q9 (T) — Private IP Address Ranges

Which of the following IP addresses belong to IANA-reserved private ranges?

A. 10.200.50.6  
B. 172.20.5.1  
C. 192.168.100.254  
D. 45.123.67.89  
E. 172.32.0.1  

**Correct Answers: A, B, C**

*Explanation: 10.x.x.x (10.0.0.0/8) — A is private. 172.16.0.0–172.31.255.255 — B (172.20.x.x) is private. 192.168.x.x — C is private. D (45.x.x.x) is a public address. E (172.32.x.x) is outside the private range for class B — the private range ends at 172.31.255.255, so 172.32.x.x is public.*

---

### Q10 (C) — Why Private Addresses Exist

Which of the following correctly explain why private IP address ranges were created?

A. There are not enough public IPv4 addresses for every device on earth  
B. Private addresses allow thousands of different networks to reuse the same internal address ranges without conflict  
C. Private addresses are more secure because they are not routable on the public internet  
D. Private addresses eliminate the need for a router in a home network  
E. Only the router needs a public IP; all internal devices can share private addresses via NAT  

**Correct Answers: A, B, C, E**

*Explanation: Address exhaustion (A), reuse of the same ranges across isolated networks (B), and non-routability as a security benefit (C) are all discussed. E directly matches the lecture's explanation. D is wrong — a router is still needed; private addresses don't remove that requirement. In fact, NAT on the router is what enables private-to-public translation.*

---

### Q11 (T) — Special and Reserved Addresses

Which of the following correctly describe special IPv4 addresses?

A. 127.0.0.1 is the loopback address — packets sent there never leave the device  
B. The entire 127.0.0.0/8 range is reserved for loopback  
C. A device with IP 169.254.45.12 has likely failed to obtain an address from DHCP  
D. 255.255.255.255 is the limited broadcast address sent to all hosts on the local subnet  
E. 0.0.0.0 represents a specific device on the local network  

**Correct Answers: A, B, C, D**

*Explanation: A and B are correct — the whole /8 loopback range is reserved. C describes APIPA (169.254.0.0/16), which Windows assigns when DHCP fails. D correctly identifies 255.255.255.255 as the limited broadcast. E is wrong — 0.0.0.0 represents "any" or "unspecified" address, not a specific device.*

---

### Q12 (S) — Subnet Calculation

A network engineer is given the subnet 192.168.1.128/25. Which of the following are correct about this subnet?

A. The subnet mask in dotted-decimal is 255.255.255.128  
B. The number of usable host addresses is 126  
C. The broadcast address is 192.168.1.255  
D. The network address is 192.168.1.128  
E. This subnet can accommodate 254 usable hosts  

**Correct Answers: A, B, C, D**

*Explanation: /25 = 25 ones → 255.255.255.128 (A). Usable = 2^7-2 = 126 (B). Broadcast = host bits all 1s → 192.168.1.255 (C). Network address = 192.168.1.128 (D, host bits all 0). E is wrong — 254 usable hosts requires a /24, not a /25.*

---

### Q13 (S) — Reading Network Information

A Linux machine shows the following from `ip addr show`:  
```
inet 10.0.5.22/8 brd 10.255.255.255
```  
Which of the following are correct interpretations of this output?

A. The device's IP address is 10.0.5.22  
B. The network prefix length is 8 bits (Class A range)  
C. The broadcast address for this subnet is 10.255.255.255  
D. 10.0.5.22 is a public IP address routable on the internet  
E. The network address for this subnet is 10.0.0.0  

**Correct Answers: A, B, C, E**

*Explanation: A (IP = 10.0.5.22), B (/8 = 8 network bits), C (broadcast confirmed in output), and E (10.0.5.22 AND 255.0.0.0 = 10.0.0.0) are all correct. D is wrong — 10.x.x.x is a private range (10.0.0.0/8) and is NOT routable on the public internet.*

---

### Q14 (S) — APIPA and DHCP Failure

A student's Windows laptop shows the IP address 169.254.33.15 after connecting to the campus Wi-Fi. Which of the following are true about this situation?

A. The laptop attempted to get an IP address from a DHCP server but failed  
B. The laptop self-assigned this address using APIPA (Automatic Private IP Addressing)  
C. 169.254.0.0/16 is the APIPA address range used when DHCP fails  
D. The laptop can communicate with other devices on the internet using this address  
E. This IP address indicates a problem with the network — likely the DHCP server is unreachable  

**Correct Answers: A, B, C, E**

*Explanation: A 169.254.x.x address means DHCP failed (A) and the machine used APIPA to self-assign (B), from the reserved 169.254.0.0/16 range (C). This is a problem indicator (E). D is wrong — APIPA addresses are link-local only and not routable; the device cannot reach the internet with this address.*

---

### Q15 (C) — The /30 Subnet and Point-to-Point Links

The lecture mentions that /30 subnets are commonly used for router-to-router links. Which of the following explain why?

A. A /30 subnet has exactly 2 usable host addresses — one for each end of the link  
B. Using a /30 wastes fewer IP addresses than assigning a /24 to a link with only 2 devices  
C. A /30 subnet has 4 total addresses: 1 network, 2 usable hosts, 1 broadcast  
D. A /30 subnet can support up to 30 devices on a single link  
E. The formula 2^(32-30) - 2 = 2 gives the correct number of usable hosts for a /30  

**Correct Answers: A, B, C, E**

*Explanation: /30 has 2 usable hosts (A), which is all that's needed for a two-endpoint link — using a larger subnet wastes addresses (B). Total addresses = 2^2 = 4: network, 2 usable, broadcast (C). The formula confirms 2^(32-30) - 2 = 4 - 2 = 2 (E). D is wrong — /30 supports exactly 2 hosts, not 30; confusing prefix length with host count is a common error.*

---

*Lecture 3 Assignment — 15 Questions | Computer Networks, Term 5, SST*
