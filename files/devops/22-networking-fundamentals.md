# Networking Fundamentals

> **TL;DR:** Know the OSI model in practice (L3/L4/L7), how DNS resolves a name, how TCP and TLS handshakes work, what CIDR notation means, and how NAT and load balancing fit into a cloud architecture.

## The OSI Stack — What You Actually Touch

| Layer | Name | Examples |
|-------|------|----------|
| 7 | Application | HTTP, gRPC, DNS, SSH |
| 6 | Presentation | TLS, MIME (rarely tuned) |
| 5 | Session | (rarely talked about) |
| 4 | Transport | TCP, UDP, QUIC |
| 3 | Network | IP, ICMP, routing |
| 2 | Data link | Ethernet, MAC, ARP |
| 1 | Physical | Cables, fiber |

Day-to-day:
- L3 = IP addresses + routing tables + VPCs.
- L4 = TCP (`SYN/SYN-ACK/ACK`), UDP, ports.
- L7 = HTTP requests, headers, paths, cookies.

## IP Addressing & CIDR

```
10.0.0.0/16        =>  10.0.0.0  - 10.0.255.255   (65,536 addresses)
10.0.1.0/24        =>  10.0.1.0  - 10.0.1.255     (256 addresses)
10.0.1.0/26        =>  10.0.1.0  - 10.0.1.63      (64 addresses)
10.0.1.0/30        =>  10.0.1.0  - 10.0.1.3       (4 addresses, point-to-point)
192.168.1.5/32     =>  one address (host route)
```

CIDR `/N` = the first N bits are network, the rest are host. Subnet mask follows: `/24` = `255.255.255.0`.

### Private (RFC 1918) Ranges

```
10.0.0.0/8         16M addresses — biggest, common for production VPCs
172.16.0.0/12      1M addresses
192.168.0.0/16     65K addresses — home routers, lab
```

Plan **non-overlapping** ranges across VPCs before you peer them or run a VPN — overlap is impossible to fix without renumbering.

### Reserved Addresses in a Subnet

In AWS subnets, the first 4 + last 1 IPs are reserved. In `10.0.1.0/24`:
- `.0` — network
- `.1` — VPC router
- `.2` — DNS (AWS-managed)
- `.3` — reserved future use
- `.255` — broadcast

So `/24` gives you ~251 usable IPs.

## DNS — How a Name Becomes an IP

```
client -> recursive resolver (8.8.8.8, ISP, corporate)
            -> root servers (.)
            -> TLD servers (.com)
            -> authoritative (example.com)
            -> answer
```

Each step has its own TTL caching. Changes take up to TTL to propagate.

### Record Types

| Type | Purpose |
|------|---------|
| **A** | Name → IPv4 |
| **AAAA** | Name → IPv6 |
| **CNAME** | Name → another name (cannot coexist with other records at the same name) |
| **ALIAS / ANAME** (cloud-specific) | Like CNAME but at apex (e.g., Route 53 alias) |
| **MX** | Mail exchanger |
| **TXT** | Free-form (SPF, DKIM, domain verification) |
| **SRV** | Service location (port + host) |
| **NS** | Nameservers for the zone |
| **CAA** | Which CAs may issue certs |

### TTL Tradeoffs

- High TTL (3600s+) → cached widely, fewer DNS queries, slower failover.
- Low TTL (60s) → fast cutover for blue-green / failover, more DNS traffic, ISPs may not honor low TTLs.

Before a planned cutover, **drop TTL to 60s a day in advance**, do the swap, then raise it back.

## TCP — the Reliable Bytestream

### Handshake

```
client                     server
  | --SYN-------> |        (I want to talk, my seq=X)
  | <--SYN-ACK--- |        (OK, my seq=Y, ack=X+1)
  | --ACK-------> |        (OK, ack=Y+1)
  | <==data====> |         (connected)
  | --FIN-------> |        (I'm done)
  | <--ACK------- |
  | <--FIN------- |
  | --ACK-------> |
```

Three-way handshake to open, four-way to close. Each RTT visible in latency.

### Why Pooling

A fresh TCP+TLS connection costs ~3 round trips + handshake CPU. Reuse: HTTP keep-alive, connection pools, gRPC channels. For DB clients, a connection pool is non-negotiable.

### Sockets and TIME_WAIT

