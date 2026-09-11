# Network-Level Footprinting

## Topics

### 1. Network Footprinting

Network footprinting is the process of gathering information about a target organization's network infrastructure — IP address ranges, live hosts, network topology, firewalls, and routing paths — rather than focusing on a single website. While DNS/domain footprinting reveals *which* hostnames and IPs belong to a target, network footprinting goes one layer deeper: it maps *how* those hosts are connected, *where* they physically/logically sit on the internet, and *what* devices exist along the path between the tester and the target.

Key objectives and information gathered during network footprinting:
- **Live host discovery**: identifying which IP addresses within a known range actually have active machines/services (as opposed to unused address space).
- **Network topology**: understanding the path packets take to reach the target — how many hops, which routers/firewalls sit in between, and whether load balancers or CDNs are involved.
- **IP address ranges owned by the organization**: a company rarely owns just one IP; footprinting aims to discover the full CIDR block(s) allocated to them.
- **Perimeter devices**: identifying firewalls, VPN gateways, and edge routers that filter or shape traffic before it reaches internal hosts.
- **ISP and hosting relationships**: revealing whether infrastructure is self-hosted, cloud-hosted (AWS/Azure/GCP), or behind a CDN/WAF (Cloudflare, Akamai).

This information feeds directly into later phases: once live hosts and IP ranges are known, active scanning tools (`nmap`) can be scoped precisely, and once network topology is understood, testers know where firewalls or filtering devices might block certain probes.

---

### 2. Traceroute — Introduction & Working

Traceroute is a network diagnostic technique used to trace the path that packets take from a source machine to a destination host, revealing every intermediate router ("hop") along the way. In recon, it is used to map network topology and identify perimeter devices such as firewalls or load balancers sitting in front of a target.

How traceroute works internally:
- It sends a series of packets (ICMP Echo Requests on Windows `tracert`, or UDP/ICMP packets on Linux `traceroute`) toward the destination, each with a progressively increasing **TTL (Time To Live)** value, starting at 1.
- Each router that forwards a packet decrements its TTL by 1. When a packet's TTL reaches 0, the router discards it and sends back an **ICMP "Time Exceeded"** message to the sender, revealing that router's IP address.
- By sending packets with TTL = 1, then TTL = 2, then TTL = 3, and so on, traceroute forces each successive router along the path to "reveal itself" one hop at a time, until the packet finally reaches the destination (which responds normally instead of with a Time Exceeded message).
- The round-trip time for each hop is also measured, giving a sense of latency at each point along the path.

This TTL-expiry mechanic is the single core idea behind every traceroute-based tool — understanding it explains why traceroute results can differ across networks, why certain hops show `* * *` (no response, often due to firewalls silently dropping the probe), and why UDP-based and ICMP-based traceroute can yield different results against the same target.

---

### 3. Traceroute Analysis

Traceroute analysis is the process of interpreting a traceroute's hop-by-hop output to extract meaningful reconnaissance value, rather than just running the command and reading raw IPs. A raw traceroute is only useful once analyzed for patterns.

