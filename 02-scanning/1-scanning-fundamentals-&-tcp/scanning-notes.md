# Scanning Fundamentals & TCP/IP Basics

> Only scan systems you own or have written permission to test. Every packet a scan sends can be logged and traced back to you.

---

# Part A: Topics

## 1. What Is Scanning

Scanning is the **active** phase that follows reconnaissance. Instead of reading public information about a target, you send packets to it and study the replies to learn what is actually running.

**Syntax (generic):** `scanner [scan type] [options] <target>`

- **Goal:** turn the target list from recon (domains, IP ranges) into a map of live hosts, open ports, services, OS details, and weaknesses.
- **What it finds:**

| Finding | Question it answers |
|---|---|
| Live hosts | Which IPs in the range are up? |
| Open ports | Which doors are listening? |
| Services and versions | What software is behind each port? |
| Operating system | Windows, Linux, router, printer? |
| Vulnerabilities | Which known flaws apply? |
| Topology | How are hosts and devices connected? |

- **Why it is detectable:** the target sees real traffic. Firewalls, IDS/IPS, and server logs can all record it. Passive recon cannot be seen this way.
- **Edge case:** a scan that finds nothing does not mean nothing is there. Firewalls can silently drop probes, so "no response" is ambiguous.
- **Legal note:** scanning without authorization can count as unauthorized access in many countries (for example the IT Act 2000 in India, and the CFAA in the US). Get written scope first. This is general information, not legal advice.

---

## 2. Footprinting vs Scanning vs Enumeration

| | Footprinting | Scanning | Enumeration |
|---|---|---|---|
| Nature | Mostly passive (OSINT) | Active probing | Active, deeper interaction |
| Touches the target? | Usually no | Yes | Yes, with real sessions |
| Output | Domains, emails, IP ranges | Live hosts, ports, services | Usernames, shares, banners, configs |
| Detectable? | Rarely | Often | Very often |
| Example tool | theHarvester, Shodan | Nmap, ping | enum4linux, SNMP walk |

In your course's six-phase model, enumeration is its own phase. Scanning answers "what is there?" and enumeration answers "what can I pull out of it?"

---

## 3. Types of Scanning

| Type | Layer it works at | Purpose | Example |
|---|---|---|---|
| Network (host) scanning | Network (ARP, ICMP) | Find live hosts | Ping sweep |
| Port scanning | Transport (TCP, UDP) | Find open ports and services | `nmap -sS` |
| Vulnerability scanning | Application | Match services to known flaws | Nessus, OpenVAS |

Common order: host discovery, then port scan, then service and OS detection, then vulnerability scan.

---

## 4. Port States and Port Ranges

### Port states (Nmap terms)

| State | Meaning | Typical sign |
|---|---|---|
| open | A service accepts connections | SYN+ACK reply |
| closed | Reachable, but nothing listening | RST reply |
| filtered | A firewall blocks the probe | No reply, or ICMP unreachable |
| unfiltered | Reachable, but open or closed is unknown | RST to an ACK probe |
| open\|filtered | Cannot tell which | No reply (UDP, FIN, NULL, Xmas) |
| closed\|filtered | Cannot tell which | Seen in idle scans |

### Port ranges

| Range | Name | Use |
|---|---|---|
| 0 to 1023 | Well-known | Standard services, need root to bind |
| 1024 to 49151 | Registered | Applications and databases |
| 49152 to 65535 | Dynamic / ephemeral | Temporary client-side ports |

(Linux uses 32768 to 60999 for ephemeral ports by default.)

### Common ports to recognize

| Port | Service | Port | Service |
|---|---|---|---|
| 21 | FTP | 143 | IMAP |
| 22 | SSH | 161 (UDP) | SNMP |
| 23 | Telnet | 389 | LDAP |
| 25 | SMTP | 443 | HTTPS |
| 53 | DNS (UDP and TCP) | 445 | SMB |
| 80 | HTTP | 3306 | MySQL |
| 110 | POP3 | 3389 | RDP |
| 123 (UDP) | NTP | 8080 | HTTP alternate |

---

## 5. Scanning Methodology

The standard scanning methodology moves from broad to specific. Each step feeds the next.

```
Authorize -> Live hosts -> Open ports -> Beyond IDS/firewall -> Banners & OS -> Vulnerabilities -> Network diagram -> Proxies
```

