# Maltego OSINT Tool

## Topics

### 1. Introduction to Maltego

Maltego is a graphical link-analysis and OSINT platform used to visually map relationships between people, organizations, domains, IP addresses, email addresses, phone numbers, social media accounts, and infrastructure. Unlike command-line OSINT tools that output flat lists of results, Maltego's core value is turning scattered data points into an interactive graph, where every discovered piece of information becomes a node ("Entity") connected by lines to the entities it relates to — making hidden connections between seemingly unrelated data points visually obvious.

How Maltego's model works:
- **Entities**: the building blocks of a Maltego graph — typed objects representing a real-world thing (a Domain, an Email Address, a Person, a Phone Number, an IP Address, a Location, a Social Network Profile, etc.). Each entity type has its own icon and set of properties.
- **Transforms**: the actions you run on an entity to discover related entities. A Transform is essentially a query sent to a data source (a search engine, a DNS server, a paid API, a breach database) that returns new entities to add to the graph, automatically linked back to the entity you ran it from. For example, running "To Email Address" on a Domain entity returns every email address Maltego's connected sources can find for that domain.
- **Transform Hub**: Maltego's built-in marketplace/plugin system for installing additional Transform packs — some free (built into every Maltego installation), others requiring a subscription or a third-party API key (Shodan, HaveIBeenPwned, VirusTotal, and many more all have their own Transform packs).
- **Machines**: pre-built sequences that automatically run multiple Transforms one after another in a defined order, saving you from manually running each Transform yourself. The built-in "Company Stalker" Machine, for instance, automatically chains together email-harvesting, domain, and document-metadata Transforms starting from a single company domain.
- **Graphs**: the visual canvas where entities and their relationships are displayed and can be rearranged, filtered, and analyzed (e.g., using the built-in centrality/importance algorithms to spot which entity has the most connections).

Why this matters for recon:
- A text-based tool tells you "these 40 emails exist"; Maltego shows you which of those emails cluster around the same person, the same subdomain, or the same leaked document — surfacing patterns a spreadsheet would hide.
- It consolidates dozens of individual OSINT tools/APIs (DNS lookups, WHOIS, breach checks, social media searches, document metadata extraction) into one graph-based workflow instead of switching between many separate tools.
- The resulting graph itself is a deliverable — it can be exported directly into a client-facing recon report as a visual asset.

---

### 2. Maltego Editions & Licensing

Maltego is distributed in several editions with different capabilities, pricing, and use cases — understanding which one you're using matters because it determines which Transforms and Machines are available and how many results a single Transform run can return.

- **Maltego Community Edition (CE)**: free, requires registering an account to activate. Limited to a smaller set of results per Transform run (results are capped, e.g., 12 per Transform) and restricted from certain commercial-use scenarios. This is the edition most learners and Kali Linux users start with.
- **Maltego Classic**: a paid single-user license removing the CE result caps, aimed at individual professional analysts.
- **Maltego XL**: built for handling much larger graphs (tens of thousands of entities) with performance optimizations for enterprise-scale investigations.
- **Maltego Enterprise / Maltego for Organizations**: adds team collaboration features, centralized Transform management, and integration with an organization's internal data sources (private Transforms querying internal databases).

For learning and most ethical-hacking coursework, the Community Edition is sufficient to understand the full Entity/Transform/Machine workflow — the paid editions mainly remove result caps and add collaboration/scale features rather than fundamentally different functionality.

---

## Tools

### 1. Maltego (Desktop Application)

**Tool Description & Objective**
The Maltego desktop client is the core application where graphs are built, Transforms are run, and Machines are executed. Its objective is to provide the visual workspace and Transform-execution engine that ties together all of Maltego's OSINT data sources into a single interactive investigation environment.

**Installation & Setup**
Pre-installed on Kali Linux (Community Edition). On other systems, download the installer directly:
1. Go to `https://www.maltego.com/downloads/`.
2. Choose the installer for your OS (Windows, macOS, or Linux `.deb`/`.rpm`).
3. Run the installer.
4. On first launch, register a free Maltego account (required to activate CE) and complete the configuration wizard, which installs the standard free Transform packs.

Verify installation by launching the application:
```
maltego
```
On Kali, if it's not already installed:
```
sudo apt update && sudo apt install maltego -y
```

**Core Syntax & Flags**
Maltego is a GUI application, so there is no traditional command-line syntax for day-to-day use. The essential interface actions that replace "syntax" are:
- **New Graph** — start a blank investigation canvas
- **Drag an Entity from the palette** onto the graph (e.g., Domain, Person, Email Address, Phone Number)
- **Double-click an entity** to edit its value (e.g., type in the target domain name)
- **Right-click an entity → Run Transform** — opens the list of available Transforms for that entity type
- **Right-click an entity → Run Machine** — executes a pre-built multi-step Transform chain
- **View → Entity List / Detail View** — switch between the graph view and a tabular list of all entities in the graph