What to look for during analysis:
- **Hops that time out (`* * *`)**: often indicates a firewall or router configured to silently drop ICMP/TTL-expired packets rather than responding — this can mark the edge of a filtered network perimeter.
- **Sudden latency jumps**: a large jump in round-trip time between two consecutive hops can indicate a long-distance link (e.g., crossing from a local ISP to an international backbone, or entering a cloud provider's network).
- **Repeated/duplicate IPs or private IP ranges appearing mid-path**: can reveal internal load-balancing or NAT'd infrastructure that wasn't expected to be visible.
- **The last few hops before the destination**: often reveal the actual edge router or firewall directly protecting the target, useful for understanding what stands between the internet and the target's internal network.
- **Reverse DNS (PTR) of each hop IP**: hostnames along the path (e.g., `edge-fw1.isp-name.net`) can reveal the ISP or hosting provider operating that hop.
- **Comparing traceroutes from multiple vantage points**: running traceroute from different networks/locations (or using online looking-glass servers) can reveal asymmetric routing or geographically distributed infrastructure (CDNs).

Combining traceroute analysis with WHOIS/ASN lookups on the IPs of each hop (especially the last few before the target) often reveals exactly which hosting provider or ISP the organization's edge network belongs to.

---

### 4. Major IP Block

An IP block (or CIDR block) is a contiguous range of IP addresses allocated to an organization by a Regional Internet Registry (RIR). Understanding how IP blocks are structured and allocated is essential for network footprinting because a target rarely has a single IP — it typically owns or is allocated a full range, and discovering that full range dramatically expands the attack surface in scope.

Key concepts:
- **CIDR notation**: IP blocks are expressed as `<network-address>/<prefix-length>` (e.g., `203.0.113.0/24`), where the prefix length defines how many addresses fall in the block (a `/24` = 256 addresses).
- **ASN (Autonomous System Number)**: a unique number assigned to a network or group of networks operating under a single routing policy — useful for identifying all IP blocks announced/owned by a specific organization on the internet.
- **Regional Internet Registries (RIRs)**: the five RIRs — ARIN (North America), RIPE NCC (Europe/Middle East), APNIC (Asia-Pacific), LACNIC (Latin America), and AFRINIC (Africa) — each maintain public WHOIS databases mapping IP blocks to the organizations they're allocated to.
- **BGP (Border Gateway Protocol) announcements**: the actual real-world "who is routing this IP block right now" data, viewable through tools like `bgp.he.net`, which can differ slightly from static RIR allocation records if a block has been leased or reassigned.

Manual lookup example using an RIR's WHOIS database (covered further under Tools):
```
whois -h whois.arin.net 203.0.113.0
```
```
# Output (example, truncated)
NetRange:       203.0.113.0 - 203.0.113.255
CIDR:           203.0.113.0/24
OrgName:        Target Organization Inc.
```
This single query instantly reveals the full block owned by the organization — meaning every IP in that range is potentially in scope for further, authorized recon.

---

## Tools

### 1. traceroute / tracert

**Tool Description & Objective**
`traceroute` (Linux/macOS) and `tracert` (Windows) are command-line utilities that implement the TTL-expiry technique described above to map the network path between the source machine and a target host. Their objective in recon is to reveal intermediate routers, firewalls, and general network topology leading up to the target.

**Installation & Setup**
Pre-installed on Kali Linux. On other Debian-based systems, if missing:
```
sudo apt update && sudo apt install traceroute -y
```
Verify installation:
```
traceroute --version
```
On Windows, `tracert` ships built-in with no installation required.

**Core Syntax & Flags**
```
traceroute [options] [destination]
```
- `-I` — use ICMP ECHO for probes instead of UDP (behaves more like Windows `tracert`)
- `-T` — use TCP SYN for probes, useful for bypassing firewalls that block ICMP/UDP
- `-p <port>` — specify a destination port (useful with `-T` to target an open service port, e.g., 80/443)
- `-m <max-ttl>` — set the maximum number of hops to probe
- `-n` — disable reverse DNS resolution of hop IPs (faster output)
- `-w <seconds>` — set the wait time for a response before marking a hop as timed out

Windows equivalent:
```
tracert [destination]
```

**Practical Workflows (Use)**
Basic trace:
```
traceroute target-site.com
```
```
# Output (example, truncated)
 1  192.168.1.1  1.203 ms
 2  10.10.0.1    5.432 ms
 3  * * *
 4  203.0.113.1  28.912 ms
```
TCP-based trace on port 443 to bypass ICMP-blocking firewalls:
```
traceroute -T -p 443 target-site.com
```

**Tool Chaining & Automation**
- Run WHOIS/ASN lookups on the last few resolved hop IPs before the target to identify the exact hosting provider or ISP protecting the destination.
- Combine with `mtr` for a continuously updating, statistically richer version of the same path analysis.
- Script traceroutes from multiple external vantage points (VPS in different regions) to detect CDN-based geographic routing differences.

**Limitations & Alternatives**
Many networks now filter ICMP/UDP traceroute probes, causing large stretches of `* * *` timeouts that reveal little; TCP-based tracing (`-T`) often works better against such targets. Alternatives: `mtr` (combines ping + traceroute with live statistics), online looking-glass servers (run traceroute from a remote vantage point).

**Disclaimer**
Traceroute is a standard, passive diagnostic technique and is legal to run against any publicly reachable host; however, running it repeatedly at high volume against a target could resemble probing activity and should stay within your authorized scope.

---

### 2. mtr (My Traceroute)

**Tool Description & Objective**
`mtr` combines the functionality of `ping` and `traceroute` into a single, continuously updating tool that shows live statistics (packet loss %, average/min/max latency) for every hop along the path to a destination. Its objective in recon is to provide a much richer, statistically reliable view of network topology and reliability than a single static traceroute snapshot.

**Installation & Setup**
Pre-installed on many Kali versions; otherwise:
```
sudo apt update && sudo apt install mtr -y
```
Verify installation:
```
mtr --version
```

**Core Syntax & Flags**
```
mtr [options] [destination]
```
- `-r` — report mode: run for a fixed number of cycles and print a summary report instead of the live interactive display
- `-c <count>` — number of ping cycles to send per hop (used with `-r`)
- `-n` — disable reverse DNS resolution for faster output
- `-T` — use TCP packets instead of ICMP (useful for bypassing ICMP-blocking firewalls)
- `-P <port>` — specify a destination port (used with `-T`)
- `--report-wide` — wider report format showing full hostnames without truncation

**Practical Workflows (Use)**
Live interactive view:
```
mtr target-site.com
```
Generate a static report (useful for saving to a file / including in a recon report):
```
mtr -r -c 10 -n target-site.com > mtr-report.txt
```
```
# Output (example, truncated)
Host                  Loss%   Avg   Best  Wrst
1. 192.168.1.1         0.0%   1.2   0.9   2.1
2. 10.10.0.1           0.0%   5.6   4.8   7.3
3. 203.0.113.1         0.0%  28.9  27.4  31.2
```

**Tool Chaining & Automation**
- Run `mtr -r` reports at different times of day and compare, to distinguish genuine network issues from transient congestion.
- Combine with `traceroute -T -p 443` results to cross-verify hops when ICMP is filtered.
- Use packet-loss statistics per hop to identify exactly which network segment (ISP vs. target's own network) is responsible for latency/reliability issues, useful context for network-level reporting.

**Limitations & Alternatives**
Like traceroute, `mtr` can still be blocked or show incomplete hops if intermediate routers filter its probes; running it for a long duration also generates more traffic than a single traceroute, which may be more noticeable. Alternatives: `traceroute` (simpler, one-shot), online looking-glass servers (for third-party vantage points).

**Disclaimer**
`mtr` sends repeated probes over time, which is more traffic than a single traceroute; only run it against systems you're authorized to test, and avoid excessively long/high-frequency runs against production infrastructure.

---

### 3. netdiscover

**Tool Description & Objective**
`netdiscover` is a network address discovery tool primarily used for identifying live hosts on a local network segment via ARP (Address Resolution Protocol) requests. Its objective in recon/pentesting is to quickly enumerate active devices (IP and MAC address) on a LAN, typically during internal network assessments or once initial access to a network segment has been obtained.

**Installation & Setup**
Pre-installed on Kali Linux. On other Debian-based systems:
```
sudo apt update && sudo apt install netdiscover -y
```
Verify installation:
```
netdiscover --help
```

**Core Syntax & Flags**
```
netdiscover [options]
```
- `-r <range>` — specify a target IP range/CIDR to scan (e.g., `-r 192.168.1.0/24`)
- `-i <interface>` — specify the network interface to use for scanning
- `-p` — passive mode: only listen for ARP traffic instead of actively sending requests (stealthier, but slower)
- `-c <count>` — number of ARP requests to send per host
- `-s <ms>` — sleep time (in milliseconds) between ARP requests, useful for throttling
- `-L <file>` — load a custom list of MAC address vendor prefixes for device fingerprinting

**Practical Workflows (Use)**
Active scan of a local subnet:
```
sudo netdiscover -r 192.168.1.0/24
```
```
# Output (example, truncated)
IP              MAC Address        Count  Vendor
192.168.1.1     aa:bb:cc:dd:ee:01    3     TP-Link
192.168.1.15    aa:bb:cc:dd:ee:2a    2     Dell Inc.
192.168.1.42    aa:bb:cc:dd:ee:7f    1     Raspberry Pi Foundation
```
Passive/stealth mode (only listens, doesn't send probes):
```
sudo netdiscover -p -i eth0
```

**Tool Chaining & Automation**
- Feed the list of discovered live IPs directly into `nmap` for detailed port/service scanning of each host.
- Use the MAC vendor information to quickly identify device types (routers, printers, IoT devices) worth prioritizing during an internal assessment.
- Combine passive mode output with Wireshark captures for deeper ARP traffic analysis on a shared network segment.

**Limitations & Alternatives**
`netdiscover` only works on the local broadcast domain (it relies on ARP, which doesn't route across subnets/the internet), so it's exclusively useful for local network/internal recon, not for footprinting remote/internet-facing targets. Alternatives: `arp-scan` (similar ARP-based tool with more output formatting options), `nmap -sn` (ping sweep, works both locally and for routed ranges).

**Disclaimer**
`netdiscover` actively sends ARP requests across a network by default, which is detectable by network monitoring tools; only run it on networks you own or have explicit authorization to assess.

---

### 4. nmap (Host Discovery / Ping Sweep)

**Tool Description & Objective**
While `nmap` is best known as a full port-scanning and service-enumeration tool, its host-discovery mode ("ping sweep") is a core part of network-level footprinting: it identifies which IP addresses within a range are actually live/reachable before any port scanning begins, saving significant time and reducing unnecessary traffic to dead addresses.

**Installation & Setup**
Pre-installed on Kali Linux. On other Debian-based systems:
```
sudo apt update && sudo apt install nmap -y
```
Verify installation:
```
nmap --version
```

**Core Syntax & Flags (host discovery mode)**
```
nmap -sn [target/range] [options]
```
- `-sn` — "ping scan": host discovery only, no port scan performed
- `-PE` — use ICMP Echo Request for discovery
- `-PS <ports>` — use TCP SYN probes to specific ports for discovery (useful when ICMP is blocked)
- `-PA <ports>` — use TCP ACK probes for discovery
- `-PU <ports>` — use UDP probes for discovery
- `-n` — disable reverse DNS resolution (faster scan)
- `-oN <file>` — save output in normal (readable) format

**Practical Workflows (Use)**
Basic ping sweep across a discovered CIDR block:
```
nmap -sn 203.0.113.0/24
```
```
# Output (example, truncated)
Nmap scan report for 203.0.113.10
Host is up (0.031s latency).
Nmap scan report for 203.0.113.25
Host is up (0.028s latency).
Nmap done: 256 IP addresses (2 hosts up) scanned in 4.12 seconds
```
TCP SYN-based discovery for networks that block ICMP:
```
nmap -sn -PS80,443 203.0.113.0/24
```

**Tool Chaining & Automation**
- Feed the CIDR block discovered from WHOIS/ASN lookup (Major IP Block topic) directly into `nmap -sn` to identify exactly which hosts within that block are alive.
- Pipe the list of "up" hosts into a full `nmap -sV -sC` scan for detailed service/version enumeration only on confirmed live targets.
- Combine with `netdiscover` results when working across both local and routed network segments during a mixed internal/external assessment.

**Limitations & Alternatives**
Many hosts and firewalls block ICMP Echo Requests by default, so a plain `-sn` scan can under-report live hosts; using TCP/UDP probe variants (`-PS`/`-PA`/`-PU`) usually gives more accurate results. Alternatives: `netdiscover`/`arp-scan` (local network only), Masscan (much faster for very large IP ranges, less accurate).

**Disclaimer**
Host discovery and scanning must only be performed against IP ranges you own or have explicit written authorization to test — scanning ranges outside your authorized scope, even just for host discovery, can be considered unauthorized access.

---

### 5. RIR WHOIS (ARIN / RIPE / APNIC)

**Tool Description & Objective**
The five Regional Internet Registries (ARIN, RIPE NCC, APNIC, LACNIC, AFRINIC) each operate public WHOIS databases that map IP address blocks and Autonomous System Numbers (ASNs) to the organizations they've been allocated to. Querying these directly (rather than a generic domain WHOIS) is the authoritative way to discover the full IP range(s) owned by a target organization.

**Installation & Setup**
No separate installation is needed — the same `whois` client covered earlier can query these registries directly by specifying the appropriate WHOIS server. Ensure `whois` is installed:
```
sudo apt update && sudo apt install whois -y
```
Web-based lookup alternative requires no setup, just a browser:
```
https://whois.arin.net
https://apps.db.ripe.net
```

**Core Syntax & Flags**
```
whois -h [rir-whois-server] [IP or ASN]
```
- `-h whois.arin.net` — query ARIN (North America) directly
- `-h whois.ripe.net` — query RIPE NCC (Europe/Middle East) directly
- `-h whois.apnic.net` — query APNIC (Asia-Pacific) directly
- Querying an `AS<number>` (e.g., `AS15169`) instead of an IP returns information about the Autonomous System itself, including all announced IP blocks

**Practical Workflows (Use)**
Look up which organization owns a specific IP block:
```
whois -h whois.arin.net 203.0.113.0
```
```
# Output (example, truncated)
NetRange:       203.0.113.0 - 203.0.113.255
CIDR:           203.0.113.0/24
OrgName:        Target Organization Inc.
```
Look up all IP blocks announced under a specific ASN:
```
whois -h whois.radb.net -- '-i origin AS15169'
```

**Tool Chaining & Automation**
- Once an organization's ASN is identified, use it to pull every CIDR block they announce, then feed the full list into `nmap -sn` for host discovery across all of it.
- Cross-reference RIR data with `bgp.he.net` (Hurricane Electric's BGP toolkit) to confirm real-time routing announcements match the static RIR allocation records.
- Combine with Netcraft/BuiltWith hosting data to distinguish which blocks are self-hosted infrastructure vs. cloud-provider address space merely leased by the target.

**Limitations & Alternatives**
RIR WHOIS records can be outdated if a block has been reassigned/leased without an update, and IPv6 allocations sometimes require slightly different query syntax. Alternatives: `bgp.he.net` (live BGP routing data), IPinfo.io / ipwhois (web-based bulk IP-to-organization lookup services).

**Disclaimer**
RIR WHOIS data is public registry information and querying it is completely legal and passive; it only identifies ownership — any further active testing of the discovered range still requires separate, explicit authorization.

---

## Cheat Sheet

| Concept | Syntax | Key Point |
|---|---|---|
| Network Footprinting | — | Maps live hosts, IP ranges, and topology beyond a single website |
| Traceroute — Intro & Working | `traceroute target-site.com` | Uses increasing TTL values to force each hop to reveal itself |
| Traceroute Analysis | Inspect `* * *` gaps, latency jumps, last hops before target | Reveals firewalls, ISP transitions, and edge routers |
| Major IP Block | `whois -h whois.arin.net 203.0.113.0` | CIDR blocks are allocated to orgs by RIRs (ARIN/RIPE/APNIC/LACNIC/AFRINIC) |
| traceroute / tracert | `traceroute -T -p 443 target-site.com` | TCP-based tracing bypasses ICMP-blocking firewalls |
| mtr | `mtr -r -c 10 -n target-site.com` | Combines ping + traceroute with live loss/latency stats |
| netdiscover | `sudo netdiscover -r 192.168.1.0/24` | ARP-based live host discovery, local network only |
| nmap (host discovery) | `nmap -sn 203.0.113.0/24` | Ping sweep to find live hosts before port scanning |
| RIR WHOIS | `whois -h whois.arin.net 203.0.113.0` | Authoritative source for IP block/ASN ownership |