| Step | What you do | Typical tools | Covered later in |
|---|---|---|---|
| 0. Scope | Confirm written authorization, targets, time window | Rules of engagement | |
| 1. Check live systems | Find which hosts respond (ICMP, ARP, TCP/UDP pings) | ping, Angry IP Scanner, Nmap `-sn` | Host Discovery |
| 2. Check open ports | Probe ports with different scan types | Nmap, hping3 | Port Scanning, Nmap |
| 3. Scan beyond IDS and firewall | Fragmentation, decoys, timing, spoofing | Nmap evasion options | Evasion |
| 4. Banner grabbing and OS detection | Read service banners, fingerprint the OS | netcat, ID Serve, Nmap `-sV -O` | Banner Grabbing |
| 5. Scan for vulnerabilities | Match versions to known flaws | Nessus, OpenVAS, nikto, WPScan | Vulnerability Scanning |
| 6. Draw network diagrams | Map hosts, links, and devices | Spiceworks, LAN State Pro | Network Mapping |
| 7. Prepare proxies | Hide or route the scan source (only where permitted) | Proxychains, Tor | |

### Host discovery methods (preview)

| Method | How it works | Notes |
|---|---|---|
| ARP ping | ARP request on the local LAN | Very reliable on a LAN, cannot cross routers |
| ICMP echo | Type 8 request, type 0 reply | Often blocked by firewalls |
| TCP SYN ping | SYN to port 80 or 443 | Passes many firewalls |
| TCP ACK ping | ACK to a port, expect RST | Bypasses simple stateless filters |
| UDP ping | UDP to a closed port, expect ICMP unreachable | Slower, less common |

### Good practice

- Start quiet (slow timing, few ports), then widen only if needed.
- Save raw output of every scan. You will need it for reports.
- A host that does not answer ping may still be alive. Do not skip it.

---

## 6. TCP/IP Model and Where Scanning Operates

| TCP/IP layer | OSI equivalent | Protocols | Scanning use |
|---|---|---|---|
| Application | 5 to 7 | HTTP, FTP, SSH, DNS | Banner grabbing, vulnerability checks |
| Transport | 4 | TCP, UDP | Port scanning |
| Internet | 3 | IP, ICMP | Ping sweeps, traceroute |
| Network access | 1 and 2 | Ethernet, ARP | Local host discovery |

Scanning mostly works by sending crafted packets at the **Transport** layer and reading what comes back.

---

## 7. TCP vs UDP

| | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (handshake) | Connectionless |
| Reliability | Acknowledgements and retransmission | None |
| Header size | 20 to 60 bytes | 8 bytes |
| Speed | Slower | Faster |
| Used by | HTTP, HTTPS, SSH, SMTP, FTP | DNS queries, DHCP, SNMP, NTP, VoIP |

**What this means for scanning**

| Port state | TCP reaction | UDP reaction |
|---|---|---|
| Open | SYN+ACK | Often silence, sometimes a service reply |
| Closed | RST | ICMP port unreachable (type 3, code 3) |
| Filtered | No reply or ICMP unreachable | No reply |

UDP scans are slow and ambiguous because silence can mean open or filtered. Many systems also rate-limit ICMP errors.

---

## 8. TCP Header

A TCP header is **20 bytes minimum** and up to 60 bytes with options.

| Field | Size | Role | Why scanners care |
|---|---|---|---|
| Source port | 16 bits | Sender's port | Scanners can fix or randomize it |
| Destination port | 16 bits | Target port | The port being probed |
| Sequence number | 32 bits | Position of the first data byte | Initial value (ISN) is random |
| Acknowledgment number | 32 bits | Next byte expected | Proves the reply is tied to your packet |
| Data offset | 4 bits | Header length in 32-bit words | |
| Flags | 8 bits (+ NS) | Control bits (SYN, ACK, ...) | The basis of nearly every scan type |
| Window size | 16 bits | Receive buffer space | Used by the Window scan and OS fingerprinting |
| Checksum | 16 bits | Error check | Bad checksums reveal firewalls or IDS in some tests |
| Urgent pointer | 16 bits | Marks urgent data | Set in Xmas scans |
| Options | 0 to 40 bytes | MSS, window scale, SACK, timestamps | Order and values help **fingerprint the OS** |