**Practical Workflows (Use)**
Basic domain investigation:
```
1. New Graph → drag a "Domain" entity onto the canvas
2. Double-click it, type "target-site.com"
3. Right-click → Run Transform → "To DNS Name – NS (name server)"
4. Right-click the domain again → Run Transform → "To Email Address [SEA]"
```
```
# Output (example, on the graph canvas)
target-site.com  →  ns1.cloudflare.com
target-site.com  →  ns2.cloudflare.com
target-site.com  →  john.smith@target-site.com
```
Running a built-in Machine for automated multi-step recon:
```
1. Right-click the Domain entity → Run Machine → "Footprint L1"
2. Maltego automatically runs a sequence of DNS, WHOIS, and related-domain Transforms
3. Review the populated graph once the Machine finishes
```

**Tool Chaining & Automation**
- Import a list of emails harvested by theHarvester/Hunter.io as Email Address entities (via CSV import), then run "To Social Network Profile" Transforms across all of them at once.
- Combine with Sherlock's discovered usernames by manually adding them as Alias entities and linking them to a Person entity for a unified graph.
- Export the finished graph as a PDF/image for inclusion in a client-facing OSINT report.

**Limitations & Alternatives**
The Community Edition's per-Transform result cap can hide the full picture on a data-rich target, and the most powerful commercial Transforms (paid data-broker sources) require a paid subscription. Alternatives: SpiderFoot (free, automated, more CLI/API-friendly OSINT aggregator with a similar "connect data sources into a graph" philosophy), manual correlation using individual tools (Sherlock, theHarvester, Shodan) for a no-cost but more manual approach.

**Disclaimer**
Maltego only aggregates what its underlying Transforms can legally access from public or properly licensed data sources; building a profile of a real person or organization still requires staying within your authorized engagement scope and applicable privacy law.

---

### 2. Maltego Transform Hub

**Tool Description & Objective**
The Transform Hub is Maltego's in-app marketplace for discovering and installing additional Transform packs beyond the default set — ranging from free community-maintained packs to official integrations with commercial threat-intel and OSINT platforms. Its objective is to let you extend Maltego's data-source coverage (adding Shodan, VirusTotal, HaveIBeenPwned, Have I Been Pwned, social media platforms, and more) without leaving the application.

**Installation & Setup**
The Transform Hub is built into the desktop client — no separate installation is needed:
1. Open Maltego and go to the **Transform Hub** tab (usually a tab alongside "New Graph" on the start screen).
2. Browse or search for a specific Transform pack (e.g., "Shodan", "VirusTotal", "HaveIBeenPwned").
3. Click **Install** on the desired pack.
4. For packs requiring a third-party API key (most premium sources), enter your API key in the pack's configuration screen after installation.

**Core Syntax & Flags**
Not applicable — the Transform Hub is entirely GUI-driven. The only "input" required per pack is your API key/credentials for that specific third-party service, entered once during setup.

**Practical Workflows (Use)**
Installing and using the Shodan Transform pack:
```
1. Transform Hub → search "Shodan" → Install
2. Enter your Shodan API key in the pack settings
3. On an IP Address entity in your graph, right-click → Run Transform → "Shodan – To Ports [Shodan]"
```
```
# Output (example, on the graph canvas)
203.0.113.10  →  Port 22 (SSH)
203.0.113.10  →  Port 443 (HTTPS)
```
Installing HaveIBeenPwned to check breach exposure directly on Email Address entities in the graph:
```
1. Transform Hub → search "Have I Been Pwned" → Install
2. Enter your HIBP API key
3. Right-click an Email Address entity → Run Transform → "To Breaches [HaveIBeenPwned]"
```

**Tool Chaining & Automation**
- Combine the Shodan Transform pack with domain/subdomain entities already in the graph to automatically pivot from "which subdomains exist" to "what's exposed on each one," all inside the same visual graph.
- Install the VirusTotal pack to automatically check discovered IPs/domains/file hashes against VirusTotal's reputation data without leaving Maltego.
- Chain a HaveIBeenPwned Transform run immediately after a theHarvester-style email-harvesting Transform to combine discovery and breach-checking in one connected workflow.

**Limitations & Alternatives**
Most of the highest-value Transform packs (Shodan, HaveIBeenPwned, premium threat-intel feeds) require you to already hold a paid account/API key with that third-party service — the Transform Hub itself doesn't grant free access to those platforms. Alternatives: manually running the equivalent standalone tool (e.g., the Shodan CLI) and importing results into Maltego by hand via CSV.

