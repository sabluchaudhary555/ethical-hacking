# Domain, DNS & Subdomain Enumeration

## Topics

### 1. WHOIS Lookup

WHOIS is a query/response protocol used to look up ownership and registration information for a domain name or IP address block. Every domain registered through ICANN-accredited registrars must have WHOIS records containing (unless privacy-protected) the registrant's details, and this is one of the very first passive recon steps performed on any target domain.

Information typically revealed by a WHOIS lookup:
- **Registrant details**: name, organization, email, and phone (often redacted by GDPR/privacy proxy services today, but sometimes still exposed for older or misconfigured domains).
- **Registrar information**: which company the domain was registered through (GoDaddy, Namecheap, etc.).
- **Creation, update, and expiry dates**: useful for understanding how long a domain/organization has existed and whether it's about to expire (potential domain-hijacking opportunity if abandoned).
- **Name servers (NS records)**: which DNS provider is authoritative for the domain — often reveals hosting/CDN provider (Cloudflare, AWS Route 53, etc.).
- **Domain status codes**: flags like `clientTransferProhibited` indicate registrar-level lock settings.

Basic manual lookup with the `whois` command (covered fully under Tools):
```
whois target-site.com
```
```
# Output (example, truncated)
Registrar: NameCheap, Inc.
Creation Date: 2015-03-12
Registry Expiry Date: 2027-03-12
Name Server: ns1.cloudflare.com
Name Server: ns2.cloudflare.com
```
Even this small snippet already tells us the domain is behind Cloudflare, which shapes the rest of the DNS/subdomain recon strategy.

---

### 2. DNS Footprinting

DNS footprinting is the process of gathering information from the Domain Name System infrastructure of a target to understand its network layout, mail servers, subdomains, and hosting relationships. Since DNS is inherently public (anyone can query it), this is a legal and passive recon technique that forms the backbone of most further reconnaissance.

Why DNS matters in recon:
- **Reveals infrastructure**: DNS records point to actual IP addresses, mail servers, and third-party services (CDNs, cloud providers) behind a domain.
- **Uncovers subdomains**: many organizations forget to secure or deprecate old subdomains (`dev.`, `staging.`, `old.`, `test.`), which DNS enumeration can surface.
- **Exposes email infrastructure**: MX and TXT (SPF/DKIM) records reveal the mail provider and can hint at email security posture.
- **Foundation for active scanning**: once IPs are known from DNS resolution, tools like `nmap` can be scoped precisely to in-scope hosts.

The core technique is querying a DNS server for various record types about a domain — which leads directly into understanding DNS Resource Records (next topic) and using tools like `dig`/`nslookup` to query them.

---

### 3. DNS Resource Records

DNS resource records are the individual entries stored in a domain's DNS zone that map names to data (IP addresses, mail servers, text data, etc.). Understanding each record type is essential because each one leaks a different category of information during recon.

Key record types and what they reveal:
- **A record**: maps a hostname to an IPv4 address — the most direct way to find a server's real IP.
- **AAAA record**: same as A, but for IPv6 addresses.
- **CNAME record**: an alias pointing one hostname to another — often reveals use of third-party services (e.g., `shop.target.com` → `shops.myshopify.com`).
- **MX record**: specifies the mail servers responsible for receiving email — reveals the email provider (Google Workspace, Microsoft 365, self-hosted).
- **NS record**: specifies authoritative name servers for the domain — reveals the DNS hosting provider.
- **TXT record**: holds arbitrary text data, commonly used for SPF, DKIM, and domain-verification strings for third-party services (Google, AWS, Facebook) — extremely useful for discovering which external services a company has integrated.
- **SOA record**: "Start of Authority" — contains administrative info about the zone, including the primary name server and a refresh/serial number.
- **PTR record**: used for reverse DNS lookups, mapping an IP address back to a hostname.

Querying a specific record type manually with `dig`:
```
dig target-site.com MX +short
```
```
# Output (example)
10 mail.target-site.com.
20 mail2.target-site.com.
```
This immediately tells us the mail infrastructure setup without touching the live web server at all.

---

### 4. Subdomain Finder Websites