After close, the kernel holds the socket in `TIME_WAIT` (default 60s on Linux) to handle late packets. Under high churn (many short-lived connections), you can exhaust ephemeral ports — `Cannot assign requested address`. Mitigations: connection reuse, larger ephemeral range, `SO_REUSEADDR`, `tcp_tw_reuse` (linux-specific).

## TLS Handshake

```
client                            server
  | --ClientHello-->|             (versions, ciphers, SNI)
  | <--ServerHello--|             (chosen cipher, cert chain)
  | <--Certificate--|
  | <--ServerHelloDone--|
  | --key exchange-->|            (ECDHE most common)
  | --ChangeCipherSpec-->|
  | --Finished (encrypted)-->|
  | <--ChangeCipherSpec--|
  | <--Finished (encrypted)--|
  | <======application data======>|
```

TLS 1.3 reduces to **1 RTT** (and 0-RTT for resumed sessions).

### Certificates

A cert chains from a leaf (your domain) → intermediates → root (in OS trust store).

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
# Or check expiry:
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
   | openssl x509 -noout -dates -subject -issuer
```

**SNI** (Server Name Indication) lets one IP serve many domains — the client sends the hostname in the ClientHello so the server picks the right cert.

**Cert expiry** is the #1 silent outage. Monitor and auto-renew (cert-manager + Let's Encrypt, ACM, Cloud Certificate Manager).

## HTTP — Methods, Status, Caching

### Methods

| Method | Idempotent? | Safe? | Body? |
|--------|-------------|-------|-------|
| GET | Yes | Yes | No |
| HEAD | Yes | Yes | No |
| POST | No | No | Yes |
| PUT | Yes | No | Yes |
| PATCH | No | No | Yes |
| DELETE | Yes | No | Optional |
| OPTIONS | Yes | Yes | No |

### Status Codes Worth Memorizing

- **2xx:** 200 OK, 201 Created, 204 No Content.
- **3xx:** 301 Moved Permanently, 302/307 Found/Temp Redirect, 304 Not Modified.
- **4xx:** 400 Bad Request, 401 Unauthorized (missing/invalid auth), 403 Forbidden (authed, denied), 404 Not Found, 409 Conflict, 422 Unprocessable, 429 Rate Limited.
- **5xx:** 500 Internal, 502 Bad Gateway (upstream), 503 Service Unavailable, 504 Gateway Timeout.

### Caching Headers

```
Cache-Control: public, max-age=3600, s-maxage=86400
ETag: "abc123"
Last-Modified: Wed, 20 May 2026 12:00:00 GMT
```

CDN, browser, and origin all participate. `Cache-Control: private` for per-user. `no-store` for sensitive responses.

### HTTP/2 and HTTP/3

- HTTP/1.1: text-based, one request per TCP connection (or pipelining, rarely used).
- HTTP/2: binary, multiplexed streams over one TCP connection, header compression. Head-of-line blocking at TCP level.
- HTTP/3 (QUIC): same idea over UDP, no HoL blocking, faster connection setup. Cloudflare/CDNs support widely.

## NAT — Why Your Laptop Doesn't Have a Public IP

**Source NAT (SNAT) / PAT:** many internal IPs share one public IP, distinguished by source port. Router rewrites packets on the way out + maintains a translation table.

**Destination NAT (DNAT):** rewrites destination to send incoming traffic to internal hosts (port forwarding).

In K8s, kube-proxy's iptables mode does DNAT on Service VIP → Pod IP. In a VPC, NAT Gateway does SNAT for private subnets → internet.

## Load Balancing — L4 vs L7

| | L4 | L7 |
|---|---|---|
| What it inspects | TCP/UDP headers (IP, port) | HTTP method, headers, path, body |
| Examples | AWS NLB, HAProxy mode tcp, IPVS | AWS ALB, nginx, Envoy, Traefik |
| Performance | Very fast, low overhead | Slower (parsing) |
| TLS termination | Optional (passthrough) | Almost always at the LB |
| Sticky sessions | By source IP (crude) | By cookie (precise) |
| Path-based routing | No | Yes |
| WebSockets / gRPC | Yes (with sticky for sessions) | Yes (with explicit support) |

### Algorithms

- **Round Robin** — simple, ignores load.
- **Least Connections** — best for variable-duration requests.
- **Weighted** — capacity-aware.
- **IP Hash / consistent hash** — session affinity without cookies.
- **EWMA / least time** — Envoy default, considers latency.

## Common Diagnostic Commands

```bash
# DNS
dig +short example.com                       # A record
dig +trace example.com                       # full resolution path
nslookup example.com 8.8.8.8                 # specific resolver
host -t MX example.com

