# Website Recon & Mirroring

## Topics

### 1. Website Footprinting

Website footprinting is the process of gathering as much information as possible about a target website without directly attacking it. The objective is to build a profile of the target's technology stack, structure, hosting environment, and hidden entry points before moving to active scanning or exploitation. This is a passive/semi-passive reconnaissance phase, meaning most of the data comes from publicly visible sources: the rendered page, the page source, HTTP headers, cookies, and metadata embedded in files.

Key information typically gathered during website footprinting:
- **Server-side technology**: web server (Apache/Nginx/IIS), backend language (PHP/Node/Python), and CMS (WordPress/Joomla/Drupal) — found in HTTP response headers and page source comments.
- **HTTP headers**: response headers like `Server`, `X-Powered-By`, and `Set-Cookie` reveal software versions and session handling.
- **Page source code**: HTML comments, JavaScript file names, and hidden form fields often leak internal paths, developer notes, or API endpoints.
- **Metadata**: document metadata (author names, software used to generate a file) found in uploaded PDFs, Word docs, or images on the site.
- **Cookies**: session cookie names/structure can hint at the underlying framework (e.g., `PHPSESSID`, `JSESSIONID`, `laravel_session`).
- **Directory structure**: robots.txt, sitemap.xml, and error pages (404/403) can reveal hidden or restricted directories.
- **Copyright & footer info**: often reveals the actual company/developer behind the site, useful for further OSINT.

Basic manual footprinting can be done with just a browser (View Page Source, Inspect Element, Network tab) or with `curl`:

```
curl -I https://target-site.com
```
```
# Output (example)
HTTP/1.1 200 OK
Server: nginx/1.18.0
X-Powered-By: PHP/7.4.3
Set-Cookie: PHPSESSID=abc123; path=/
```

This single command already reveals the web server, backend language, and session cookie type — the foundation for deciding which deeper tools (WhatWeb, Wappalyzer, Nikto, etc.) to run next.

---

### 2. Internet Archive & Wayback Machine

The Wayback Machine (archive.org) is a digital archive of the web that has been periodically crawling and storing snapshots of websites since 1996. In ethical hacking, it is used to view how a target site looked and behaved in the past — this is extremely valuable because old, "forgotten" pages, admin panels, backup files, or comments sometimes remain accessible even after the live site has been updated or secured.

Why this matters for recon:
- **Historical exposure**: a developer comment, exposed API key, or internal endpoint that was removed from the live site may still exist in an archived snapshot.
- **Structure changes over time**: comparing old vs. new versions of a site reveals what was hidden, renamed, or deprecated (old login pages, staging subdomains, retired file paths).
- **Passive and legal**: since you're only viewing archive.org's cached copy, no requests touch the target's live infrastructure, making it a completely passive recon technique.

Manual usage is simply browsing to:
```
https://web.archive.org/web/*/https://target-site.com/*
```
This shows a calendar-style snapshot timeline. You can also query it directly for a full URL list using the CDX API:

```
curl "http://web.archive.org/cdx/search/cdx?url=target-site.com/*&output=text&fl=original&collapse=urlkey"
```
```
# Output (example, truncated)
https://target-site.com/
https://target-site.com/login.php
https://target-site.com/old-admin/
https://target-site.com/backup.zip
```

This raw CDX output is exactly what dedicated tools like `waybackurls` (covered under Tools) automate and filter at scale.

---

## Tools

### 1. wget

**Tool Description & Objective**
`wget` is a free, non-interactive command-line utility for downloading content from the web over HTTP, HTTPS, and FTP. In recon and mirroring, its objective is to create a local, offline copy of a target website's files and directory structure so it can be examined at leisure — searching for hidden files, comments, JS secrets, or old backups — without repeatedly hitting the live server. It ships by default on almost every Linux distribution, including Kali.

**Installation & Setup**
`wget` comes pre-installed on Kali Linux and most Debian-based systems. If missing:
```
sudo apt update && sudo apt install wget -y
```
Verify installation:
```
wget --version
```