---

## 9. TCP Flags

Flags are single bits in byte 13 of the TCP header. Each one tells the receiver what the packet is for.

| Flag | Name | Hex value | tcpdump | Meaning | Scanning relevance |
|---|---|---|---|---|---|
| FIN | Finish | 0x01 | `F` | Sender has no more data, close politely | FIN scan |
| SYN | Synchronize | 0x02 | `S` | Start a connection, sync sequence numbers | SYN (stealth) scan |
| RST | Reset | 0x04 | `R` | Abort the connection | A reply of RST means "closed" |
| PSH | Push | 0x08 | `P` | Deliver data to the application now | Part of Xmas scan |
| ACK | Acknowledge | 0x10 | `.` | Confirms received data | ACK scan maps firewall rules |
| URG | Urgent | 0x20 | `U` | Urgent pointer is valid | Part of Xmas scan |
| ECE | ECN Echo | 0x40 | `E` | Congestion signal echo | Rarely used in scans |
| CWR | Congestion Window Reduced | 0x80 | `W` | Sender slowed down | Rarely used in scans |

(A ninth bit, NS, is experimental and almost never seen.)

**Syntax for reading flags:** add the hex values of the set flags.

| Flags set | Calculation | Hex | Wireshark filter |
|---|---|---|---|
| SYN | 0x02 | 0x002 | `tcp.flags == 0x002` |
| SYN, ACK | 0x02 + 0x10 | 0x012 | `tcp.flags == 0x012` |
| ACK | 0x10 | 0x010 | `tcp.flags == 0x010` |
| PSH, ACK | 0x08 + 0x10 | 0x018 | `tcp.flags == 0x018` |
| FIN, ACK | 0x01 + 0x10 | 0x011 | `tcp.flags == 0x011` |
| RST | 0x04 | 0x004 | `tcp.flags == 0x004` |
| RST, ACK | 0x04 + 0x10 | 0x014 | `tcp.flags == 0x014` |
| FIN, PSH, URG (Xmas) | 0x01 + 0x08 + 0x20 | 0x029 | `tcp.flags == 0x029` |
| None (NULL) | 0 | 0x000 | `tcp.flags == 0x000` |

### Normal and abnormal flag combinations

| Combination | Seen in normal traffic? | Meaning |
|---|---|---|
| SYN | Yes | Connection request |
| SYN+ACK | Yes | Connection accepted |
| ACK | Yes | Almost every packet after the handshake |
| PSH+ACK | Yes | Data being delivered |
| FIN+ACK | Yes | Connection closing |
| RST or RST+ACK | Yes | Closed port or abort |
| NULL (no flags) | **No** | NULL scan probe |
| FIN alone | **No** (without a connection) | FIN scan probe |
| FIN+PSH+URG | **No** | Xmas scan probe |
| SYN+FIN | **No** | Invalid, used to confuse filters |

IDS and firewalls flag the "No" rows because legitimate stacks do not send them.

### Sequence and acknowledgment numbers

- A SYN or FIN consumes **one** sequence number. Data consumes one per byte.
- The ACK number is the **next sequence number the receiver expects**, so `ack = received seq + 1` for a SYN.
- The initial sequence number (ISN) is random. A predictable ISN is a security weakness.

---

## 10. TCP Three-Way Handshake

Before any data moves, TCP sets up a connection with three packets.

**Syntax (what you see in tcpdump):**
`SRC > DST: Flags [S], seq X` then `Flags [S.], seq Y, ack X+1` then `Flags [.], ack Y+1`

```
 Client                                   Server
 (CLOSED)                                 (LISTEN)
    |  1. SYN          seq = x             |
    |------------------------------------->|   client: SYN_SENT
    |                                      |   server: SYN_RECEIVED
    |  2. SYN + ACK    seq = y, ack = x+1  |
    |<-------------------------------------|
    |  3. ACK          ack = y+1           |
    |------------------------------------->|   both: ESTABLISHED
    |                                      |
    |============ data flows ==============|
```

| Step | From | Flags | Numbers | Purpose |
|---|---|---|---|---|
| 1 | Client | SYN | seq = x | "I want to talk, my starting number is x" |
| 2 | Server | SYN+ACK | seq = y, ack = x+1 | "OK, my starting number is y, I got yours" |
| 3 | Client | ACK | ack = y+1 | "Got yours, we are connected" |

