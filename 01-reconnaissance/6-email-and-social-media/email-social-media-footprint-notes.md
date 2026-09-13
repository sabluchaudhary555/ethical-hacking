# Email & Social Media Intelligence

## Topics

### 1. Email Footprinting

Email footprinting is the process of gathering information about a target organization's email addresses, naming conventions, and mail infrastructure. Email addresses are one of the most valuable recon artifacts because they double as usernames for many corporate services (VPNs, webmail, intranets) and are the primary vector for phishing/social-engineering attacks.

Key information gathered during email footprinting:
- **Email address harvesting**: collecting real employee email addresses from public sources — company websites, PDF/document metadata, press releases, job postings, GitHub commits, and search engine results.
- **Naming convention pattern**: once a few real addresses are found (e.g., `john.smith@target.com`, `jane.doe@target.com`), the underlying pattern (`firstname.lastname@domain`) can be inferred and used to guess additional employee addresses for names found via LinkedIn or company "About Us" pages.
- **Mail server / provider identification**: MX records (covered under DNS Resource Records) reveal whether the organization uses Google Workspace, Microsoft 365, or a self-hosted mail server — this shapes which known phishing/security bypass techniques might be relevant.
- **Breach exposure**: checking whether harvested email addresses have appeared in known public data breaches, which can reveal reused/weak passwords or additional personal data.
- **Email verification**: confirming that a guessed or harvested address is actually active/deliverable without sending a real message (via SMTP handshake tricks or third-party verification APIs).

Manual email harvesting can start with a simple search engine dork:
```
site:target-site.com "@target-site.com"
```
This surfaces publicly indexed pages containing real email addresses at the target domain, which is often enough to establish the organization's naming pattern before turning to automated tools like theHarvester (covered under Tools).

---

### 2. Email Header Analysis

Email header analysis is the technique of examining the hidden metadata attached to an email message to trace its true origin, the path it traveled through mail servers, and whether it shows signs of spoofing or tampering. Every email contains headers that are normally hidden from the average user but are visible via "View Original" / "Show Source" options in most mail clients.

Key header fields and what they reveal:
- **Received**: a stack of entries, each added by a mail server the message passed through, listed in reverse chronological order (bottom-most `Received` entry is the earliest/closest to the original sender). Tracing these reveals the originating mail server's IP address.
- **From / Reply-To**: the displayed sender address — attackers can forge this field, so it should never be trusted alone; it must be cross-checked against the `Received` chain.
- **Return-Path**: the address to which bounce messages are sent — often reveals the true sending mailbox even when `From` is spoofed.
- **Authentication-Results (SPF, DKIM, DMARC)**: shows whether the message passed sender-verification checks. A `fail` or `softfail` result is a strong indicator of spoofing.
- **Message-ID**: a unique identifier usually containing the sending mail server's domain, which can help confirm or contradict the claimed sender.
- **X-Originating-IP / X-Mailer**: non-standard but commonly present headers that can reveal the sender's real IP address or the email client/software used to send the message.

Manual analysis workflow: open the raw email source (in Gmail: "Show original"; in Outlook: "View message source"), then read the `Received` chain from bottom to top to trace the message's path, and check the `Authentication-Results` header to see if SPF/DKIM/DMARC passed. This is the core manual technique that automated header-analysis tools (e.g., Google's MessageHeader tool, MXToolbox) simply visualize more conveniently.

---

### 3. Social Media Intelligence (SOCMINT)

Social Media Intelligence, or SOCMINT, is the discipline of collecting and analyzing publicly available information from social media platforms to build a profile of a target organization or individual. In an ethical hacking/pentesting context, SOCMINT is primarily used during the social-engineering and phishing-preparation phase, since employees frequently overshare organizational details, technology stacks, and personal information on public profiles.

Key information gathered through SOCMINT:
- **Employee identification**: LinkedIn is the primary source for mapping an organization's staff, job titles, and reporting structure — extremely useful for crafting believable spear-phishing pretexts (e.g., impersonating a manager or IT staff member).
- **Technology stack leaks**: employees' LinkedIn skills sections, GitHub profiles, or conference talk bios often reveal exactly which internal tools, frameworks, or security products a company uses.
- **Physical security details**: photos posted by employees (badge photos, office backgrounds, desk setups) can inadvertently reveal building layouts, badge designs, or screen contents useful for physical social engineering.
- **Personal details for password/security-question guessing**: birthdays, pet names, hometowns, and hobbies shared publicly can help guess passwords or answer security questions during account-recovery attacks.
- **Username correlation across platforms**: many people reuse the same username across Twitter/X, Instagram, GitHub, Reddit, etc. — finding one username lets an investigator pull a full cross-platform profile of an individual.
- **Real-time operational intelligence**: employees sometimes post about ongoing projects, office locations, travel plans, or system outages in real time, which can be exploited for timing-based social engineering.

