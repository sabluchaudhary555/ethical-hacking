# Shodan Search Engine

## Topics

### 1. Introduction to Shodan

Shodan is a search engine that indexes internet-connected devices rather than web pages. Instead of crawling website content like Google, Shodan continuously scans the entire IPv4 address space (and parts of IPv6) on thousands of ports, collects the "banner" each service returns on connection, and makes that banner data searchable. This makes it one of the most important reconnaissance tools in ethical hacking, since it reveals exactly what's publicly exposed on the internet — web servers, databases, webcams, industrial control systems (ICS/SCADA), routers, IoT devices, and more — often including software versions, default configurations, and even open/unauthenticated services.

How Shodan works internally:
- **Banner grabbing**: Shodan's crawlers connect to a huge range of ports across the internet and record the raw response ("banner") a service sends back — for example, an HTTP server's response headers, an SSH server's version string, or an FTP server's welcome message.
- **Continuous re-scanning**: unlike a one-time scan, Shodan re-crawls the internet regularly, so search results include a "Last Update" timestamp showing how recent the data is.
- **Metadata enrichment**: alongside the raw banner, Shodan attaches geolocation, ASN/organization, reverse DNS hostname, and (when applicable) SSL certificate details, screenshots (for RDP/VNC), and vulnerability (CVE) matches.
- **No active exploitation**: Shodan only performs what's called "opportunistic" or "passive" collection — it connects and records what a service says, it does not attempt to log in, exploit, or brute-force anything.

Why this matters for recon:
- Instantly reveals an organization's exposed infrastructure without sending a single packet to the target yourself (fully passive from your side, since you're only querying Shodan's existing database).
- Surfaces forgotten or shadow-IT devices (an old webcam, an exposed database, a misconfigured IoT device) that internal teams may not even know are internet-facing.
- Lets you pivot from a known IP/domain/organization to discover everything else they have exposed.

Manual first step — searching by organization name directly in the search bar:
```
org:"Target Organization Inc"
```
This alone often reveals dozens of exposed hosts belonging to the target, before narrowing further with the specific filters covered below.

---

## Tools

### 1. Shodan (Web Interface)

**Tool Description & Objective**
The Shodan website (shodan.io) is the primary way most people interact with the search engine — a search bar plus filter system, combined with a world map view, "Facets" (aggregate statistics), and per-result detail pages showing full banners, vulnerabilities, and (where available) screenshots. Its objective is to let a tester quickly search, filter, and visually explore exposed devices without writing any code.

**Installation & Setup**
No installation required — used entirely through the browser:
1. Go to `https://www.shodan.io`.
2. Create a free account (a free account unlocks basic filters and a limited number of results per search; a paid membership removes most result caps and unlocks premium filters like `vuln` and `screenshot`).
3. Log in and use the search bar at the top of the page.

**Core Syntax & Flags**
```
[filter]:[value] [filter]:[value] [free-text terms]
```
- Multiple filters combined in one search are treated as AND (all conditions must match).
- Free-text terms (no filter prefix) search across the banner data itself.
- Quotes (`"..."`) group multi-word values, e.g., `org:"Target Organization Inc"`.
- A minus sign (`-`) excludes a term or filter value, e.g., `-title:"login"`.

**Practical Workflows (Use)**
Basic free-text search for a known banner string:
```
apache
```
Filtered search for a specific organization's exposed RDP servers:
```
org:"Target Organization Inc" port:3389
```
```
# Output (example, on results page)
203.0.113.10   Target Organization Inc, India   Port: 3389   RDP
203.0.113.44   Target Organization Inc, India   Port: 3389   RDP
```
Using the map view (toggle at the top of results) to visually see the geographic spread of matching hosts, and the "Facets" panel on the results page to see aggregate breakdowns (e.g., top countries, top ports, top organizations) for the current query.

**Tool Chaining & Automation**
- Pivot from an IP address found via traceroute/WHOIS (Network Footprinting module) directly into Shodan's `ip:` filter to see every open port/service on that host.
- Export search results (CSV/JSON, available on paid plans) and feed them into `nmap`/further active scanning of only the confirmed live, in-scope hosts.
- Use the "Vulnerabilities" tab on a result's detail page (when `vuln` data is present) to cross-reference exposed CVEs against `searchsploit` for known exploits.

**Limitations & Alternatives**
Free accounts have a strict cap on visible results per search and cannot use premium filters (`vuln`, `screenshot`, `org` on some plans); data can also be several days to weeks old depending on how frequently Shodan last scanned a given host/port. Alternatives: Censys (similar internet-wide scan search engine, different data-collection methodology), ZoomEye (Chinese equivalent, useful for different geographic coverage), FOFA (another similar search engine).