**Why three steps:** both sides pick and confirm their own starting sequence numbers, each side confirms the other can receive, and old duplicate packets cannot open a false connection.

### Real capture (loopback, options trimmed)

```
127.0.0.1.59502 > 127.0.0.1.9000: Flags [S],  seq 1644431437
127.0.0.1.9000 > 127.0.0.1.59502: Flags [S.], seq 1027674059, ack 1644431438
127.0.0.1.59502 > 127.0.0.1.9000: Flags [.],  ack 1027674060
```

Check the math: `1644431437 + 1 = 1644431438` and `1027674059 + 1 = 1027674060`.

### How the target answers a connection attempt

| Port state | Reply to SYN | Result for the scanner |
|---|---|---|
| Open | SYN+ACK | Open |
| Closed | RST+ACK | Closed (real capture: `Flags [R.], seq 0, ack ...`) |
| Filtered (firewall drops) | Nothing | Filtered, after retransmissions and a timeout |
| Filtered (firewall rejects) | ICMP type 3 (codes 1, 2, 3, 9, 10, 13) | Filtered |

A silent drop makes scans slow because the scanner waits and retries.

---

## 11. TCP Connection Termination and Reset

**Graceful close (FIN):** each side closes its own direction.

```
 A: FIN+ACK  ->  B
 A  <-  ACK       B
 A  <-  FIN+ACK   B
 A: ACK      ->  B       (A waits in TIME_WAIT before fully closing)
```

In practice the middle two steps often merge, so you see three packets. The real capture showed `[F.]`, then `[F.]`, then `[.]`.

**Abrupt close (RST):** one packet ends the connection at once. Sent when a packet arrives for a closed port or an unknown connection.

### TCP states you will see in `ss` or `netstat`

| State | Meaning |
|---|---|
| LISTEN | Server waiting for connections |
| SYN-SENT | Client sent SYN, waiting |
| SYN-RECV | Server got SYN, sent SYN+ACK, waiting for ACK |
| ESTABLISHED | Handshake done, data flowing |
| FIN-WAIT-1 / FIN-WAIT-2 | Local side closing |
| CLOSE-WAIT | Remote side closed, local app has not |
| LAST-ACK | Waiting for the final ACK |
| TIME-WAIT | Waiting out stray packets after close |

Many `SYN-RECV` entries from many sources can indicate a **SYN flood** (half-open connections filling the queue). The standard defense is SYN cookies.

---

## 12. How Scan Types Use the Handshake (Preview)

Every common TCP scan is a variation on the handshake. Later notes cover each one in detail.

| Scan | Nmap flag | Packet sent | Open port reply | Closed port reply |
|---|---|---|---|---|
| TCP connect | `-sT` | Full handshake, then close | Handshake completes | RST |
| SYN (stealth, half-open) | `-sS` | SYN, then RST instead of the final ACK | SYN+ACK | RST |
| NULL | `-sN` | No flags | No reply | RST |
| FIN | `-sF` | FIN | No reply | RST |
| Xmas | `-sX` | FIN+PSH+URG | No reply | RST |
| ACK | `-sA` | ACK | RST (unfiltered) | RST (unfiltered) |
| UDP | `-sU` | UDP packet | Often none | ICMP port unreachable |

**Key points**

- **Connect scan** finishes the handshake, so the application logs the connection. It works without root.
- **SYN scan** never completes the handshake, so applications usually do not log it, but firewalls and IDS still see it. It needs raw sockets (root).
- **NULL, FIN, Xmas** follow the original TCP standard (RFC 793): closed ports reply RST, open ports stay silent. Windows and some other systems send RST for every port, so these scans are unreliable there.
- **ACK scan** does not find open ports. It shows which ports a firewall filters.

---

# Part B: Tools

## 1. tcpdump

### Tool Description & Objective

A command-line packet sniffer built on libpcap. It captures packets on an interface and prints them, with filters to keep only what you need. In scanning work it lets you **see the flags and handshakes** your scanner sends and receives, and it shows what a defender would see.

### Installation & Setup

```bash
sudo apt install tcpdump        # preinstalled on Kali
tcpdump --version
ip -br link                     # list interfaces (eth0, wlan0, lo)
```