**Core Syntax & Flags**
```
wget [options] [URL]
```
- `-r` — recursive download (follow links)
- `-l <depth>` — limit recursion depth (default is 5)
- `-k` — convert links for local/offline viewing
- `-p` — download all page requisites (images, CSS, JS) needed to display the page
- `-np` — "no parent": don't ascend to the parent directory
- `-A <ext>` — accept only specific file extensions
- `-U "<user-agent>"` — spoof the User-Agent string
- `-e robots=off` — ignore robots.txt restrictions
- `--limit-rate=<speed>` — throttle download speed to avoid triggering rate-limits/WAFs

**Practical Workflows (Use)**
Full offline mirror of a site for later inspection:
```
wget -r -k -p -np -e robots=off -U "Mozilla/5.0" https://target-site.com
```
```
# Output (example)
Saving to: 'target-site.com/index.html'
...
Converted links in 42 files in 0.3 seconds.
```
Download only specific file types (useful for hunting exposed backups/configs):
```
wget -r -A "zip,sql,bak,txt" -np https://target-site.com
```

**Tool Chaining & Automation**
- Mirror with `wget`, then grep the local copy for secrets: `grep -r "api_key" target-site.com/`
- Feed the recursively downloaded file list into `gau`/`waybackurls` output to cross-check which historical URLs still exist live.
- Combine with `diff` to compare two mirrors taken at different times, revealing what changed on the target site.

**Limitations & Alternatives**
`wget` cannot render JavaScript, so single-page applications (React/Angular/Vue) that build content client-side will mirror as mostly empty shells. It also has no built-in GUI or link-visualization. Alternatives include HTTrack (better structured mirroring with a browsable local site) and Cyotek WebCopy (Windows GUI alternative).

**Disclaimer**
Only mirror websites you own or have explicit written authorization to test. Aggressive recursive downloading can be mistaken for a denial-of-service attempt and may violate the target's terms of service or local law.

---

### 2. HTTrack

**Tool Description & Objective**
HTTrack Website Copier is a free, open-source tool purpose-built for mirroring entire websites — including HTML, images, and other files — into a local directory while rebuilding the relative link structure so the copy can be browsed offline exactly like the live site. Compared to `wget`, HTTrack is more recon-friendly because it is designed specifically for full-site cloning rather than generic file downloading.

**Installation & Setup**
On Kali/Debian-based systems:
```
sudo apt update && sudo apt install httrack -y
```
Verify installation:
```
httrack --version
```
A GUI front-end (`webhttrack`) is also available:
```
sudo apt install webhttrack -y
```

**Core Syntax & Flags**
```
httrack [URL] -O [output-directory] [options]
```
- `-O <path>` — set output directory for the mirrored site
- `-r<depth>` — set recursion depth
- `-%e<n>` — set number of connections (concurrency)
- `-F "<user-agent>"` — set a custom User-Agent
- `-%P` — enable "private" mode (bypass some restrictions)
- `--ext-depth=<n>` — control how deep to follow external links
- `-c<n>` — number of simultaneous connections

**Practical Workflows (Use)**
Basic full mirror:
```
httrack https://target-site.com -O ./target-mirror
```
```
# Output (example)
Mirroring website...
Done: 1 site, 240 files, 15 links scanned
```
Interactive mode (prompts for URL, output path, and options step by step):
```
httrack
```
GUI-based mirroring for beginners:
```
webhttrack
```

**Tool Chaining & Automation**
- Combine with `grep`/`ripgrep` on the mirrored output to search for exposed credentials, comments, or endpoint names across the entire offline copy at once.
- Schedule periodic HTTrack mirrors with `cron` to track how a target site changes over time (useful for long-term bug bounty recon).
- Feed discovered file paths from the mirror into directory-brute-forcing tools like `ffuf` or `gobuster` to check if similar paths exist that weren't linked.

**Limitations & Alternatives**
HTTrack, like `wget`, cannot execute JavaScript, so modern JS-heavy frameworks mirror poorly. Very large sites can also take a long time and consume significant disk space. Alternatives: `wget` (more scriptable/lightweight), SiteSucker (macOS GUI alternative), Cyotek WebCopy (Windows GUI alternative).

**Disclaimer**
Use HTTrack only against systems you are authorized to test. Mirroring a site without permission can be considered unauthorized access/scraping under computer misuse laws in many jurisdictions.

---

### 3. WhatWeb

