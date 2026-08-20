# Lecture 9: DNS and Internet Applications
## Student Notes — SST Computer Networks (Term 5)

---

## 🎯 Learning Objectives
- Explain the DNS resolution process step-by-step
- Understand the three-tier DNS hierarchy
- Interpret DNS record types
- Parse URLs and understand each component
- Explain the HTTP request/response cycle, methods, and status codes
- Understand how cookies and sessions work

---

## 1. Why DNS?

Computers communicate using IP addresses, not domain names. **DNS (Domain Name System)** translates human-readable domain names (e.g., `google.com`) to IP addresses (e.g., `142.250.195.46`).

**Why not use IPs directly?**
- IPs are hard to remember (especially IPv6)
- Servers change IPs; the name stays constant
- One name can map to many IPs (load balancing)
- Geographic routing via DNS (CDNs give you the nearest IP)

---

## 2. DNS Hierarchy

DNS is a **distributed, hierarchical** system — no single server knows all answers.

```
                    . (root)
                   /|\ 
                 com org in net uk ...
                /   \
           google  amazon ...
           /   \
          www  mail maps ...
```

### Three Types of DNS Servers

| Type | Role | Example |
|------|------|---------|
| **Root Servers** | Know TLD server addresses (13 logical servers, A-M) | Operated by ICANN, Verisign, etc. |
| **TLD Servers** | Know authoritative NS for each domain in that TLD | Verisign runs .com TLD |
| **Authoritative Servers** | Know the actual IP for a specific domain | ns1.google.com |

### Recursive Resolver

Your ISP or configured DNS (e.g., `8.8.8.8`) runs a **recursive resolver** — it queries the hierarchy on your behalf and caches results.

---

## 3. DNS Resolution — Step by Step

**Query:** Browser wants IP for `www.google.com`

```
Step 1: Browser cache
  "Do I have a recent cached answer?"
  YES → use it (if TTL not expired). DONE.
  NO → continue

Step 2: OS resolver cache + /etc/hosts
  Check local system cache
  YES → return. DONE.
  NO → continue

Step 3: Ask Recursive Resolver (ISP's DNS or 8.8.8.8)
  Query: "What is www.google.com?"

Step 4: Resolver cache
  Already cached? YES → return. DONE.
  NO → start walking hierarchy

Step 5: Resolver asks Root Server
  Root: "I don't know, but the .com TLD is at 192.5.6.30"
  → REFERRAL

Step 6: Resolver asks .com TLD Server
  TLD: "I don't know the IP, but Google's nameservers are ns1.google.com"
  → REFERRAL

Step 7: Resolver asks ns1.google.com (Authoritative)
  ns1.google.com: "www.google.com = 142.250.195.46, TTL=300"
  → ANSWER

Step 8: Resolver caches and returns answer to browser
Step 9: Browser connects to 142.250.195.46
```

### DNS Caching and TTL

- **TTL (Time To Live):** Each DNS record specifies how many seconds it should be cached
- After TTL expires, the resolver must re-query
- Low TTL = flexibility (fast changes) but more DNS traffic
- High TTL = stable (fewer queries) but slow to propagate changes

---

## 4. DNS Record Types

| Record | Purpose | Example |
|--------|---------|---------|
| **A** | Domain → IPv4 | `google.com → 142.250.195.46` |
| **AAAA** | Domain → IPv6 | `google.com → 2607:f8b0::200e` |
| **CNAME** | Alias (canonical name) | `www.example.com → example.com` |
| **MX** | Mail server for domain | `google.com MX → smtp.google.com` |
| **NS** | Nameserver for domain | `google.com NS → ns1.google.com` |
| **TXT** | Arbitrary text | SPF, DKIM, site verification |
| **PTR** | IP → Domain (reverse) | `46.195.250.142 → google.com` |
| **SOA** | Zone metadata | Serial, refresh, expire, TTL |

### DNS Lookup Commands

```bash
# Basic lookup
dig www.google.com
nslookup www.google.com

# Ask a specific DNS server
dig @8.8.8.8 www.google.com

# Trace full resolution path
dig +trace www.google.com

# Specific record type
dig google.com MX       # Mail records
dig google.com AAAA     # IPv6 records
dig google.com NS       # Nameserver records

# Reverse lookup
dig -x 142.250.195.46
```

---

## 5. URL Anatomy

```
https://user:pass@www.example.com:8080/path/to/resource?q=search&page=2#section3

┌──────────────────────────────────────────────────────────────────────────────┐
│ https://      → Protocol/Scheme                                              │
│ user:pass@    → User info (optional, rarely used)                            │
│ www.example.com → Hostname (resolved by DNS)                                 │
│ :8080         → Port (default: 80 HTTP, 443 HTTPS)                          │
│ /path/to/resource → Path on server                                           │
│ ?q=search&page=2 → Query string (key=value pairs)                           │
│ #section3     → Fragment (client-side anchor; NOT sent to server)            │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Key notes:**
- Port is omitted when it's the default (80 for HTTP, 443 for HTTPS)
- The fragment `#section3` never leaves the browser — it's for page anchors
- Query parameters are sent to the server and must be URL-encoded

---

## 6. HTTP — HyperText Transfer Protocol

**Key properties:**
- **Text-based**: requests and responses are human-readable ASCII
- **Stateless**: each request is independent; server has no memory of prior requests
- **Request-response**: client always initiates; server always responds

### HTTP Request Format