Capturing needs root (or the right capabilities).

### Core Syntax & Flags

**Syntax:** `sudo tcpdump -i <interface> [options] '<filter>'`

| Option | Meaning |
|---|---|
| `-i <if>` | Interface (`any` for all, `lo` for loopback) |
| `-n` / `-nn` | No DNS lookups / no port-name lookups |
| `-S` | Show absolute sequence numbers |
| `-c <N>` | Stop after N packets |
| `-w file.pcap` | Write raw packets to a file |
| `-r file.pcap` | Read packets from a file |
| `-v` / `-vv` | More detail |
| `-A` / `-X` | Show payload as ASCII / hex+ASCII |
| `-d` | Print the compiled filter (good for checking syntax) |

**Flag letters in the output:** `S` SYN, `.` ACK, `F` FIN, `P` PSH, `R` RST, `U` URG, `E` ECE, `W` CWR, `none` no flags. So `[S.]` is SYN+ACK and `[P.]` is PSH+ACK.

**Filters (all tested):**

| Goal | Filter |
|---|---|
| One host or port | `host 192.168.1.20`, `port 80`, `tcp port 80` |
| Combine | `host 192.168.1.20 and tcp port 80` |
| SYN only | `'tcp[tcpflags] == tcp-syn'` |
| SYN+ACK | `'tcp[tcpflags] & (tcp-syn\|tcp-ack) == (tcp-syn\|tcp-ack)'` |
| Any RST | `'tcp[tcpflags] & tcp-rst != 0'` |
| NULL scan | `'tcp[tcpflags] == 0'` |
| Xmas scan | `'tcp[tcpflags] == (tcp-fin\|tcp-push\|tcp-urg)'` |
| By raw byte | `'tcp[13] == 2'` (SYN), `'tcp[13] == 18'` (SYN+ACK), `'tcp[13] == 41'` (Xmas) |
| ICMP unreachable | `'icmp[icmptype] == 3'` |

### Practical Workflows (Use)

**Lab: watch a handshake safely on your own machine (loopback)**

```bash
# Terminal 1: start a throwaway web server
python3 -m http.server 9000 --bind 127.0.0.1

# Terminal 2: capture
sudo tcpdump -i lo -nn -S 'tcp port 9000 or tcp port 9001'

# Terminal 3: connect to the open port, then to a closed one
curl -s -o /dev/null http://127.0.0.1:9000/
curl -s -o /dev/null --max-time 2 http://127.0.0.1:9001/
```

What you will see (trimmed from a real capture):

```
.59502 > .9000: Flags [S],  seq 1644431437                  <- SYN
.9000 > .59502: Flags [S.], seq 1027674059, ack 1644431438  <- SYN+ACK (open)
.59502 > .9000: Flags [.],  ack 1027674060                  <- ACK, connected
  ... [P.] data, [F.] [F.] [.] close ...
.56846 > .9001: Flags [S],  seq 3795083831                  <- SYN to closed port
.9001 > .56846: Flags [R.], seq 0, ack 3795083832           <- RST+ACK (closed)
```

**Watch a scan from the target's side**

```bash
sudo tcpdump -i eth0 -nn 'tcp[tcpflags] == tcp-syn'      # only SYN probes
sudo tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-rst != 0'  # only resets
```

### Tool Chaining & Automation

```bash
# Capture a scan, then analyze it later
sudo tcpdump -i eth0 -nn -w scan.pcap host 192.168.1.20

# Count SYN probes per source address
tcpdump -nn -r scan.pcap 'tcp[tcpflags] == tcp-syn' \
  | awk '{print $3}' | cut -d. -f1-4 | sort | uniq -c | sort -rn
# example result:   2 127.0.0.1
```

- Run Nmap in one terminal and tcpdump in another to see exactly what each Nmap option sends.
- Open the `.pcap` in Wireshark or tshark for deeper analysis.

### Limitations & Alternatives

- Needs root, no graphical view, and cannot read encrypted payloads (TLS).
- The `awk` pipeline above assumes IPv4 output. IPv6 addresses need different parsing.
- Busy links can drop packets, so use `-w` and a tight filter.
- Alternatives: Wireshark, tshark, ngrep, Zeek.

### Disclaimer