# Reachability
ping -c 4 example.com                        # ICMP — may be blocked
mtr example.com                              # traceroute + per-hop loss
traceroute -T -p 443 example.com             # via TCP (around ICMP filters)

# Ports
nc -zv example.com 443                       # is the port open?
ss -tnp                                      # established TCP connections (Linux)
ss -tlnp                                     # listening TCP
ss -s                                        # summary

# HTTP
curl -v https://example.com
curl -w "@curl-format.txt" -o /dev/null -s https://example.com   # timing breakdown
curl --resolve example.com:443:1.2.3.4 https://example.com       # test before DNS cutover

# TLS
openssl s_client -connect example.com:443 -servername example.com
nmap --script ssl-enum-ciphers -p 443 example.com
```

`curl-format.txt`:

```
time_namelookup:    %{time_namelookup}\n
time_connect:       %{time_connect}\n
time_appconnect:    %{time_appconnect}\n
time_starttransfer: %{time_starttransfer}\n
time_total:         %{time_total}\n
```

## Interview Questions

**Q: A user reports the site is slow. How do you isolate to network vs app?**
A: `curl -w` for timing breakdown (DNS, TCP connect, TLS, TTFB, total). MTR to find packet loss. Browser DevTools waterfall. Compare same request from another region. Check CDN cache hit rate. If TTFB is high, it's the app/origin; if connect/TLS is high, network or LB.

**Q: Walk through what happens when I type `example.com` in a browser.**
A: (1) Browser checks cache + OS hosts file. (2) DNS resolution: stub → recursive → root → TLD → authoritative → IP. (3) TCP 3-way handshake. (4) TLS handshake (cert exchange, ECDHE key exchange). (5) HTTP request. (6) Server response (possibly through CDN, LB, app). (7) Browser renders.

**Q: A and AAAA vs CNAME?**
A: A → IPv4 address, AAAA → IPv6 address. CNAME → another hostname. CNAMEs can't coexist with other records at the same name and traditionally can't be at the zone apex — use ALIAS/ANAME records (cloud-specific) for apex.

**Q: HTTP/1.1 vs HTTP/2 vs HTTP/3 — what changed?**
A: 1.1: text, one request per connection (or pipelining). 2: binary, multiplexed streams on one TCP connection, header compression. 3 (QUIC): same multiplexing over UDP, no TCP head-of-line blocking, faster handshake. Each step reduces RTT and improves multi-request page loads.

**Q: When would you use L4 vs L7 load balancing?**
A: L4 for TCP/UDP (databases, gRPC streaming, generic TCP) where you need raw performance and not HTTP semantics. L7 for HTTP services — path/host routing, cookie affinity, WAF, request rewrites, TLS termination at the edge.

**Q: How does TLS protect a request?**
A: Server presents cert → client validates against trust store + checks hostname + checks expiry. Then ECDHE-based key exchange establishes a session key — both sides derive the same symmetric key without sending it. All payload encrypted with that key. Forward secrecy means past sessions stay safe even if a long-term key is later compromised.

## Common Pitfalls

- Overlapping CIDRs across VPCs → no peering possible.
- Setting TTL = 86400 on a record you might need to swap → 24h wait for changes.
- One IP behind round-robin DNS to two ports — clients pick weirdly, no failover.
- Counting all 5 reserved IPs in `/24` — usable space is ~251, not 256.
- TLS cert with no SAN, only CN — modern browsers reject.
- Forgetting `--resolve` when testing a new server before DNS — wastes hours.
- Trusting `ping` to confirm reachability — ICMP is often filtered.

## Related

- [01-linux-essentials.md](01-linux-essentials.md)
- [07-kubernetes-networking.md](07-kubernetes-networking.md)
- [23-load-balancers-and-proxies.md](23-load-balancers-and-proxies.md)
- [15-cloud-aws-basics.md](15-cloud-aws-basics.md)