**Tool Description & Objective**
WhatWeb is a website fingerprinting tool that identifies what a website is built with — CMS, web server, JavaScript libraries, analytics packages, frameworks, and more — by analyzing HTTP responses, HTML content, and specific signatures ("plugins"). Its objective in recon is to quickly reveal the target's technology stack so testers know which known vulnerabilities or specialized tools (e.g., WPScan for WordPress) are relevant.

**Installation & Setup**
Pre-installed on Kali Linux. On other Debian-based systems:
```
sudo apt update && sudo apt install whatweb -y
```
Verify installation:
```
whatweb --version
```

**Core Syntax & Flags**
```
whatweb [options] [target]
```
- `-v` — verbose output, shows plugin details and match confidence
- `-a <0-4>` — aggression level (0 = passive, 4 = most aggressive/intrusive)
- `--log-json=<file>` — save results in JSON format
- `-i <file>` — read a list of targets from a file (bulk scanning)
- `-U "<user-agent>"` — spoof User-Agent
- `--color=never` — disable colored output (useful for piping/logging)

**Practical Workflows (Use)**
Basic scan:
```
whatweb target-site.com
```
```
# Output (example)
target-site.com [200 OK] Apache[2.4.41], Country[INDIA][IN],
HTTPServer[Apache/2.4.41], PHP[7.4.3], WordPress[5.8],
X-Powered-By[PHP/7.4.3]
```
Aggressive scan with verbose plugin detail, saved as JSON:
```
whatweb -a 3 -v target-site.com --log-json=whatweb-result.json
```

**Tool Chaining & Automation**
- Pipe WhatWeb's JSON output into a script that automatically triggers WPScan if WordPress is detected, or Joomscan if Joomla is detected.
- Run WhatWeb across a bulk subdomain list (`-i subdomains.txt`) generated earlier by Sublist3r to fingerprint an entire organization's web assets in one pass.
- Cross-reference WhatWeb's version findings with CVE databases (e.g., `searchsploit`) to flag outdated, vulnerable software versions.

**Limitations & Alternatives**
Higher aggression levels send more requests and can be noisier/more detectable by WAFs or IDS systems. Fingerprint signatures also need regular updates to stay accurate against new software versions. Alternatives: Wappalyzer (browser-based, similar goal), BuiltWith (cloud-based/commercial, broader database).

**Disclaimer**
Only run WhatWeb, especially at higher aggression levels, against systems you are explicitly authorized to test — aggressive scans can resemble intrusive probing.

---

### 4. Wappalyzer

**Tool Description & Objective**
Wappalyzer is a technology-profiling tool, most commonly used as a browser extension (Chrome/Firefox/Edge), that instantly identifies the technologies powering a website as you browse — CMS, e-commerce platform, JavaScript frameworks, analytics/tracking tools, server software, and more. Its objective is to give a quick, visual, zero-command-line snapshot of a site's stack, making it ideal for fast manual recon alongside a browser session.

**Installation & Setup**
Browser extension install (most common method):
1. Go to the Chrome Web Store / Firefox Add-ons page.
2. Search "Wappalyzer" and click **Add to Chrome** / **Add to Firefox**.
3. Pin the extension icon to the toolbar for quick access.

CLI version (for automation/scripting), via npm:
```
npm install -g wappalyzer
```
Verify installation:
```
wappalyzer --help
```

**Core Syntax & Flags (CLI version)**
```
wappalyzer [URL] [options]
```
- `--pretty` — pretty-print JSON output
- `--recursive` — crawl and analyze linked pages within the same domain
- `--max-depth=<n>` — set crawl depth for recursive mode
- `--user-agent="<string>"` — spoof User-Agent

**Practical Workflows (Use)**
Browser extension: simply click the icon while visiting the target site — it displays a popup with detected technologies grouped by category (CMS, analytics, JS framework, server, etc.), no command needed.

CLI scan:
```
wappalyzer https://target-site.com --pretty
```
```
# Output (example, truncated)
{
  "technologies": [
    { "name": "WordPress", "categories": ["CMS"] },
    { "name": "PHP", "categories": ["Programming languages"] },
    { "name": "Google Analytics", "categories": ["Analytics"] }
  ]
}
```

**Tool Chaining & Automation**
- Use the browser extension for quick manual triage while doing OSINT, then confirm/expand findings with the WhatWeb CLI for a scriptable, loggable result.
- Feed CLI JSON output into a report-generation script to auto-populate a recon report's "Technology Stack" section.
- Combine with BuiltWith for cross-verification, since different fingerprinting engines occasionally catch different technologies.

