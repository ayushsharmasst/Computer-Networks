# Lecture 9: DNS and Internet Applications
## Instructor Script — SST Computer Networks (Term 5)

---

### 🎯 Learning Objectives
By the end of this session, students will:
- Explain the DNS resolution process step-by-step
- Describe the DNS hierarchy (root, TLD, authoritative servers)
- Parse a URL and understand each component
- Explain HTTP request/response cycle, methods, and status codes
- Understand how cookies and sessions work

**Duration:** ~90 minutes  
**Demo:** `dig`, `nslookup`, browser DevTools Network tab

---

## 📦 OPENING HOOK (5 minutes)

**[INSTRUCTOR: Open browser DevTools → Network tab → Navigate to google.com → Show DNS timing]**

> "When you type `google.com` and press Enter, your browser doesn't know where to send the request. It knows `google.com` but doesn't know the IP address. How does it find out?"

> "Every single web request starts with this question. And the answer involves a distributed database system that handles TRILLIONS of lookups per day — completely invisible to you. Let's demystify it."

---

## SECTION 1: Why DNS? (8 minutes)

> "Computers route to IP addresses — not domain names. A request to google.com ultimately becomes a request to 142.250.195.46. The mapping from names to IPs is what DNS handles."

**Why not just use IPs directly?**
1. IPs are hard to remember (and IPv6 is even worse)
2. IP addresses of servers change — DNS lets the name stay the same
3. Load balancing — a single name can resolve to multiple IPs
4. CDN routing — resolve to different IPs based on geography

**What is DNS?**
> "DNS = Domain Name System. It's the phonebook of the internet. You give it a name, it gives you an IP."

**Before DNS:**
> "In the early ARPANET days (1970s-80s), there was a single file called `HOSTS.TXT` maintained by Stanford Research Institute. Every computer would download this file to know all host-to-IP mappings. When ARPANET grew to hundreds of hosts, this didn't scale. DNS was invented in 1983 by Paul Mockapetris."

---

## SECTION 2: DNS Hierarchy (12 minutes)

**[INSTRUCTOR: Draw the DNS tree on the board]**

```
                          . (root)
                         / \
                       com  org  net  in  uk  ...
                      /    \
                   google  amazon  ...
                   /
                www   mail   maps   ...
```

> "DNS is a distributed, hierarchical database. The hierarchy mirrors the tree you see here."

### Three Types of DNS Servers

**1. Root Name Servers**
> "There are 13 sets of root name servers (named A through M, operated by 12 organizations, replicated worldwide — actually 1,700+ physical servers via anycast). They don't know the IP of google.com, but they know who does — the .com TLD servers."

**2. TLD (Top Level Domain) Servers**
> "TLD servers manage one level of the hierarchy. The .com TLD server knows all the nameservers registered for .com domains — google.com, amazon.com, etc."

Common TLDs: `.com`, `.org`, `.net`, `.edu`, `.in`, `.io`, `.ai`

**3. Authoritative Name Servers**
> "The authoritative server for google.com knows the actual IPs of google.com's hosts. This is the server that Google manages. When you update an IP for your domain, you update it here."

### The Recursive Resolver

> "Your ISP (or your configured DNS, like 8.8.8.8) runs a **recursive resolver**. This is the server that does all the work of walking the hierarchy on your behalf."

---

## SECTION 3: DNS Resolution — Full Walkthrough (15 minutes)

**[INSTRUCTOR: Walk through this step-by-step on the board. Draw each component as you go.]**

**Scenario:** You type `www.google.com` in your browser.

```
Step 1 — Browser Cache
  "Did I look this up recently?"
  → Found in cache (TTL not expired)? Return cached IP. Done.
  → Not found → proceed

Step 2 — OS Cache / hosts file
  Check /etc/hosts (Linux/Mac) or C:\Windows\System32\drivers\etc\hosts
  → Found? Return IP. Done.
  → Not found → proceed

Step 3 — Ask Recursive Resolver
  OS sends DNS query to configured DNS server (ISP's server or 8.8.8.8)
  Query: "What is the IP of www.google.com?"
  
Step 4 — Resolver Cache
  Has the resolver seen this before?
  → Found in cache? Return it. Done.
  → Not found → proceed to full resolution

Step 5 — Ask Root Server
  Resolver asks root: "www.google.com?"
  Root says: "I don't know, but for .com, ask 192.5.6.30 (com TLD server)"
  (This is a REFERRAL, not an answer)

Step 6 — Ask .com TLD Server
  Resolver asks TLD: "www.google.com?"
  TLD says: "I don't know the IP, but Google's nameservers are ns1.google.com, ns2.google.com..."
  (Another REFERRAL)

Step 7 — Ask Authoritative Server (Google's NS)
  Resolver asks Google's nameserver: "www.google.com?"
  Google's NS says: "www.google.com = 142.250.195.46" ← ANSWER!

Step 8 — Return and Cache
  Resolver sends answer back to your OS
  Caches it for TTL seconds (e.g., 300 seconds)
  Your browser caches it too
  
Step 9 — Browser connects
  Now browser knows IP: 142.250.195.46:80 → send HTTP request
```