Manual SOCMINT starts with simple platform-native searches (LinkedIn's people search filtered by company name, Twitter/X advanced search operators, Google dorking with `site:linkedin.com "target company"`), and scales up using dedicated cross-platform username/profile correlation tools such as Sherlock and Social Searcher (covered under Tools).

---

## Tools

### 1. theHarvester

**Tool Description & Objective**
theHarvester is a Python-based OSINT tool designed to gather emails, subdomains, IPs, and employee names for a target domain by querying multiple public sources — search engines (Google, Bing, DuckDuckGo), Shodan, Hunter, and other APIs — in a single automated pass. Its objective in email footprinting is to rapidly compile a list of real, harvestable email addresses and associated infrastructure data for a target organization.

**Installation & Setup**
Pre-installed on Kali Linux. On other Debian-based systems, install from GitHub:
```
git clone https://github.com/laramies/theHarvester.git
cd theHarvester
pip install -r requirements.txt --break-system-packages
```
Verify installation:
```
theHarvester -h
```

**Core Syntax & Flags**
```
theHarvester -d [domain] -b [source] [options]
```
- `-d <domain>` — target domain to search
- `-b <source>` — data source to query (e.g., `google`, `bing`, `duckduckgo`, `linkedin`, `all`)
- `-l <limit>` — limit the number of results returned per source
- `-f <file>` — save results to an HTML/XML file
- `-n` — perform DNS resolution on discovered hosts
- `-c` — perform a DNS brute-force on the target domain

**Practical Workflows (Use)**
Basic email/subdomain harvest using multiple sources:
```
theHarvester -d target-site.com -b all -l 200 -f harvest-results.html
```
```
# Output (example, truncated)
[*] Emails found: 14
--------------------
john.smith@target-site.com
jane.doe@target-site.com

[*] Hosts found: 6
--------------------
mail.target-site.com
vpn.target-site.com
```
Focus on a single source (e.g., Google only) for a faster, lighter query:
```
theHarvester -d target-site.com -b google
```

**Tool Chaining & Automation**
- Feed harvested email addresses directly into Have I Been Pwned's API to check for breach exposure.
- Cross-reference discovered employee names/emails with LinkedIn to build a full organizational chart for social-engineering pretext development.
- Combine harvested subdomains with Sublist3r/Amass output to enrich the overall DNS/subdomain master list from earlier modules.

**Limitations & Alternatives**
Result completeness heavily depends on which sources are accessible (some require paid API keys, e.g., Shodan/Hunter), and search engines occasionally rate-limit or CAPTCHA automated queries. Alternatives: Hunter.io (email-specific, higher accuracy verification), simple Google dorking (`site:target.com "@target.com"`) for a lightweight manual alternative.

**Disclaimer**
theHarvester only queries publicly available, passive data sources, making it legal and low-risk to run; however, any subsequent use of harvested emails (e.g., sending test phishing emails) must stay within the explicitly agreed engagement scope.

---

### 2. Hunter.io

**Tool Description & Objective**
Hunter.io is a web-based (and API-accessible) email-finding and verification service that specializes in discovering professional email addresses associated with a domain, along with confidence scores and the underlying naming pattern used by the organization. Its objective is to provide highly accurate, verified email intelligence — often cleaner and more reliable than general-purpose harvesting tools.

**Installation & Setup**
No installation required for the web interface:
1. Go to `https://hunter.io`.
2. Create a free account (free tier includes a limited number of monthly searches).
3. Enter the target domain in the "Domain Search" box.

For API/automated use, generate an API key from the account dashboard, then query it with `curl`:
```
curl "https://api.hunter.io/v2/domain-search?domain=target-site.com&api_key=<your_api_key>"
```

**Core Syntax & Flags**
Not applicable for the web UI; the REST API's main query parameters are:
- `domain` — the target domain to search
- `api_key` — your Hunter.io API key
- `type=personal` / `type=generic` — filter results by personal vs. generic (e.g., `info@`) addresses
- `limit` — number of results to return per request

**Practical Workflows (Use)**
Web UI: enter `target-site.com` into Domain Search to instantly see a list of discovered emails, each with a confidence score and its source (webpage, PDF, etc.), plus the detected organizational naming pattern (e.g., `{first}.{last}@target-site.com`).

API-based lookup for scripting:
```
curl "https://api.hunter.io/v2/domain-search?domain=target-site.com&api_key=<your_api_key>"
```
```
# Output (example, truncated)
{
  "data": {
    "pattern": "{first}.{last}",
    "emails": [
      { "value": "john.smith@target-site.com", "confidence": 92 }
    ]
  }
}
```
Verify whether a specific guessed email address is valid/deliverable:
```
curl "https://api.hunter.io/v2/email-verifier?email=jane.doe@target-site.com&api_key=<your_api_key>"
```

**Tool Chaining & Automation**
- Use Hunter's detected naming pattern to generate a full list of guessed emails from a LinkedIn-sourced employee name list, then verify each with the Email Verifier endpoint.
- Feed confirmed valid addresses into theHarvester's output for a combined, deduplicated master email list.
- Integrate the Hunter API into a recon automation script alongside Have I Been Pwned to simultaneously discover and check breach exposure for each address.

**Limitations & Alternatives**
The free tier has strict monthly search limits, and full historical/bulk lookups require a paid subscription. Alternatives: theHarvester (free, broader but less accurate), Snov.io (similar commercial email-finding service).

**Disclaimer**
Hunter.io only surfaces publicly discoverable professional email data; using its output for any active outreach (test phishing, social engineering) must remain within your authorized engagement scope.

---

### 3. Have I Been Pwned (HIBP)

**Tool Description & Objective**
Have I Been Pwned is a free web service (and API) that lets you check whether an email address or domain has appeared in known, publicly disclosed data breaches. Its objective in recon is to reveal which harvested employee email addresses have been exposed in past breaches — a strong indicator of potential password reuse and a valuable data point for password-spraying risk assessment (informational only, not for actually attempting compromised credentials).

**Installation & Setup**
No installation required for basic single-email checks via the browser:
1. Go to `https://haveibeenpwned.com`.
2. Enter an email address in the search box.

For domain-wide/bulk checks or API automation, an API key is required (paid, low-cost tier):
```
curl -H "hibp-api-key: <your_api_key>" "https://haveibeenpwned.com/api/v3/breachedaccount/john.smith@target-site.com"
```

**Core Syntax & Flags**
Not applicable for the web UI; the API's key elements are:
- `hibp-api-key` header — required for all API v3 requests
- `/breachedaccount/{email}` — endpoint to check breaches for a specific email
- `/breaches` — endpoint to list all breaches HIBP has on record (no API key required for this one)
- `truncateResponse=true` — return only breach names, without full breach details (lighter response)

**Practical Workflows (Use)**
Manual single check via browser: enter `john.smith@target-site.com` into the search box to see a list of breaches that address appeared in (e.g., LinkedIn 2012, Adobe 2013), along with what data was exposed in each (passwords, emails, phone numbers).

API-based check for a harvested email:
```
curl -H "hibp-api-key: <your_api_key>" "https://haveibeenpwned.com/api/v3/breachedaccount/john.smith@target-site.com?truncateResponse=true"
```
```
# Output (example)
[
  { "Name": "LinkedIn" },
  { "Name": "Collection1" }
]
```

**Tool Chaining & Automation**
- Loop through the full list of email addresses harvested by theHarvester/Hunter.io, checking each against HIBP to identify which employees have historically exposed credentials.
- Cross-reference breach names with public breach-data write-ups to understand exactly what data type (passwords, security questions) was exposed in each incident.
- Use aggregate exposure statistics (e.g., "40% of harvested employee emails found in breaches") as a risk-communication data point in a client-facing recon report.

**Limitations & Alternatives**
HIBP only reports whether an email *appeared* in a breach dataset — it does not reveal the actual leaked password, and API access for bulk/automated queries requires a paid subscription. Alternatives: DeHashed (paid, shows actual leaked credential data for authorized use cases), Intelligence X (broader leaked-data search engine).

**Disclaimer**
Checking breach exposure is informational and passive; under no circumstances should any password associated with a breach be used to attempt an actual login without explicit, written authorization — doing so is unauthorized access and illegal in most jurisdictions.

---

### 4. Sherlock

**Tool Description & Objective**
Sherlock is a free, open-source Python tool that searches for a given username across hundreds of social media platforms and websites simultaneously, reporting where an account with that exact username exists. Its objective in SOCMINT is to rapidly build a cross-platform profile of an individual once a single username has been identified (e.g., from a leaked email's local part, a GitHub handle, or a forum post).

**Installation & Setup**
Install from GitHub (requires Python 3):
```
git clone https://github.com/sherlock-project/sherlock.git
cd sherlock
pip install -r requirements.txt --break-system-packages
```
Verify installation:
```
python3 sherlock --help
```

**Core Syntax & Flags**
```
python3 sherlock [username] [options]
```
- `--timeout <seconds>` — set the request timeout per site (useful on slow connections)
- `--print-found` — only display sites where the username was found (hides "not found" results for cleaner output)
- `--csv` — export results to a CSV file
- `--folderoutput <path>` — specify a directory to save output files
- (multiple usernames can be passed at once as separate arguments to check several at once)

**Practical Workflows (Use)**
Basic cross-platform username search:
```
python3 sherlock johnsmith123 --print-found
```
```
# Output (example, truncated)
[+] GitHub: https://github.com/johnsmith123
[+] Twitter: https://twitter.com/johnsmith123
[+] Reddit: https://reddit.com/user/johnsmith123
```
Export results to CSV for reporting:
```
python3 sherlock johnsmith123 --csv --folderoutput ./results
```

**Tool Chaining & Automation**
- Take a username guessed from a harvested email's local part (e.g., `john.smith` from `john.smith@target-site.com`) and run Sherlock to discover the individual's personal social media footprint.
- Feed discovered GitHub profiles into further code/commit-history recon to check for accidentally leaked credentials or internal project names.
- Combine Sherlock results with Social Searcher for a fuller picture — Sherlock confirms *where* an account exists, Social Searcher helps analyze *what* that account has publicly posted.

**Limitations & Alternatives**
Sherlock can produce false positives/negatives since some sites return misleading HTTP status codes for both existing and non-existing profiles, and results only confirm a username exists — not that it belongs to the intended target individual (name/context verification is still needed). Alternatives: WhatsMyName (similar concept, web-based and actively maintained site list), Maltego (visual, link-analysis-based alternative for deeper correlation).

**Disclaimer**
Username searches across public platforms are passive and generally legal, but compiling a personal profile of a real individual still requires careful adherence to privacy expectations and the boundaries of your authorized engagement.

---

### 5. Maltego

**Tool Description & Objective**
Maltego is a graphical link-analysis and OSINT tool that visually maps relationships between people, email addresses, domains, social media accounts, phone numbers, and organizations by pulling data from a wide range of built-in and third-party "transforms" (data-source plugins). Its objective is to turn scattered OSINT data points (from email footprinting, SOCMINT, DNS recon, etc.) into a single, visual relationship graph that reveals connections a text-based tool would miss.

**Installation & Setup**
Pre-installed on Kali Linux (Community Edition). On other systems, download the installer from the official site:
1. Go to `https://www.maltego.com/downloads/`.
2. Download the Community Edition (CE) installer for your OS.
3. Run the installer and register a free Maltego CE account (required to activate the Community Edition transforms).
Verify installation by launching:
```
maltego
```

**Core Syntax & Flags**
Maltego is primarily a GUI application, so there are no traditional CLI flags. The core workflow revolves around:
- **Entities** — the graph nodes representing data types (Domain, Email Address, Person, Phone Number, Social Media Alias).
- **Transforms** — right-click actions run on an entity to pull related data from a connected source (e.g., "To Email Addresses" run on a Domain entity).
- **Machines** — pre-built chains of multiple transforms run automatically in sequence for a common recon workflow (e.g., "Company Stalker").

**Practical Workflows (Use)**
Typical workflow: create a new graph, add a Domain entity for `target-site.com`, then run the built-in "Footprint L1/L2/L3" machine, which automatically chains DNS, WHOIS, email, and related-domain transforms and populates the graph visually.
```
# Example flow (GUI-driven, not command-line)
1. New Graph → Add Entity → Domain → "target-site.com"
2. Right-click entity → Run Transform → "To DNS Name" / "To Email Address"
3. Run Machine → "Footprint L2" for an automated multi-hop recon chain
```
The resulting graph visually links discovered subdomains, email addresses, and associated social media profiles, making it easy to spot which individuals or infrastructure pieces are most central/connected.

**Tool Chaining & Automation**
- Import email addresses harvested by theHarvester/Hunter.io as Email Address entities, then run "To Social Network Profile" transforms to visually connect them to social media accounts.
- Combine with Sherlock's discovered usernames by manually adding them as Alias entities and linking them to the corresponding Person entity for a unified graph.
- Export the final graph as a report-ready image/PDF for client-facing OSINT documentation.

**Limitations & Alternatives**
The free Community Edition has a transform-run limit per day and restricts commercial use; the most powerful transforms (e.g., paid data-broker sources) require a commercial license. Alternatives: SpiderFoot (free, automated, more CLI/API-friendly OSINT aggregator), manual correlation using Sherlock + Google dorking for a no-cost approach.

**Disclaimer**
Maltego aggregates only what its underlying transforms can legally access from public/licensed sources; building a profile of a real person still requires staying within your authorized engagement scope and applicable privacy law.

---

### 6. Social Searcher

**Tool Description & Objective**
Social Searcher is a free, web-based social media monitoring and search tool that lets you search for keywords, usernames, or hashtags across multiple public social platforms (Twitter/X, Instagram, Facebook public posts, YouTube, Reddit, and more) in one place, along with basic sentiment analysis. Its objective in SOCMINT is to quickly see what a target organization or individual is publicly posting/being mentioned about in near real time, without needing separate accounts or API access to each platform.

**Installation & Setup**
No installation required — used entirely through the browser:
1. Go to `https://www.social-searcher.com`.
2. Enter a keyword, username, or company name into the search box.
3. Free tier shows recent results across supported platforms; a paid tier unlocks historical search, alerts, and full analytics/export.

**Core Syntax & Flags**
Not applicable (web-based tool); the main "parameters" are the search box query and platform filter checkboxes (Twitter, Instagram, Facebook, YouTube, etc.) available on the results page.

**Practical Workflows (Use)**
Enter the target company's name or a discovered employee's username into the search box to retrieve recent public posts mentioning it, filterable by platform and sorted by date/popularity — useful for spotting employees discussing internal projects, complaints about IT systems, or upcoming events that could inform a social-engineering pretext.

Setting up a free alert (paid feature on most tiers, but worth noting for reporting) allows ongoing monitoring of a company name or product for newly appearing public mentions.

**Tool Chaining & Automation**
- Cross-reference usernames found via Sherlock with Social Searcher to read the actual public post content associated with each account, rather than just confirming the account exists.
- Feed interesting findings (e.g., an employee mentioning a specific software rollout) into the technology-stack picture built earlier with WhatWeb/Wappalyzer/BuiltWith for a fuller organizational profile.
- Use sentiment/keyword trend data as supporting evidence in a social-engineering risk assessment report.

**Limitations & Alternatives**
The free tier only searches recent posts (no deep historical search) and coverage of some platforms (especially Instagram/Facebook) is limited by those platforms' own API restrictions. Alternatives: native platform search (Twitter/X advanced search operators), TweetDeck (for real-time Twitter/X monitoring), Google Alerts (for broader web-wide keyword monitoring).

**Disclaimer**
Social Searcher only surfaces already-public posts; using discovered content for a social-engineering pretext must remain strictly within your authorized engagement scope and should never involve harassment or unauthorized impersonation.

---

## Cheat Sheet

| Concept | Syntax | Key Point |
|---|---|---|
| Email Footprinting | `site:target-site.com "@target-site.com"` | Harvests real addresses to infer the organization's naming pattern |
| Email Header Analysis | View "Show Original" / "View Source" in mail client | Trace the `Received` chain bottom-up; check SPF/DKIM/DMARC results |
| Social Media Intelligence (SOCMINT) | LinkedIn people search, `site:linkedin.com "target company"` | Maps employees, tech stack leaks, and personal details from public profiles |
| theHarvester | `theHarvester -d target-site.com -b all -l 200 -f harvest-results.html` | Aggregates emails, subdomains, and hosts from multiple OSINT sources |
| Hunter.io | `curl "https://api.hunter.io/v2/domain-search?domain=target-site.com&api_key=<key>"` | High-accuracy email finder with confidence scores and naming-pattern detection |
| Have I Been Pwned | `curl -H "hibp-api-key: <key>" ".../breachedaccount/<email>"` | Checks whether an email has appeared in known public breaches |
| Sherlock | `python3 sherlock johnsmith123 --print-found` | Finds a username across hundreds of social platforms at once |
| Maltego | GUI: Add Domain entity → run "Footprint L2" machine | Visual link-analysis graph connecting emails, domains, and social profiles |
| Social Searcher | `https://www.social-searcher.com` search box | Cross-platform public post search with basic sentiment analysis |