**Disclaimer**
Each installed Transform pack is only as legal/passive as the underlying data source it queries — running a Transform against a paid threat-intel API is exactly as authorized as using that API directly, and any active follow-up on discovered hosts still requires separate authorization.

---

## Entity Reference (Core Entity Types)

Entities are the typed nodes you place on a Maltego graph. These are the most commonly used built-in entity types:

**Domain** — represents a registered domain name; the typical starting point for organizational recon.
```
Value: target-site.com
```

**DNS Name** — represents a specific hostname/subdomain, distinct from the root Domain entity.
```
Value: mail.target-site.com
```

**IP Address (IPv4/IPv6)** — represents a specific IP address, the pivot point for infrastructure/Shodan-style Transforms.
```
Value: 203.0.113.10
```

**Netblock** — represents a CIDR range/IP block, useful when pivoting from an ASN or WHOIS lookup.
```
Value: 203.0.113.0/24
```

**AS (Autonomous System)** — represents an ASN, useful for discovering every netblock announced by an organization.
```
Value: AS15169
```

**Person** — represents a named individual, the central entity type for building an SOCMINT profile.
```
Value: John Smith
```

**Email Address** — represents a specific email address, one of the most connected entity types (links to Person, Domain, breach data, and social profiles).
```
Value: john.smith@target-site.com
```

**Phone Number** — represents a phone number, useful for pivoting into caller-ID/carrier Transforms.
```
Value: +912223334444
```

**Alias** — represents a username/handle, used to link an individual across multiple social platforms.
```
Value: johnsmith123
```

**Social Network Profile / Twitter Affiliation / Facebook Affiliation** — platform-specific entities representing a specific social media account.
```
Value: twitter.com/johnsmith123
```

**Location / GPS Coordinates** — represents a physical location or specific latitude/longitude, useful when combined with EXIF or WHOIS data.
```
Value: 28.6139, 77.2090
```

**Document / File** — represents a specific document (PDF, DOCX), useful for metadata-extraction Transforms that reveal the document's author and software used.
```
Value: annual-report.pdf
```

**Company / Organization** — represents a named organization, often the very first entity dropped onto a new graph.
```
Value: Target Organization Inc
```

**Hash (MD5/SHA)** — represents a file hash, useful for pivoting into malware/threat-intel Transform packs (e.g., VirusTotal).
```
Value: 5d41402abc4b2a76b9719d911017c592
```

---

## Transform Reference (Common Transform Categories with Examples)

Transforms are grouped by what data source they query and what kind of new entity they produce. These are the most frequently used categories:

**DNS Transforms** — resolve a Domain/DNS Name entity into related DNS records.
```
Right-click Domain → "To DNS Name – NS (name server)"
Right-click Domain → "To DNS Name – MX (mail server)"
```

**WHOIS Transforms** — pull registrant/registrar information for a Domain or IP entity.
```
Right-click Domain → "To Whois Details [IBM WHOIS]"
```

**Email-Harvesting Transforms** — discover email addresses associated with a Domain or Person entity, typically pulling from search engines.
```
Right-click Domain → "To Email Address [SEA]"
```

**Document/Metadata Transforms** — search for public documents on a domain and extract embedded author/software metadata.
```
Right-click Domain → "To Files (documents)"
Right-click Document → "To Metadata [Extract Metadata]"
```

**Social Network Transforms** — pivot from an Email Address, Alias, or Person entity to associated social media profiles.
```
Right-click Email Address → "To Social Network Profile [SNA]"
```

**Infrastructure Transforms (paid packs, e.g., Shodan)** — discover open ports/services for an IP Address entity.
```
Right-click IP Address → "Shodan – To Ports [Shodan]"
```

**Breach-Data Transforms (paid packs, e.g., HaveIBeenPwned)** — check an Email Address entity against known breach datasets.
```
Right-click Email Address → "To Breaches [HaveIBeenPwned]"
```

**Geolocation Transforms** — resolve an IP Address entity to an approximate physical Location entity.
```
Right-click IP Address → "To Location [Geolocation]"
```

---

## Machine Reference (Built-In Machines)

Machines chain multiple Transforms together automatically. The most commonly used built-in Machines (available in the free CE Transform packs):

**Footprint L1** — a light, fast reconnaissance pass on a Domain entity, gathering basic DNS, WHOIS, and related-domain data.
```
Right-click Domain → Run Machine → "Footprint L1"
```