Capture only on networks and devices you own or are authorized to monitor.

---

## 2. Wireshark

### Tool Description & Objective

A graphical packet analyzer. It decodes hundreds of protocols and shows handshakes, flags, and conversations visually. Use it to study scan traffic and to confirm what a scan type actually sends.

### Installation & Setup

```bash
sudo apt install wireshark tshark
sudo usermod -aG wireshark $USER     # allow non-root capture, then log out and back in
wireshark &
```

Pick an interface (use `lo` for the loopback lab) and press the shark-fin button to start.

### Core Syntax & Flags

Wireshark has two filter types. They use **different syntax**.

| | Capture filter | Display filter |
|---|---|---|
| Applied | Before capture (what to save) | After capture (what to show) |
| Syntax | BPF, same as tcpdump | Wireshark field names |
| Example | `tcp port 80` | `tcp.port == 80` |

**Display filters (all tested):**

| Goal | Filter |
|---|---|
| SYN (first handshake packet) | `tcp.flags.syn == 1 && tcp.flags.ack == 0` |
| SYN+ACK | `tcp.flags.syn == 1 && tcp.flags.ack == 1` |
| Any reset | `tcp.flags.reset == 1` |
| Exact flag value | `tcp.flags == 0x002`, `0x012`, `0x014` |
| NULL scan | `tcp.flags == 0x000` |
| Xmas scan | `tcp.flags == 0x029` |
| One host or port | `ip.addr == 192.168.1.20`, `tcp.port == 80` |
| One conversation | `tcp.stream == 0` |
| ICMP port unreachable | `icmp.type == 3 && icmp.code == 3` |
| TCP problems (retransmits, resets) | `tcp.analysis.flags` |

**Useful menus**

| Menu | Use |
|---|---|
| Analyze, Follow, TCP Stream | Read one conversation end to end |
| Statistics, Conversations | See which hosts and ports talked |
| Statistics, Flow Graph | Draw the handshake as arrows |
| Preferences, Protocols, TCP, Relative sequence numbers | Toggle absolute or relative numbers |

### Practical Workflows (Use)

1. Capture on `lo`, run the loopback lab, then apply `tcp.flags.syn == 1`. Click the SYN packet and expand **Transmission Control Protocol, Flags** to see each bit.
2. Open **Statistics, Flow Graph** to see SYN, SYN+ACK, ACK as three arrows.
3. Capture while scanning your own lab host. Look for one source sending SYNs to many ports, with RSTs coming back. That pattern is a port scan.

### Tool Chaining & Automation

```bash
# tshark is the command-line version of Wireshark
tshark -r scan.pcap -Y 'tcp.flags.syn==1 && tcp.flags.ack==0' \
       -T fields -e ip.src -e tcp.dstport | sort | uniq -c | sort -rn | head
# example result:   1 127.0.0.1   9000
```

- Capture with `tcpdump -w`, then analyze in Wireshark.
- Use `-T fields -e ...` to export columns to CSV for scripts.

### Limitations & Alternatives

- Heavy on large captures, so filter first.
- Cannot decrypt TLS without keys.
- Alternatives: tcpdump, tshark, NetworkMiner, Zeek.

### Disclaimer

Capture only traffic you are authorized to inspect. Packet captures can contain passwords and personal data, so store and share them carefully.

---

## 3. ss and netstat

### Tool Description & Objective

`ss` lists the sockets on the **local machine**: listening ports, active connections, and TCP states. It shows your own attack surface, and it helps match a scan result (for example "port 8080 is open") to the process behind it.

### Installation & Setup

```bash
ss --version               # part of iproute2, preinstalled on Kali
sudo apt install net-tools # only if you want the older netstat
```

### Core Syntax & Flags

**Syntax:** `ss [options] [state <STATE>] [filter]`

| Option | Meaning |
|---|---|
| `-t` / `-u` | TCP / UDP |
| `-l` | Listening sockets only |
| `-a` | All sockets |
| `-n` | Numeric (no name lookups) |
| `-p` | Show the process (needs root) |
| `-s` | Summary counts by state |
| `-H` | No header (easier for scripts) |
| `state <STATE>` | Filter by TCP state (`established`, `listening`, `syn-recv`, `time-wait`, ...) |