```
METHOD /path HTTP/1.1           ← Request Line
Host: www.example.com           ← Headers (required in HTTP/1.1)
User-Agent: Mozilla/5.0
Accept: text/html,application/json
Connection: keep-alive
Cookie: session=abc123
                                ← Blank line (end of headers)
{ "key": "value" }              ← Body (POST/PUT only)
```

### HTTP Methods

| Method | Purpose | Has Body | Idempotent | Safe |
|--------|---------|---------|-----------|------|
| **GET** | Retrieve resource | No | Yes | Yes |
| **POST** | Create / send data | Yes | No | No |
| **PUT** | Replace resource | Yes | Yes | No |
| **PATCH** | Partial update | Yes | No | No |
| **DELETE** | Remove resource | No | Yes | No |
| **HEAD** | Headers only (no body) | No | Yes | Yes |
| **OPTIONS** | Query allowed methods | No | Yes | Yes |

> **Idempotent**: calling the same operation multiple times has the same effect as calling it once.  
> **Safe**: operation doesn't modify anything on the server.

**REST API patterns:**
```
GET    /api/users       → List all users
GET    /api/users/42    → Get user 42
POST   /api/users       → Create new user
PUT    /api/users/42    → Replace user 42
PATCH  /api/users/42    → Partial update of user 42
DELETE /api/users/42    → Delete user 42
```

### HTTP Response Format

```
HTTP/1.1 200 OK                         ← Status Line
Content-Type: text/html; charset=UTF-8  ← Headers
Content-Length: 4523
Date: Sat, 15 Aug 2026 10:00:00 GMT
Set-Cookie: session=abc; HttpOnly; Secure
Cache-Control: max-age=3600
                                         ← Blank line
<!DOCTYPE html>                          ← Body
<html>...</html>
```

---

## 7. HTTP Status Codes

| Range | Category | Common Examples |
|-------|----------|----------------|
| **1xx** | Informational | 100 Continue |
| **2xx** | Success | 200 OK, 201 Created, 204 No Content |
| **3xx** | Redirection | 301 Moved Permanently, 302 Found, 304 Not Modified |
| **4xx** | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Rate Limited |
| **5xx** | Server Error | 500 Internal Server Error, 502 Bad Gateway, 503 Unavailable, 504 Timeout |

### Key Distinctions

| Codes | Difference |
|-------|-----------|
| 301 vs 302 | 301 = permanent redirect (browser/SEO updates); 302 = temporary |
| 401 vs 403 | 401 = not authenticated (who are you?); 403 = not authorized (I know you, but no) |
| 500 vs 502 | 500 = this server failed; 502 = upstream server gave bad response |

---

## 8. Cookies and Sessions

### The Stateless Problem

HTTP has no memory between requests. Cookies solve this: a server gives you a token, your browser returns it on every request.

### Cookie Flow

```
1. Login POST → server authenticates you
2. Server responds: Set-Cookie: session_id=abc123; HttpOnly; Secure; Max-Age=86400
3. Browser stores cookie for this domain
4. All future requests include: Cookie: session_id=abc123
5. Server looks up session_id → finds your session → knows who you are
```

### Cookie Security Attributes

| Attribute | Effect |
|-----------|--------|
| **HttpOnly** | JavaScript cannot access cookie → protects against XSS |
| **Secure** | Cookie only sent over HTTPS → prevents interception |
| **SameSite=Strict** | Not sent on cross-site requests → prevents CSRF |
| **SameSite=Lax** | Sent on top-level navigation from other sites |
| **Max-Age=N** | Cookie expires after N seconds |
| **Domain=.example.com** | Accessible by all subdomains |

### Cookie vs. Session

| | Cookie | Server Session |
|-|--------|---------------|
| **Stored at** | Browser | Server (DB, Redis) |
| **Contains** | Session ID (or data) | Actual user data |
| **Size limit** | ~4 KB | No practical limit |
| **Visibility** | Visible in browser | Hidden on server |

### Authentication vs Authorization

| Term | Definition | Example |
|------|-----------|---------|
| **Authentication** | Proving identity | Logging in with password |
| **Authorization** | Permission check | Checking if user has admin role |

**HTTP Authorization header:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...  ← JWT (stateless)
Authorization: Basic dXNlcjpwYXNz              ← Base64(user:pass) — avoid!
```

---

## 📌 Key Takeaways

1. **DNS** = distributed hierarchical database; translates names to IPs
2. **Resolution order**: browser cache → OS → recursive resolver → root → TLD → authoritative
3. **DNS records**: A (IPv4), AAAA (IPv6), CNAME (alias), MX (mail), NS (nameserver), TXT (text)
4. **URL** = protocol://host:port/path?query#fragment; fragment never sent to server
5. **HTTP** = text-based, stateless, request-response protocol
6. **Methods**: GET (read), POST (create), PUT (replace), PATCH (update), DELETE
7. **Status codes**: 2xx success, 3xx redirect, 4xx client error, 5xx server error
8. **Cookies** enable stateful sessions over stateless HTTP; HttpOnly+Secure are critical security flags

---

## 🧠 Quick Self-Check Questions

1. In what order does the OS check locations when resolving a DNS name?
2. What is a CNAME record and when would you use it?
3. A URL ends in `#contact`. Is `#contact` sent to the web server? Why or why not?
4. Difference between HTTP PUT and PATCH?
5. You get a 403 response. What does this mean? How is it different from 401?
6. What does `Set-Cookie: session=abc; HttpOnly; Secure; SameSite=Strict` mean?
7. Why does a server give you a session_id in a cookie rather than your actual user data?
8. What DNS record type would you use to configure email for your domain?

---

*Lecture 9 of 13 — Computer Networks, Term 5, SST*