**Footprint L2** — a deeper reconnaissance pass than L1, chaining additional DNS/subdomain/email Transforms for a fuller picture.
```
Right-click Domain → Run Machine → "Footprint L2"
```

**Footprint L3** — the most exhaustive built-in footprinting Machine, running the widest possible set of chained Transforms on a Domain entity (takes noticeably longer to complete).
```
Right-click Domain → Run Machine → "Footprint L3"
```

**Company Stalker** — starts from a Domain entity and automatically chains email-harvesting and document-metadata Transforms to build a picture of an organization's employees and leaked document authorship data.
```
Right-click Domain → Run Machine → "Company Stalker"
```

**Twitter Digger / Twitter Monitor** — chains Transforms starting from a Twitter/X-related entity to map an account's connections and recent activity (availability depends on current platform API access).
```
Right-click Twitter Affiliation entity → Run Machine → "Twitter Digger X"
```

---

## Tool Chaining & Automation (Cross-Module)

- Feed a CIDR block discovered via RIR WHOIS (Network-Level Footprinting module) into Maltego as a Netblock entity, then run infrastructure Transforms (Shodan pack) to visually map every exposed service across the range.
- Import subdomains discovered by Sublist3r/Amass/crt.sh (Domain, DNS & Subdomain Enumeration module) as DNS Name entities via CSV import, then run "To IP Address" Transforms to see which ones actually resolve to live hosts.
- Add usernames found via Sherlock as Alias entities and link them manually to a Person entity to combine Sherlock's raw cross-platform hits with Maltego's visual relationship mapping.
- Run the HaveIBeenPwned Transform pack directly on Email Address entities harvested by theHarvester/Hunter.io, keeping the entire discovery-to-breach-check pipeline inside one graph.

---

## Limitations & Alternatives (Overall)

The free Community Edition's per-Transform result cap (commonly 12 results per run) means a data-rich target's full picture may require repeated Transform runs or a paid edition to see completely; the most valuable Transform packs also depend on holding separate paid API keys with the underlying third-party services (Shodan, HaveIBeenPwned, VirusTotal), so Maltego itself doesn't grant free access to that data. Graphs can also become visually cluttered and hard to read once they grow past a few hundred entities without careful layout/filtering. Alternatives: SpiderFoot (free, automated, more scriptable/CLI-friendly with a similar data-source-aggregation philosophy), manual correlation using individual standalone tools (Sherlock, theHarvester, Shodan CLI, WHOIS) when a no-cost, fully scriptable approach is preferred over Maltego's visual/GUI workflow.

---

## Disclaimer

Maltego only aggregates data that its Transforms can legally access from public sources or properly licensed/paid third-party APIs — it does not grant any special access beyond what those underlying sources already provide, and running a Transform is exactly as passive or active as the data source it queries. Building a relationship graph of a real organization or individual must remain strictly within your authorized engagement scope, and any active follow-up on infrastructure or accounts discovered through the graph requires separate, explicit authorization.

---

## Cheat Sheet

| Concept / Item | Syntax / Action | Key Point |
|---|---|---|
| Introduction to Maltego | Entities → Transforms → Machines | Turns scattered OSINT data into a visual relationship graph |
| Maltego Editions | CE (free) vs Classic/XL/Enterprise (paid) | CE caps results per Transform; paid tiers remove caps and add scale/collaboration |
| Maltego Desktop App | `maltego` (launch) | Core GUI workspace for building graphs and running Transforms |
| Transform Hub | Transform Hub tab → search pack → Install | In-app marketplace for adding Shodan, VirusTotal, HIBP, and more |
| Domain entity | Drag "Domain" → type `target-site.com` | Starting point for most organizational recon graphs |
| Email Address entity | Right-click → "To Email Address [SEA]" | Harvests emails associated with a Domain/Person |
| IP Address entity | Right-click → "Shodan – To Ports [Shodan]" | Pivots into exposed-service discovery via Shodan pack |
| Netblock / AS entity | Right-click AS → "To Netblock" | Maps an organization's full announced IP ranges |
| Document/Metadata Transform | Right-click Document → "To Metadata [Extract Metadata]" | Extracts author/software info from public files |
| Social Network Transform | Right-click Email → "To Social Network Profile [SNA]" | Links an email/alias to discovered social accounts |
| Breach Transform (HIBP pack) | Right-click Email → "To Breaches [HaveIBeenPwned]" | Checks breach exposure directly inside the graph |
| Footprint L1/L2/L3 Machine | Right-click Domain → Run Machine → "Footprint L2" | Automated multi-step DNS/WHOIS/email recon chain |
| Company Stalker Machine | Right-click Domain → Run Machine → "Company Stalker" | Chains email-harvesting + document-metadata Transforms |