Beyond command-line tools, there is a category of free web-based services purpose-built for discovering subdomains of a target domain by aggregating data from Certificate Transparency logs, historical DNS records, and search engine indexing. These are useful for a quick, no-install, browser-based first pass before running heavier automated tools.

Common subdomain finder websites and what they use as their data source:
- **crt.sh**: queries Certificate Transparency (CT) logs — since every publicly issued SSL/TLS certificate is logged, searching by domain reveals every subdomain that has ever had a certificate issued for it.
- **Rapid7 Sonar / Project Sonar (via securitytrails.com or similar aggregators)**: uses internet-wide scan data to list discovered subdomains.
- **SecurityTrails**: offers historical DNS and subdomain data through a searchable web UI (with an API for automation on paid tiers).
- **VirusTotal**: under the "Relations" tab for a domain, lists observed subdomains alongside malware/reputation data.

Manual usage example (crt.sh, no install needed):
```
https://crt.sh/?q=%25.target-site.com
```
This returns every certificate ever logged for any subdomain of `target-site.com`, which is often the fastest way to find forgotten subdomains like `dev.target-site.com` or `vpn.target-site.com`.

---

## Tools

### 1. whois

**Tool Description & Objective**
`whois` is a command-line client for querying WHOIS servers to retrieve domain and IP registration data. Its objective in recon is to quickly identify the registrant, registrar, creation/expiry dates, and authoritative name servers for a target domain — all from a single, fast, passive query.

**Installation & Setup**
Pre-installed on Kali Linux. On other Debian-based systems:
```
sudo apt update && sudo apt install whois -y
```
Verify installation:
```
whois --version
```

**Core Syntax & Flags**
```
whois [domain or IP]
```
- `-h <server>` — query a specific WHOIS server instead of the default
- `-p <port>` — specify a custom port for the WHOIS query
- `-H` — hide legal disclaimers from the output for cleaner parsing

**Practical Workflows (Use)**
Basic domain lookup:
```
whois target-site.com
```
```
# Output (example, truncated)
Registrar: NameCheap, Inc.
Registrant Organization: REDACTED FOR PRIVACY
Creation Date: 2015-03-12
Name Server: ns1.cloudflare.com
```
IP address lookup (reveals the network block owner/ASN):
```
whois 104.21.14.101
```

**Tool Chaining & Automation**
- Extract the name servers from `whois` output and cross-check them against known CDN/hosting provider name-server patterns (e.g., `*.cloudflare.com`) to instantly know if the target is behind a WAF/CDN.
- Feed discovered registrant organization names into further OSINT (LinkedIn, Google Dorking) to map real-world employees or related domains.
- Script bulk WHOIS lookups across a list of subdomains found via Sublist3r/crt.sh to check which ones are separately registered vs. part of the same parent domain.

**Limitations & Alternatives**
Most registrars now redact personal registrant details by default (GDPR privacy protection), so WHOIS often reveals only registrar/name-server info rather than personal data. Alternatives: web-based WHOIS lookup tools (whois.domaintools.com), RDAP (the newer, more structured replacement protocol for WHOIS).

**Disclaimer**
WHOIS data is public by design, so querying it is legal and passive; however, any further action taken based on registrant information (e.g., contacting them) should stay within the agreed scope of your engagement.

---

### 2. dig / nslookup

**Tool Description & Objective**
`dig` (Domain Information Groper) and `nslookup` are command-line DNS query tools used to directly interrogate DNS servers for specific resource records (A, MX, NS, TXT, etc.) about a domain. Their objective in recon is to map out a target's DNS infrastructure precisely — resolving IPs, mail servers, and third-party service integrations record by record.

**Installation & Setup**
Both are pre-installed on Kali Linux (part of the `dnsutils`/`bind9-dnsutils` package). On other Debian-based systems:
```
sudo apt update && sudo apt install dnsutils -y
```
Verify installation:
```
dig -v
nslookup
```

**Core Syntax & Flags**
```
dig [domain] [record-type] [options]
```
- `+short` — concise output, just the answer
- `@<server>` — query a specific DNS server (e.g., `@8.8.8.8` for Google Public DNS)
- `MX` / `NS` / `TXT` / `A` / `AAAA` / `CNAME` / `SOA` — specify record type to query
- `-x <IP>` — perform a reverse DNS (PTR) lookup