**[INSTRUCTOR: Point out — in practice, recursive resolvers cache heavily. The root servers are only contacted when a new TLD or domain is seen for the first time.]**

**Demonstrate with `dig`:**

```bash
# Full recursive lookup
dig www.google.com

# Ask a specific nameserver
dig @8.8.8.8 www.google.com

# Trace the full resolution path
dig +trace www.google.com

# Lookup MX records (mail servers)
dig google.com MX

# Reverse DNS lookup (IP to name)
dig -x 142.250.195.46
```

### DNS Record Types

| Record | Meaning | Example |
|--------|---------|---------|
| A | IPv4 address of host | google.com → 142.250.195.46 |
| AAAA | IPv6 address of host | google.com → 2607:f8b0::200e |
| CNAME | Alias (canonical name) | www.google.com → google.com |
| MX | Mail exchanger server | google.com MX → smtp.google.com |
| NS | Nameserver for domain | google.com NS → ns1.google.com |
| TXT | Text record (used for SPF, DKIM, verification) | "v=spf1 include:..." |
| PTR | Reverse lookup (IP → name) | 46.195.250.142 → google.com |
| SOA | Start of Authority (zone metadata) | Serial, TTL, etc. |

---

## SECTION 4: URL Anatomy (5 minutes)

**[INSTRUCTOR: Write a complete URL on board and dissect each part]**

```
https://user:pass@www.example.com:8080/path/to/resource?q=search&page=2#section3

Protocol:   https://
User info:  user:pass@ (optional, rare)
Host:       www.example.com
Port:       :8080 (optional; default 80 for http, 443 for https)
Path:       /path/to/resource
Query:      ?q=search&page=2
Fragment:   #section3 (client-side only, not sent to server)
```

**Key points:**
- **Protocol/Scheme**: tells browser what protocol to use
- **Host**: goes to DNS for resolution
- **Port**: default 80 (HTTP) and 443 (HTTPS); only specified if non-default
- **Path**: the resource on the server
- **Query string**: parameters passed to the server (key=value pairs, separated by &)
- **Fragment**: anchor in the page; browser uses it locally, NOT sent to server

---

## SECTION 5: HTTP — HyperText Transfer Protocol (15 minutes)

### HTTP is Text-Based, Stateless

> "HTTP is a simple, text-based protocol. Requests and responses are human-readable. And crucially — HTTP is **stateless**: each request is completely independent. The server has no memory of who you are from one request to the next."

### HTTP Request Structure

```
GET /search?q=networks HTTP/1.1
Host: www.google.com
User-Agent: Mozilla/5.0 (Chrome/120)
Accept: text/html,application/json
Accept-Language: en-US
Connection: keep-alive
Cookie: session=abc123; prefs=dark_mode

[Body — only for POST, PUT, etc.]
```

**Parts of an HTTP request:**
1. **Request Line:** `METHOD /path HTTP/version`
2. **Headers:** Key: Value pairs providing metadata
3. **Blank line:** Separates headers from body
4. **Body:** Data sent with request (form data, JSON, file uploads)

### HTTP Methods

| Method | Purpose | Body? | Idempotent? |
|--------|---------|-------|-------------|
| **GET** | Retrieve a resource | No | Yes |
| **POST** | Create/send data | Yes | No |
| **PUT** | Replace entire resource | Yes | Yes |
| **PATCH** | Partial update | Yes | No |
| **DELETE** | Remove resource | No | Yes |
| **HEAD** | Like GET but only headers | No | Yes |
| **OPTIONS** | What methods are allowed? | No | Yes |