**Limitations & Alternatives**
The browser extension only fingerprints the page you're currently viewing (no bulk/automated scanning), and detection accuracy depends on how up-to-date its signature database is. Alternatives: WhatWeb (CLI-native, scriptable), BuiltWith (larger commercial database, includes historical technology data).

**Disclaimer**
Wappalyzer's browsing-based fingerprinting is passive and low-risk, but any automated/recursive crawling should still only be done against authorized targets, respecting the site's `robots.txt` and rate limits.

---

### 5. BuiltWith

**Tool Description & Objective**
BuiltWith is a web-based (SaaS) technology profiling service that maintains one of the largest commercial databases of website technology usage. Its objective is similar to WhatWeb/Wappalyzer — identifying CMS, hosting provider, analytics, advertising, and e-commerce technologies — but it adds historical tracking (when a technology was added/removed) and relationship data (other sites using the same tech stack), which is valuable for wider organizational recon (e.g., finding sibling domains of a target company).

**Installation & Setup**
No installation required — BuiltWith is used entirely through the browser:
1. Go to `https://builtwith.com`.
2. Enter the target domain in the search bar.
3. Free tier shows a summary; the paid tiers unlock historical data, lead lists, and bulk lookups via API.

An API is also available for programmatic/bulk use, requiring an API key from a paid plan:
```
curl "https://api.builtwith.com/v21/api.json?KEY=<your_api_key>&LOOKUP=target-site.com"
```

**Core Syntax & Flags**
Being a web service, there are no CLI flags; the main "parameters" are the API query string fields:
- `KEY` — your API key
- `LOOKUP` — the domain to profile
- `NOMETA` — exclude metadata for a lighter response
- `NOLIVE` — return only historical data, skipping a live re-check

**Practical Workflows (Use)**
Manual browser lookup: enter `target-site.com` into the search box and review the categorized technology report (CMS, hosting, email provider, analytics, widgets).

API-based bulk lookup for a list of domains (paid tier), useful when profiling many subdomains discovered earlier via Sublist3r:
```
for d in $(cat subdomains.txt); do
  curl -s "https://api.builtwith.com/v21/api.json?KEY=<your_api_key>&LOOKUP=$d"
done
```

**Tool Chaining & Automation**
- Use BuiltWith to identify a target's email marketing/CMS vendor, then cross-check that vendor's known CVEs.
- Combine with Netcraft to compare hosting/registrar data across both services for consistency.
- Use BuiltWith's "sites using the same technology" feature to discover related organizational assets beyond the primary target scope.

**Limitations & Alternatives**
Full historical data and bulk API access are paywalled; the free tier is limited to a basic snapshot. Alternatives: Wappalyzer and WhatWeb (free, but with smaller/less historical databases).

**Disclaimer**
BuiltWith only surfaces publicly observable technology signals; using its data to plan any further testing still requires explicit authorization on the actual target.

---

### 6. Netcraft (Site Report)

**Tool Description & Objective**
Netcraft is a web-based security and internet research service best known for its "Site Report," which profiles a domain's hosting history, IP address, network block owner, SSL/TLS certificate details, and even the hosting country/provider changes over time. Its objective in recon is to reveal infrastructure-level details — who actually hosts the site, how long it's been there, and what hosting migrations occurred — which is useful for identifying shared hosting environments or CDN configurations.

**Installation & Setup**
No installation required — used through the browser:
1. Go to `https://sitereport.netcraft.com`.
2. Enter the target domain.
3. Review the generated report (hosting history, IP, SSL details, technology).

**Core Syntax & Flags**
Not applicable (web-based tool); the primary "input" is simply the target domain typed into the report form.

**Practical Workflows (Use)**
Enter `target-site.com` into the Site Report form to instantly get:
- Current and historical IP addresses/hosting providers
- SSL/TLS certificate issuer and validity dates
- Network block / ASN (Autonomous System Number) owner
- Site technology summary

This is typically used early in recon, right after WHOIS/DNS footprinting, to understand the hosting landscape before moving to active scanning.