```
nslookup [domain]
nslookup -query=[record-type] [domain]
```

**Practical Workflows (Use)**
Query all common record types quickly with `dig`:
```
dig target-site.com ANY +short
```
Query mail servers specifically:
```
dig target-site.com MX +short
```
```
# Output (example)
10 mail.target-site.com.
```
Reverse lookup an IP to find its associated hostname:
```
dig -x 104.21.14.101 +short
```
Equivalent basic lookup with `nslookup`:
```
nslookup -query=NS target-site.com
```

**Tool Chaining & Automation**
- Script a loop that runs `dig` for A, MX, NS, and TXT records across a large subdomain list generated by Sublist3r, building a full DNS inventory automatically.
- Feed resolved IP addresses from `dig` output directly into `nmap` for scoped port scanning.
- Use TXT record output to detect third-party service verification strings (Google Workspace, AWS SES, Facebook domain verification), revealing what external platforms the organization uses.

**Limitations & Alternatives**
`dig`/`nslookup` only reveal what's published in the target's DNS zone — they can't uncover subdomains that aren't already known or guessed (that requires brute-force/enumeration tools like Sublist3r or Amass). Alternatives: Google Public DNS web interface, `host` command (a simpler, less detailed DNS query tool).

**Disclaimer**
DNS queries are a normal, passive part of internet usage and are legal to perform against any public domain; only the follow-up active steps (e.g., scanning resolved IPs) require explicit authorization.

---

### 3. DNS Dumpster

**Tool Description & Objective**
DNS Dumpster is a free, web-based domain research tool that maps a target's DNS infrastructure visually — showing subdomains, associated IP addresses, MX/NS/TXT records, and even a network diagram — all aggregated from multiple passive data sources without directly touching the target's servers.

**Installation & Setup**
No installation required — used entirely through the browser:
1. Go to `https://dnsdumpster.com`.
2. Enter the target domain and submit.
3. Review the generated table of subdomains/records and the auto-generated network map graphic.

**Core Syntax & Flags**
Not applicable (web-based tool); the only "input" is the domain typed into the search form. An unofficial API/scraper wrapper exists in some third-party Python scripts for automation, but the official service is browser-only.

**Practical Workflows (Use)**
Enter `target-site.com` into the DNS Dumpster search box to receive:
- A table of discovered subdomains with their resolved IP and geolocation
- MX record hosts (mail infrastructure)
- TXT record entries
- A visual network diagram showing relationships between discovered hosts

This is typically run right after WHOIS and before deeper subdomain brute-forcing, to get an immediate visual sense of the target's DNS footprint.

**Tool Chaining & Automation**
- Export the subdomain list from DNS Dumpster and merge it with Sublist3r/Amass output to build a single, deduplicated master subdomain list.
- Cross-reference discovered IPs with Shodan to see what services are running on each host.
- Use the visual network map to prioritize which subdomains look most interesting (e.g., `vpn.`, `admin.`, `dev.`) for deeper manual investigation.

**Limitations & Alternatives**
The free tier has rate limits and doesn't cover every possible subdomain (it relies on passive data sources, so very obscure or brand-new subdomains may be missed). Alternatives: Sublist3r, Amass, crt.sh (Certificate Transparency-based).

**Disclaimer**
DNS Dumpster only aggregates publicly available passive DNS data; it does not authorize any active testing of the hosts it reveals.

---

### 4. Google Public DNS

**Tool Description & Objective**
Google Public DNS is a free, globally distributed DNS resolution service (`8.8.8.8` / `8.8.4.4`) that can be used both as an alternative DNS resolver and, via its web-based lookup tool, as a quick way to query DNS records for a domain without relying on potentially cached/manipulated local resolvers. Its objective in recon is to get fast, reliable, and consistent DNS answers, and to cross-verify results obtained from other resolvers.

**Installation & Setup**
No installation is needed to use it as a resolver — simply point queries at Google's IPs. To set it as your system's default DNS resolver on Kali (temporary, for the current session):
```
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```
For the web-based lookup tool, no setup is required — just visit:
```
https://dns.google
```