**[INSTRUCTOR: Ask — "Why is GET idempotent? What does idempotent mean?"  
Answer: calling it once or 100 times has the same effect — you get the resource, you don't change anything]**

**REST API example:**
```
GET    /api/users/42          → Get user with ID 42
POST   /api/users             → Create a new user
PUT    /api/users/42          → Replace user 42 entirely
PATCH  /api/users/42          → Update specific fields of user 42
DELETE /api/users/42          → Delete user 42
```

### HTTP Response Structure

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 4523
Date: Sat, 15 Aug 2026 10:00:00 GMT
Server: Google Frontend
Set-Cookie: session=xyz; HttpOnly; Secure; SameSite=Strict
Cache-Control: max-age=3600

<!DOCTYPE html>
<html>
...page content...
</html>
```

**Parts:**
1. **Status Line:** `HTTP/version STATUS_CODE STATUS_TEXT`
2. **Headers:** Response metadata (content type, length, caching, etc.)
3. **Blank line**
4. **Body:** The actual response data (HTML, JSON, image, etc.)

### HTTP Status Codes

**[INSTRUCTOR: Group by category — ask students if they've seen any of these]**

| Code | Meaning | When |
|------|---------|------|
| **1xx Information** | | |
| 100 | Continue | Client should continue sending body |
| **2xx Success** | | |
| 200 | OK | Standard success |
| 201 | Created | Resource was created (POST/PUT) |
| 204 | No Content | Success, nothing to return (DELETE) |
| **3xx Redirect** | | |
| 301 | Moved Permanently | URL has moved forever; update your bookmarks |
| 302 | Found (Temporary Redirect) | Use this URL for now |
| 304 | Not Modified | Cached version is still valid |
| **4xx Client Errors** | | |
| 400 | Bad Request | Malformed request |
| 401 | Unauthorized | Not authenticated |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 429 | Too Many Requests | Rate limited |
| **5xx Server Errors** | | |
| 500 | Internal Server Error | Generic server failure |
| 502 | Bad Gateway | Upstream server returned bad response |
| 503 | Service Unavailable | Server overloaded or down |
| 504 | Gateway Timeout | Upstream server too slow |

**[INSTRUCTOR: Ask — "What's the difference between 401 and 403?"  
401: I don't know who you are (not logged in) → send me your credentials  
403: I know who you are, but you don't have permission]**

---

## SECTION 6: Cookies and Sessions (10 minutes)

### The Stateless Problem

> "HTTP is stateless — every request is independent. But websites need to KNOW who you are across requests. How does Amazon know your shopping cart when you navigate from page to page?"

**Answer: Cookies**

### What is a Cookie?

> "A **cookie** is a small piece of data that the server sends to the browser, and the browser automatically includes in every subsequent request to that domain."

**Flow:**
```
1. You log in:
   POST /login with username + password

2. Server creates a session, sends:
   Set-Cookie: session_id=abc123; HttpOnly; Secure; Max-Age=86400

3. Browser stores the cookie

4. Every future request to this domain:
   GET /profile
   Cookie: session_id=abc123  ← browser adds automatically

5. Server looks up session_id=abc123 in its session store
   Finds your user account → knows it's you
```

### Cookie Attributes

| Attribute | Purpose |
|-----------|---------|
| **HttpOnly** | Cookie inaccessible to JavaScript → prevents XSS theft |
| **Secure** | Only sent over HTTPS |
| **SameSite=Strict** | Only sent for same-site requests → prevents CSRF |
| **SameSite=Lax** | Sent for top-level navigation from other sites |
| **Max-Age** | When the cookie expires (seconds) |
| **Domain** | Which domains can receive this cookie |
| **Path** | Which paths this cookie applies to |

### Cookie vs. Session

| | Cookie | Session |
|-|--------|---------|
| Stored | Browser | Server |
| Contains | Session ID (or data) | Actual user data |
| Size limit | ~4KB | No practical limit |
| Security | Can be stolen if not secured | Data stays on server |

**Session store:** The server stores session data in memory, database, or Redis. The cookie just carries the session ID.

**[INSTRUCTOR: Show DevTools → Application → Cookies to see real cookies]**

### Authentication vs Authorization

> "Two concepts often confused:"

- **Authentication:** Proving who you are (login with password, fingerprint, OTP)
- **Authorization:** What you're allowed to do (role-based access)

**HTTP headers involved:**
```
Authorization: Bearer eyJhbGc...      ← JWT token (stateless auth)
Authorization: Basic dXNlcjpwYXNz    ← base64(user:pass) — insecure
```

---

## SECTION 7: Browser DevTools Demo (5 minutes)

**[INSTRUCTOR: Open DevTools in Chrome → Network tab → Navigate to any website]**

**Point out:**
- First request: HTML document
- Subsequent requests: CSS, JavaScript, images, fonts, API calls
- DNS lookup time (in timing breakdown)
- Connection setup time
- TTFB (Time to First Byte) — server processing time
- Content transfer time

**For one request, expand and show:**
- Request headers (User-Agent, Accept, Cookie)
- Response headers (Content-Type, Set-Cookie, Cache-Control)
- Status code
- Response body (HTML/JSON)

---

## SUMMARY (5 minutes)

```
✅ DNS: distributed hierarchical database; name → IP mapping
✅ Resolution: Browser cache → OS → Recursive resolver → Root → TLD → Authoritative
✅ DNS records: A, AAAA, CNAME, MX, NS, TXT, PTR
✅ URL: protocol://host:port/path?query#fragment
✅ HTTP: text-based, stateless, request-response
✅ Methods: GET (read), POST (create), PUT (replace), PATCH (update), DELETE
✅ Status codes: 2xx (success), 3xx (redirect), 4xx (client error), 5xx (server error)
✅ Cookies: server-sent data; browser returns on every request; enables sessions
✅ HttpOnly, Secure, SameSite are key security attributes
```

---

*End of Lecture 9 Script*