**Tool Chaining & Automation**
- Cross-reference the ASN/IP block from Netcraft with Shodan to discover other services running on the same infrastructure.
- Compare Netcraft's hosting history with Wayback Machine snapshots to correlate infrastructure changes with content changes over time.
- Feed the discovered IP range into `nmap` for a scoped port scan (only within authorized boundaries).

**Limitations & Alternatives**
Netcraft's free report is comprehensive but doesn't offer bulk/API automation without a commercial subscription. Alternatives: `whois`, Shodan (for infrastructure), Censys (certificate/infrastructure search engine).

**Disclaimer**
Netcraft only aggregates publicly available registration and hosting data; it does not grant any right to test the identified infrastructure without separate authorization.

---

### 7. waybackurls

**Tool Description & Objective**
`waybackurls` is a command-line tool (by Tomnomnom) that automates querying the Wayback Machine's CDX API to fetch every URL the Internet Archive has ever recorded for a given domain. Its objective is to turn the manual browsing described in the "Internet Archive & Wayback Machine" topic into a fast, scriptable bulk-URL-extraction step — commonly used to discover old endpoints, parameters, and forgotten files at scale.

**Installation & Setup**
Requires Go to be installed first:
```
sudo apt update && sudo apt install golang-go -y
go install github.com/tomnomnom/waybackurls@latest
```
Ensure Go's bin directory is in your PATH:
```
export PATH=$PATH:$(go env GOPATH)/bin
```
Verify installation:
```
waybackurls -h
```

**Core Syntax & Flags**
```
echo [domain] | waybackurls [options]
```
- `-no-subs` — exclude subdomains, only fetch URLs for the exact domain given
- (Output is plain text, one URL per line, making it easy to pipe into other tools)

**Practical Workflows (Use)**
Fetch all archived URLs for a domain:
```
echo "target-site.com" | waybackurls > all-urls.txt
```
```
# Output (example, truncated)
https://target-site.com/login.php
https://target-site.com/backup.zip
https://target-site.com/api/v1/users?id=1
```
Filter for interesting file types (backups, configs, docs):
```
cat all-urls.txt | grep -E "\.(zip|sql|bak|env|config)$"
```

**Tool Chaining & Automation**
- Pipe the URL list into `httpx` to check which historical URLs are still live: `cat all-urls.txt | httpx -silent`
- Extract URL parameters from the list and feed them into a fuzzing tool like `ffuf` to test for injection points on parameters that existed historically.
- Combine with `gau` (Get All URLs) which pulls from additional sources (Common Crawl, AlienVault OTX) alongside the Wayback Machine for broader coverage.

**Limitations & Alternatives**
Results are only as complete as what the Wayback Machine actually crawled — it may miss recently created or never-crawled pages. Alternatives: `gau` (broader source coverage), manual Wayback Machine browsing (for verifying context around a specific snapshot).

**Disclaimer**
Querying archive.org's public CDX API is passive and doesn't touch the target's live servers, but any active follow-up (e.g., visiting discovered live endpoints) must stay within the scope of your authorization.

---

## Cheat Sheet

| Concept | Syntax | Key Point |
|---|---|---|
| Website Footprinting | `curl -I https://target-site.com` | Reveals server, backend tech, and cookies from HTTP headers |
| Internet Archive & Wayback Machine | `https://web.archive.org/web/*/https://target-site.com/*` | View historical snapshots; may expose removed/forgotten content |
| Wayback CDX API | `curl "http://web.archive.org/cdx/search/cdx?url=target-site.com/*&output=text"` | Raw list of every archived URL for a domain |
| wget | `wget -r -k -p -np -e robots=off https://target-site.com` | Recursive offline mirror; can't render JavaScript |
| HTTrack | `httrack https://target-site.com -O ./target-mirror` | Purpose-built full-site cloning with browsable local copy |
| WhatWeb | `whatweb -a 3 -v target-site.com` | CLI fingerprinting of CMS, server, and frameworks |
| Wappalyzer | Browser extension icon click, or `wappalyzer https://target-site.com --pretty` | Instant visual + scriptable tech stack detection |
| BuiltWith | `https://builtwith.com` search box | Commercial-grade tech profiling with historical data |
| Netcraft Site Report | `https://sitereport.netcraft.com` search box | Hosting history, IP, ASN, and SSL certificate details |
| waybackurls | `echo "target-site.com" \| waybackurls` | Bulk-fetch every archived URL for a domain via CDX API |