**Disclaimer**
Searching Shodan itself is completely passive and legal — you're only querying Shodan's existing database, not scanning the target yourself. However, any follow-up action against a discovered host (connecting to it, testing credentials, exploiting a service) requires explicit authorization from the host's owner.

---

### 2. Shodan CLI (Command-Line Interface)

**Tool Description & Objective**
The Shodan CLI is an official Python-based command-line client that lets you run Shodan searches, download results, and query specific hosts directly from the terminal — useful for scripting, automation, and integrating Shodan data into larger recon workflows without using the browser.

**Installation & Setup**
Install via pip:
```
pip install shodan --break-system-packages
```
Initialize the CLI with your API key (found on your Shodan account page under "My Account"):
```
shodan init <YOUR_API_KEY>
```
Verify installation:
```
shodan version
```

**Core Syntax & Flags**
```
shodan [command] [options]
```
- `shodan search "<query>"` — run a search query and print matching results
- `shodan count "<query>"` — return only the total number of matching results (doesn't consume query credits the way `search` does)
- `shodan host <IP>` — get full details for a specific IP address
- `shodan download <filename> "<query>"` — download raw search results to a local file for offline processing
- `shodan parse <filename>` — parse a downloaded results file into readable output
- `shodan stats "<query>"` — show aggregate facet statistics for a query (e.g., top ports, top countries)
- `--fields <field1,field2>` — specify which fields to display in search output
- `--limit <n>` — limit the number of results returned

**Practical Workflows (Use)**
Search for a target organization's exposed hosts:
```
shodan search "org:\"Target Organization Inc\""
```
```
# Output (example, truncated)
203.0.113.10   3389  Target Organization Inc  India
203.0.113.44   22    Target Organization Inc  India
```
Get the total count of matching results without listing them (useful for quick scoping):
```
shodan count "org:\"Target Organization Inc\" port:3389"
```
```
# Output (example)
7
```
Get full detail on a specific host:
```
shodan host 203.0.113.10
```
Download results for offline analysis:
```
shodan download target-results "org:\"Target Organization Inc\""
shodan parse --fields ip_str,port,org target-results.json.gz
```

**Tool Chaining & Automation**
- Script a loop that runs `shodan host <IP>` across a list of IPs discovered from WHOIS/traceroute/DNS enumeration, building a consolidated exposure report automatically.
- Pipe `shodan search` output (with `--fields ip_str`) directly into `nmap`/`httpx` for a follow-up active scan of only the confirmed hosts.
- Combine `shodan stats` with a broad organization query to quickly see which ports/services are most commonly exposed across the entire target's infrastructure, prioritizing where to look deeper.

**Limitations & Alternatives**
CLI usage consumes the same API query credits as the website, and free-tier accounts have a limited monthly credit allowance, so large/bulk automated searches can exhaust it quickly. Alternatives: the Shodan Python library (`import shodan` for full programmatic control inside custom scripts), the web interface (for one-off manual exploration).

**Disclaimer**
CLI-based querying is exactly as passive/legal as web searches — you're only pulling data from Shodan's database. Any automated follow-up action against discovered hosts still requires explicit authorization.

---

## Search Filters & Dorks (Complete Reference)

This section lists every major Shodan search filter/"code," its syntax, what it does, and a working example — combine multiple filters in one query for precise results.

### Location Filters

**`country`** — filter results to a specific two-letter country code.
```
country:IN
```
```
# Output (example)
Returns all indexed devices located in India.
```

**`city`** — filter by a specific city name.
```
city:"Mumbai"
```

**`geo`** — filter by latitude/longitude and radius (in kilometers).
```
geo:19.0760,72.8777,10
```
```
# Output (example)
Returns devices within a 10 km radius of the given coordinates (Mumbai).
```

### Network & Host Filters

**`net`** — filter by an IP address or CIDR range.
```
net:203.0.113.0/24
```

**`ip`** — filter by a single exact IP address (equivalent to `shodan host <IP>` on the CLI).
```
ip:203.0.113.10
```

**`hostname`** — filter by a full or partial hostname (matches reverse DNS/PTR data).
```
hostname:"target-site.com"
```

**`org`** — filter by the organization name registered against the IP block (from WHOIS/ASN data).
```
org:"Target Organization Inc"
```

**`isp`** — filter by the Internet Service Provider name.
```
isp:"Reliance Jio"
```

**`asn`** — filter by Autonomous System Number.
```
asn:AS15169
```

### Port & Protocol Filters

**`port`** — filter by a specific open port number.
```
port:22
```
```
# Output (example)
Returns all indexed hosts with an open SSH port (22).
```

**`os`** — filter by detected operating system (when Shodan can fingerprint it, e.g., via SMB/RDP banners).
```
os:"Windows Server 2019"
```

### Web / HTTP Filters

**`http.title`** — filter by the target page's HTML `<title>` tag content.
```
http.title:"Dashboard"
```

**`http.status`** — filter by HTTP response status code.
```
http.status:200
```

**`html`** — free-text search within a page's raw HTML content.
```
html:"Index of /"
```

**`http.component`** — filter by a detected web technology/component (e.g., CMS, framework).
```
http.component:"WordPress"
```

**`http.favicon.hash`** — filter by a hashed favicon icon, useful for finding every instance of a specific admin panel/product that reuses the same default favicon.
```
http.favicon.hash:-335242539
```

### SSL / TLS Filters

**`ssl`** — filter results that have SSL/TLS data present.
```
ssl:true
```

**`ssl.cert.subject.cn`** — filter by the Common Name (CN) field of an SSL certificate.
```
ssl.cert.subject.cn:"target-site.com"
```

**`ssl.cert.issuer.cn`** — filter by the certificate's issuing authority.
```
ssl.cert.issuer.cn:"Let's Encrypt"
```

**`ssl.version`** — filter by the TLS/SSL protocol version in use, useful for finding hosts still running outdated, insecure versions.
```
ssl.version:sslv3
```

### Product & Service Filters

**`product`** — filter by the specific software/service product name Shodan has fingerprinted.
```
product:"Apache httpd"
```

**`version`** — filter by a specific software version (usually combined with `product`).
```
product:"Apache httpd" version:"2.4.41"
```

**`category`** — filter by high-level device category on Shodan's newer categorization system (e.g., `ics` for industrial control systems).
```
category:ics
```

**`device`** — filter by a detected device type (e.g., webcam, router, printer).
```
device:"webcam"
```

### Vulnerability Filters (Premium)

**`vuln`** — filter for hosts matching a specific known CVE identifier (requires a paid Shodan membership).
```
vuln:CVE-2021-44228
```
```
# Output (example)
Returns hosts Shodan has flagged as vulnerable to Log4Shell (CVE-2021-44228).
```

**`has_vuln`** — filter for any host that has at least one known vulnerability flagged, regardless of which CVE.
```
has_vuln:true
```

### Screenshot & Visual Filters (Premium)

**`has_screenshot`** — filter for results that include a captured screenshot (commonly available for RDP and VNC services).
```
has_screenshot:true port:3389
```

### Time-Based Filters

**`before`** / **`after`** — filter results by when Shodan last scanned/updated the record (date format: DD/MM/YYYY).
```
after:01/01/2026 before:31/12/2026
```

### Tag & Miscellaneous Filters

**`tag`** — filter by Shodan's internal tag classification (e.g., `self-signed`, `starttls`, `cloud`, `iot`, `database`, `medical`, `industrial`, `honeypot`).
```
tag:database
```

**`hash`** — filter by the hash of a banner's response body, useful for finding every host serving an identical, potentially copy-pasted default page.
```
hash:1234567890
```

---

## Practical Dork Examples (Combined Queries)

The real power of Shodan comes from combining multiple filters into a single precise query. These example "dorks" illustrate common patterns:

Finding exposed MongoDB databases with no authentication:
```
product:"MongoDB" port:27017 -authentication
```

Finding exposed webcams by a specific manufacturer:
```
device:"webcam" org:"Target Organization Inc"
```

Finding default/unconfigured login pages for a common admin panel:
```
http.title:"Login" http.component:"Grafana"
```

Finding hosts vulnerable to a specific, high-profile CVE within a target's IP range:
```
net:203.0.113.0/24 vuln:CVE-2021-44228
```

Finding exposed RDP servers with screenshots available, for a specific organization:
```
org:"Target Organization Inc" port:3389 has_screenshot:true
```

Finding servers running an outdated, unsupported SSL/TLS version within a target's network:
```
net:203.0.113.0/24 ssl.version:sslv3
```

---

## Tool Chaining & Automation (Cross-Module)

- Feed an organization's CIDR block, discovered earlier via RIR WHOIS (Network-Level Footprinting module), directly into Shodan's `net:` filter to find every exposed service across the entire range in one query.
- Cross-reference `ssl.cert.subject.cn` results with subdomains found via Sublist3r/Amass/crt.sh (Domain, DNS & Subdomain Enumeration module) to confirm which discovered subdomains actually have a live, indexed host behind them.
- Use `hostname:` searches combined with the target's root domain to catch subdomains Shodan has independently resolved via reverse DNS, even ones your own enumeration tools may have missed.
- Combine `vuln:` filter results with `searchsploit` to check whether a public exploit exists for a specific CVE Shodan has flagged on a target host.

---

## Limitations & Alternatives (Overall)

Shodan's data is only as fresh as its last scan of a given host/port — a service that changed or was patched since the last crawl may not be reflected yet, so results should always be treated as "as of last seen," not real-time truth. Premium filters (`vuln`, `screenshot`, unlimited results) require a paid membership, and the free tier's result cap can hide the full picture for a large target. Alternatives: Censys (different scan methodology and data freshness, sometimes catches what Shodan misses), ZoomEye (stronger coverage in some Asian regions), FOFA (similar concept, different filter syntax), BinaryEdge (another internet-wide scan data provider).

---

## Disclaimer

Shodan itself is a completely passive, legal reconnaissance tool — searching it only queries data Shodan has already collected and never sends any traffic to the target from your machine. However, discovering an exposed service on Shodan does not grant permission to connect to, log into, or interact with that service in any way; any such follow-up action requires explicit, written authorization from the system's owner and, without it, may constitute unauthorized access under computer misuse laws in most jurisdictions.

---

## Cheat Sheet

| Concept / Filter | Syntax | Key Point |
|---|---|---|
| Introduction to Shodan | `org:"Target Organization Inc"` | Search engine for internet-connected device banners, not web pages |
| Shodan Web Interface | `org:"Target Organization Inc" port:3389` | Browser-based search, map view, and per-host detail pages |
| Shodan CLI | `shodan search "org:\"Target Organization Inc\""` | Scriptable terminal access to the same search/data |
| country | `country:IN` | Filter by two-letter country code |
| city | `city:"Mumbai"` | Filter by city name |
| geo | `geo:19.0760,72.8777,10` | Filter by lat/long + radius in km |
| net | `net:203.0.113.0/24` | Filter by IP or CIDR range |
| ip | `ip:203.0.113.10` | Filter by exact IP address |
| hostname | `hostname:"target-site.com"` | Filter by reverse DNS hostname |
| org | `org:"Target Organization Inc"` | Filter by WHOIS-registered organization |
| isp | `isp:"Reliance Jio"` | Filter by ISP name |
| asn | `asn:AS15169` | Filter by Autonomous System Number |
| port | `port:22` | Filter by open port number |
| os | `os:"Windows Server 2019"` | Filter by fingerprinted operating system |
| http.title | `http.title:"Dashboard"` | Filter by page `<title>` content |
| http.status | `http.status:200` | Filter by HTTP response status code |
| html | `html:"Index of /"` | Free-text search inside raw HTML |
| http.component | `http.component:"WordPress"` | Filter by detected web technology |
| http.favicon.hash | `http.favicon.hash:-335242539` | Find hosts sharing an identical favicon |
| ssl | `ssl:true` | Filter for hosts with SSL/TLS data present |
| ssl.cert.subject.cn | `ssl.cert.subject.cn:"target-site.com"` | Filter by certificate Common Name |
| ssl.cert.issuer.cn | `ssl.cert.issuer.cn:"Let's Encrypt"` | Filter by certificate issuer |
| ssl.version | `ssl.version:sslv3` | Filter by outdated/insecure TLS versions |
| product | `product:"Apache httpd"` | Filter by fingerprinted software product |
| version | `product:"Apache httpd" version:"2.4.41"` | Filter by specific software version |
| category | `category:ics` | Filter by high-level device category |
| device | `device:"webcam"` | Filter by detected device type |
| vuln (premium) | `vuln:CVE-2021-44228` | Filter for hosts flagged with a specific CVE |
| has_vuln (premium) | `has_vuln:true` | Filter for any host with a known flagged vulnerability |
| has_screenshot (premium) | `has_screenshot:true port:3389` | Filter for results with a captured screenshot |
| before / after | `after:01/01/2026 before:31/12/2026` | Filter by last-scan date range |
| tag | `tag:database` | Filter by Shodan's internal classification tag |
| hash | `hash:1234567890` | Filter by banner response-body hash |