**Core Syntax & Flags**
Using it as a specific DNS server target with `dig`:
```
dig target-site.com @8.8.8.8 +short
```
- `@8.8.8.8` — direct the query specifically to Google's resolver instead of the system default
DNS-over-HTTPS (DoH) query via `curl`, useful for querying DNS through firewalls that block traditional port 53:
```
curl -s -H "accept: application/dns-json" "https://dns.google/resolve?name=target-site.com&type=A"
```

**Practical Workflows (Use)**
Compare results from Google Public DNS against the organization's own/local DNS resolver to detect DNS-based filtering, split-horizon DNS setups, or geo-based responses:
```
dig target-site.com @8.8.8.8 +short
dig target-site.com @1.1.1.1 +short
```
```
# Output (example)
104.21.14.101   (from Google 8.8.8.8)
104.21.14.101   (from Cloudflare 1.1.1.1)
```
Matching results across independent resolvers increases confidence that the resolved IP is accurate and not locally spoofed/poisoned.

**Tool Chaining & Automation**
- Use Google Public DNS as a consistent baseline resolver in automated recon scripts, avoiding reliance on potentially cached or tampered local DNS.
- Combine DoH queries with scripts that need to bypass network-level DNS blocking during authorized testing.
- Cross-check MX/TXT record results between Google Public DNS and the domain's authoritative name servers to spot propagation delays or misconfigurations.

**Limitations & Alternatives**
Being a public resolver, it only returns what's published for public resolution — internal/split-horizon DNS records used only on private networks won't be visible. Alternatives: Cloudflare DNS (`1.1.1.1`), OpenDNS, or directly querying the domain's authoritative name servers with `dig @<ns-server>`.

**Disclaimer**
Using a public DNS resolver to query publicly published records is completely passive and legal; it carries no additional risk beyond standard DNS footprinting.

---

### 5. Sublist3r

**Tool Description & Objective**
Sublist3r is a Python-based subdomain enumeration tool that aggregates subdomains for a target domain by querying multiple search engines (Google, Bing, Yahoo, Baidu, Ask) and other public sources (VirusTotal, Netcraft, DNSdumpster, Threat Crowd) in one automated pass. Its objective is to quickly compile a broad list of a target's subdomains for further fingerprinting and testing.

**Installation & Setup**
Pre-installed on many Kali versions; otherwise, install from GitHub:
```
git clone https://github.com/aboul3la/Sublist3r.git
cd Sublist3r
pip install -r requirements.txt --break-system-packages
```
Verify installation:
```
python3 sublist3r.py -h
```

**Core Syntax & Flags**
```
python3 sublist3r.py -d [domain] [options]
```
- `-d <domain>` — target domain to enumerate
- `-b` — enable brute-force enumeration mode using a built-in subdomain wordlist
- `-p <ports>` — port-scan discovered subdomains (e.g., `-p 80,443`)
- `-t <threads>` — number of threads for brute-force mode
- `-o <file>` — save output to a file
- `-v` — verbose mode, shows the engine currently being queried

**Practical Workflows (Use)**
Basic passive enumeration:
```
python3 sublist3r.py -d target-site.com -o subdomains.txt
```
```
# Output (example, truncated)
[-] Enumerating subdomains now for target-site.com
[-] Searching now in Google..
[-] Searching now in VirusTotal..
Total Unique Subdomains Found: 27
dev.target-site.com
mail.target-site.com
staging.target-site.com
```
Combine passive search with brute-force for wider coverage:
```
python3 sublist3r.py -d target-site.com -b -t 40 -o subdomains.txt
```

**Tool Chaining & Automation**
- Pipe the `subdomains.txt` output into `httpx` to filter only the subdomains that resolve and respond over HTTP/HTTPS.
- Feed live subdomains into WhatWeb/Wappalyzer for bulk technology fingerprinting across the entire subdomain list.
- Merge Sublist3r's results with crt.sh and DNS Dumpster output, then deduplicate (`sort -u`) to build the most complete possible subdomain list before further enumeration (e.g., with `amass` for even deeper coverage).

**Limitations & Alternatives**
Sublist3r relies on external search engines and APIs, some of which throttle or block automated queries over time, occasionally causing incomplete results. It's also not actively maintained as heavily as newer tools. Alternatives: Amass (more comprehensive, actively maintained, combines passive + active techniques), Assetfinder (lightweight, fast, similar passive approach), Subfinder (ProjectDiscovery's modern, actively maintained equivalent).