`netstat -tulnp` gives a similar view. `netstat` is deprecated on modern Linux.

### Practical Workflows (Use)

```bash
sudo ss -tlnp                      # what is listening, and which process owns it
ss -tan state established          # active connections
ss -tan state syn-recv             # half-open connections (flood check)
ss -s                              # quick totals
```

Real output from the loopback lab while the server was running:

```
LISTEN 0  5  127.0.0.1:9000  0.0.0.0:*  users:(("python3",pid=1506,fd=3))
```

**Audit before you scan:** run `ss -tlnp` on a target you own, write down the expected open ports, then compare with what Nmap reports from another machine. Any difference is worth investigating.

### Tool Chaining & Automation

```bash
# List just the listening port numbers
ss -tlnH | awk '{print $4}' | sed 's/.*://' | sort -n | uniq
```

Compare that list with an Nmap scan of the same host to spot ports that are open but unexpected.

### Limitations & Alternatives

- Shows the local machine only, so it cannot inspect remote hosts.
- Process names need root.
- Alternatives: `lsof -i`, `netstat`, and `nmap localhost` for a scanner's-eye view.

### Disclaimer

Use on systems you own or administer.

---

# Cheat Sheet

| Concept | Syntax | Key Point |
|---|---|---|
| Scanning | `nmap <target>` | Active probing for hosts, ports, services, OS, vulnerabilities |
| Footprinting vs scanning | n/a | Footprinting is mostly passive, scanning touches the target |
| Open port | SYN, then SYN+ACK | A service is listening |
| Closed port | SYN, then RST | Reachable, nothing listening |
| Filtered port | SYN, then silence or ICMP type 3 | A firewall is interfering |
| Port ranges | `0-1023 / 1024-49151 / 49152-65535` | Well-known, registered, dynamic |
| Scanning methodology | n/a | Live hosts, ports, beyond IDS, banners and OS, vulnerabilities, diagram, proxies |
| TCP vs UDP | n/a | TCP replies RST for closed, UDP replies ICMP unreachable |
| TCP header | n/a | 20 to 60 bytes, flags in byte 13 |
| SYN / ACK / FIN | `0x02 / 0x10 / 0x01` | Open, acknowledge, close |
| RST / PSH / URG | `0x04 / 0x08 / 0x20` | Abort, push data, urgent |
| Flag hex math | add the values | SYN+ACK = `0x012`, Xmas = `0x029` |
| Abnormal flags | NULL, FIN alone, Xmas, SYN+FIN | Scan signatures that IDS flag |
| Three-way handshake | `SYN(x)`, `SYN+ACK(y, x+1)`, `ACK(y+1)` | ACK number is the next byte expected |
| SYN consumes seq | `ack = seq + 1` | A SYN or FIN counts as one byte |
| Graceful close | `FIN+ACK`, `ACK`, `FIN+ACK`, `ACK` | Often merges into 3 packets |
| Reset | `RST` | Immediate abort, reply to closed ports |
| Connect scan | `nmap -sT` | Full handshake, logged by apps, no root needed |
| SYN scan | `nmap -sS` | Half-open, needs root, still visible to firewalls |
| NULL / FIN / Xmas | `nmap -sN / -sF / -sX` | Follow RFC 793, unreliable on Windows |
| ACK scan | `nmap -sA` | Maps firewall rules, does not find open ports |
| tcpdump capture | `sudo tcpdump -i eth0 -nn -w f.pcap` | `-nn` skips lookups, `-S` shows absolute seq |
| tcpdump SYN filter | `'tcp[tcpflags] == tcp-syn'` | `[S]` SYN, `[S.]` SYN+ACK, `[R.]` RST+ACK |
| Wireshark SYN filter | `tcp.flags.syn==1 && tcp.flags.ack==0` | Display filter syntax differs from capture filter |
| tshark extract | `tshark -r f.pcap -Y '<filter>' -T fields -e ip.src` | Command-line Wireshark for scripts |
| Local sockets | `sudo ss -tlnp` | Shows listening ports and owning processes |
| TCP states | `ss -tan state syn-recv` | Many SYN-RECV can mean a SYN flood |

---

## Disclaimer

Only scan and capture traffic on systems you own, or where you hold explicit written authorization. Treat scan results as a snapshot of one moment, and verify findings before acting on them.