**Disclaimer**
Passive subdomain enumeration through public sources is legal, but the optional brute-force mode (`-b`) actively sends DNS queries to the target's infrastructure and should only be used with explicit authorization.

---

### 6. Amass

**Tool Description & Objective**
Amass (by OWASP) is an advanced attack-surface mapping and subdomain enumeration tool that combines passive data-source aggregation (like Sublist3r) with active techniques such as DNS zone walking, brute-forcing, and certificate transparency analysis, plus the ability to visualize relationships between discovered assets. Its objective is to provide the most thorough, professional-grade subdomain and infrastructure map of a target organization.

**Installation & Setup**
Install via Go (recommended, ensures latest version):
```
sudo apt update && sudo apt install golang-go -y
go install -v github.com/owasp-amass/amass/v4/...@master
export PATH=$PATH:$(go env GOPATH)/bin
```
Or install directly via apt on Kali:
```
sudo apt install amass -y
```
Verify installation:
```
amass -version
```

**Core Syntax & Flags**
```
amass enum [options] -d [domain]
```
- `-d <domain>` — target domain
- `-passive` — passive-only mode (no direct DNS resolution/brute-forcing, purely OSINT-source based)
- `-active` — enables active techniques including zone transfers and certificate checks
- `-brute` — enable brute-force subdomain guessing using a wordlist
- `-o <file>` — save results to a file
- `-src` — show which data source each result came from

**Practical Workflows (Use)**
Passive-only enumeration (safest, fully OSINT-based):
```
amass enum -passive -d target-site.com -o amass-results.txt
```
Full active enumeration with brute-forcing (requires authorization, as it directly queries the target's DNS infrastructure):
```
amass enum -active -brute -d target-site.com -o amass-results.txt
```
```
# Output (example, truncated)
dev.target-site.com
api.target-site.com
old-portal.target-site.com
```

**Tool Chaining & Automation**
- Combine Amass's `-src` output with Sublist3r and crt.sh results to understand which data source is most productive for a given target, refining future recon strategy.
- Feed the final subdomain list into `httpx`/`nmap` for live-host verification and scoped port scanning.
- Use Amass's built-in visualization (`amass viz`) to generate a graph of discovered assets for reporting purposes.

**Limitations & Alternatives**
Amass's active/brute-force modes can be slow and noisy against heavily monitored targets, and the tool has a steeper learning curve than simpler alternatives. Alternatives: Sublist3r (simpler, faster for a quick pass), Subfinder (lightweight, fast, passive-focused).

**Disclaimer**
Passive Amass usage is safe and legal against any public domain; active modes (zone transfers, brute-forcing) send direct queries to the target's DNS servers and must only be run with explicit written authorization.

---

## Cheat Sheet

| Concept | Syntax | Key Point |
|---|---|---|
| WHOIS Lookup | `whois target-site.com` | Reveals registrar, creation/expiry dates, and name servers |
| DNS Footprinting | `dig target-site.com ANY +short` | Passive querying of DNS infrastructure to map hosts and services |
| DNS Resource Records | `dig target-site.com MX +short` | A/AAAA/CNAME/MX/NS/TXT/SOA/PTR each leak a different data category |
| Subdomain Finder Websites | `https://crt.sh/?q=%25.target-site.com` | Uses Certificate Transparency logs to reveal every subdomain with a cert |
| whois (tool) | `whois target-site.com` | CLI client for domain/IP registration data |
| dig / nslookup | `dig target-site.com MX +short` | Direct, precise DNS record queries by type |
| DNS Dumpster | `https://dnsdumpster.com` search box | Visual DNS/subdomain map from passive sources |
| Google Public DNS | `dig target-site.com @8.8.8.8 +short` | Consistent public resolver; also supports DNS-over-HTTPS |
| Sublist3r | `python3 sublist3r.py -d target-site.com -o subdomains.txt` | Aggregates subdomains from search engines and public APIs |
| Amass | `amass enum -passive -d target-site.com -o amass-results.txt` | Advanced passive + active attack-surface mapping |