# DDoS Simulation & Mitigation
### A Hands-On Network Security Lab Project for Authorized, Isolated Environments

---

## ⚠️ LEGAL & ETHICAL DISCLAIMER

> **This project is strictly for educational use in an isolated, self-owned lab environment.**
>
> - All traffic generation, load testing, and "attack simulation" techniques described in this document must **only** be run against systems you own, inside a private, isolated lab network (host-only virtual network, GNS3/EVE-NG simulation, or a private cloud sandbox you control end-to-end).
> - **Never** run any denial-of-service or stress-testing tool against a live production system, a public website, a third-party server, or any system you do not own or have **explicit written authorization** to test.
> - Cloud providers (AWS, Azure, GCP, DigitalOcean, etc.) explicitly prohibit unauthorized stress-testing/DDoS simulation in their Terms of Service, even against your own hosted instances, unless you follow their specific pre-approved testing policies (e.g., AWS's Simulated Event Submission process). Always check and follow your provider's policy before testing anything cloud-hosted.
> - Launching a denial-of-service attack against a system without authorization is a criminal offense in most jurisdictions (e.g., under the U.S. Computer Fraud and Abuse Act, the UK Computer Misuse Act, and equivalent laws worldwide) — even a "test."
> - This document is intended to demonstrate network security and defensive engineering skills for portfolio, training, and certification-preparation purposes.
> - The author/reader assumes full responsibility for ensuring all testing occurs within a legally authorized, fully isolated lab scope with no route to the public internet.

---

## 1. Project Overview

### 1.1 Objective
This project demonstrates a full defensive-security workflow around Distributed Denial of Service (DDoS) attacks: simulating controlled, low-impact attack traffic against a self-owned lab server, detecting that traffic, applying mitigation techniques, and validating their effectiveness — all within an isolated network with zero exposure to the public internet.

The goal is not to build "attack skills" in an offensive sense, but to understand **how DDoS traffic behaves, how it's detected, and how it's mitigated** — core knowledge for network security, SOC analyst, and infrastructure engineering roles.

### 1.2 Scope

| Item | Detail |
|---|---|
| **In Scope** | Isolated lab network (`192.168.100.0/24`), attacker VM(s), victim web server VM, monitoring VM |
| **Out of Scope** | Any public IP, cloud-hosted production instance, third-party network, or the open internet |
| **Testing Window** | Self-paced lab exercise |
| **Authorization** | Self-authorized — lab fully owned and operated by the tester |
| **Rules of Engagement** | No internet-facing NAT/bridged adapters active during any traffic-generation step; all traffic capped to lab-safe rate limits |

### 1.3 Tools Used

| Tool | Purpose |
|---|---|
| **Kali Linux** | Attacker VM — traffic generation and analysis tooling |
| **hping3** | Crafting and sending controlled SYN/ICMP/UDP flood-style packets (rate-limited) |
| **Apache Bench (ab)** | Simulating HTTP-layer load/application-layer request floods |
| **Target VM (Ubuntu Server + Apache/Nginx)** | Victim web server under test |
| **Wireshark / tcpdump** | Packet capture and traffic pattern analysis |
| **iptables / nftables** | Firewall-based rate limiting and mitigation |
| **Suricata** *(optional)* | Lightweight IDS for detecting attack signatures |
| **htop / vmstat / ss / netstat** | Real-time server resource and connection monitoring |
| **GNS3 / EVE-NG** *(optional)* | Simulated network topology if testing routing/multi-hop scenarios |

### 1.4 Lab Architecture

```
 ┌───────────────────────────────────────────────────────────────┐
 │                     Host Machine (Physical)                    │
 │                                                                 │
 │  ┌─────────────────┐   ┌─────────────────┐   ┌───────────────┐ │
 │  │  Kali Linux      │   │  Ubuntu Server   │   │  Monitor VM    │ │
 │  │ (Attacker VM)    │   │ (Victim/Target)  │   │ (IDS/Wireshark)│ │
 │  │ 192.168.100.10   │──►│ 192.168.100.20   │◄──│ 192.168.100.30 │ │
 │  └─────────────────┘   └─────────────────┘   └───────────────┘ │
 │            └───────────── Isolated Host-Only Network ──────────┘
 │                          (VirtualBox: vboxnet1)                 │
 │                     No internet / NAT bridging enabled          │
 └───────────────────────────────────────────────────────────────┘
```

- **Attacker VM:** Kali Linux — generates controlled, rate-limited test traffic
- **Victim VM:** Ubuntu Server running Apache/Nginx — the system under test
- **Monitoring VM:** Runs Wireshark/Suricata to observe traffic in real time, positioned to span/mirror traffic on the isolated segment
- **Network:** Fully isolated host-only virtual switch — no route to the internet or physical LAN

---

## 2. Environment Setup

### 2.1 Prerequisites
- Host machine with virtualization enabled (Intel VT-x / AMD-V)
- VirtualBox or VMware Workstation/Player
- Kali Linux VM image
- Ubuntu Server ISO (for the victim VM)
- Optional: GNS3 or EVE-NG if simulating a multi-hop network topology instead of a flat segment

### 2.2 Step-by-Step Lab Build

**Step 1 — Create an Isolated Host-Only Network**
1. VirtualBox → **File → Host Network Manager**
2. Create a new adapter (e.g., `vboxnet1`), IPv4 `192.168.100.1/24`
3. **Disable DHCP** — all VMs will use static IPs

**Step 2 — Build the Attacker VM (Kali Linux)**
1. Import Kali `.ova`, set network adapter to **Host-only Adapter → vboxnet1**
2. Assign static IP:
   ```bash
   sudo ip addr add 192.168.100.10/24 dev eth0
   sudo ip link set eth0 up
   ```
3. Install traffic tools if not already present:
   ```bash
   sudo apt update && sudo apt install hping3 apache2-utils -y
   ```

**Step 3 — Build the Victim VM (Ubuntu Server + Web Server)**
1. Install Ubuntu Server, set network adapter to **Host-only Adapter → vboxnet1**
2. Assign static IP:
   ```bash
   sudo ip addr add 192.168.100.20/24 dev enp0s3
   sudo ip link set enp0s3 up
   ```
3. Install a web server for a realistic target:
   ```bash
   sudo apt update && sudo apt install apache2 -y
   sudo systemctl enable --now apache2
   ```

**Step 4 — Build the Monitoring VM (Optional but Recommended)**
1. Any lightweight Linux distro with Wireshark/tcpdump/Suricata installed
2. Static IP `192.168.100.30/24` on the same host-only network
3. Install monitoring tools:
   ```bash
   sudo apt install wireshark tcpdump suricata -y
   ```

**Step 5 — Verify Isolation**
- Confirm **no VM** has a Bridged or NAT adapter active during traffic-generation steps
- Optionally disable the host's physical network adapter for a hard air-gap during testing

### 2.3 Connectivity Verification

From Kali (attacker):
```bash
ping -c 4 192.168.100.20
curl http://192.168.100.20
```
Expected: successful ping replies and a valid HTTP response from the Apache default page, confirming the lab network and target service are reachable and isolated.

---

## 3. DDoS Attack Simulation (Conceptual & Lab-Safe)

### 3.1 DDoS Attack Categories (Conceptual Overview)

| Category | Layer | Description | Example |
|---|---|---|---|
| **Volumetric** | Network (L3/L4) | Overwhelms bandwidth/capacity with high packet volume | UDP flood, ICMP flood |
| **Protocol-based** | Transport (L4) | Exploits protocol handshake behavior to exhaust connection state tables | SYN flood, Ping of Death |
| **Application-layer** | Application (L7) | Overwhelms application logic with seemingly legitimate requests | HTTP GET/POST flood, Slowloris |

Real-world DDoS attacks combine these layers and originate from many distributed sources (botnets). In this lab, we simulate the **traffic patterns and impact** of each category using a **single controlled attacker VM at rate-limited intensity** — enough to observe detection/mitigation effects without needing (or risking) an actual botnet-scale attack.

### 3.2 Controlled, Rate-Limited Traffic Generation

**Scenario A — Simulated SYN Flood (Protocol-layer)**
```bash
sudo hping3 -S --flood --rand-source -p 80 192.168.100.20
```
> ⚠️ Lab-safe note: even `--flood` here is confined to the isolated segment and a VM you own. Start without `--flood` first and use a controlled rate (e.g., `-i u10000` for 10,000-microsecond intervals) to observe effects gradually before increasing intensity.

Controlled/rate-limited version:
```bash
sudo hping3 -S -p 80 -i u10000 192.168.100.20
```

**Scenario B — Simulated HTTP Application-Layer Load (L7)**
```bash
ab -n 5000 -c 200 http://192.168.100.20/
```
- `-n 5000`: total requests
- `-c 200`: concurrent requests
- Simulates a burst of legitimate-looking HTTP traffic to observe how the web server's connection/request handling degrades under load.

**Scenario C — Simulated ICMP Flood (Volumetric)**
```bash
sudo hping3 --icmp --flood 192.168.100.20
```

### 3.3 Monitoring Impact

On the **victim VM**, observe resource and connection impact in real time:
```bash
htop
watch -n1 'ss -s'
watch -n1 'netstat -ant | grep :80 | wc -l'
```

On the **monitoring VM**, capture and inspect traffic:
```bash
sudo tcpdump -i eth0 -w ddos_capture.pcap
```
Open `ddos_capture.pcap` in Wireshark and filter, e.g.:
```
tcp.flags.syn == 1 and tcp.flags.ack == 0
```
to visualize the SYN flood pattern — a high volume of SYN packets with no completed handshake.

---

## 4. Detection

### 4.1 Signs of a DDoS Attack in Logs/Captures

| Indicator | Where to Look |
|---|---|
| High rate of SYN packets with no ACK completion | Wireshark/tcpdump capture |
| Explosion in `ss -s` TCP connection counts, many in `SYN-RECV` state | Victim VM connection table |
| Apache/Nginx access logs showing thousands of requests/sec from few or spoofed sources | `/var/log/apache2/access.log` |
| Sudden CPU/memory spike correlating with traffic onset | `htop`, `vmstat 1` |
| Suricata/IDS alerts for known flood signatures | Suricata `fast.log` / `eve.json` |

### 4.2 Basic Monitoring/Alerting Setup

Quick connection-state check:
```bash
ss -s
netstat -ant | awk '{print $6}' | sort | uniq -c | sort -nr
```

Basic Suricata rule example (conceptual, for lab detection of SYN flood-like behavior):
```
alert tcp any any -> $HOME_NET 80 (msg:"Possible SYN Flood Detected"; flags:S; threshold: type both, track by_dst, count 200, seconds 10; sid:1000001;)
```
This rule triggers an alert if more than 200 SYN packets to the web server are seen within 10 seconds from a tracked destination — a simple, lab-appropriate detection threshold.

---

## 5. Mitigation Strategies

### 5.1 Rate Limiting & Connection Throttling (iptables/nftables)

**Limit new SYN connections per source IP:**
```bash
sudo iptables -A INPUT -p tcp --syn -m limit --limit 1/s --limit-burst 3 -j ACCEPT
sudo iptables -A INPUT -p tcp --syn -j DROP
```

**Enable SYN cookies (kernel-level SYN flood protection):**
```bash
sudo sysctl -w net.ipv4.tcp_syncookies=1
```
SYN cookies allow the server to avoid holding half-open connection state until the handshake is verified, significantly reducing SYN flood impact without dropping legitimate traffic.

**nftables equivalent:**
```bash
sudo nft add rule inet filter input tcp flags syn limit rate 5/second accept
sudo nft add rule inet filter input tcp flags syn drop
```

### 5.2 Web Server-Level Hardening

**Apache — mod_ratelimit / mod_evasive (conceptual config):**
```apache
<IfModule mod_evasive24.c>
    DOSHashTableSize    3097
    DOSPageCount        5
    DOSPageInterval     1
    DOSSiteCount        50
    DOSSiteInterval     1
    DOSBlockingPeriod   600
</IfModule>
```
This limits how many requests a single client can make per second/site before being temporarily blocked.

**Nginx — basic rate limiting:**
```nginx
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

server {
    location / {
        limit_req zone=mylimit burst=20 nodelay;
    }
}
```

**Connection timeout tuning:**
```apache
Timeout 30
KeepAliveTimeout 5
```
Shorter timeouts reduce how long the server holds resources for slow or incomplete connections (relevant to Slowloris-style attacks).

### 5.3 Cloud/Enterprise Mitigation (Conceptual Overview Only)

These are **not implemented** in this lab since they require real cloud/ISP-level infrastructure, but are important to understand conceptually:

| Technique | How It Helps |
|---|---|
| **CDN (Content Delivery Network)** | Distributes traffic load across many edge nodes globally, absorbing volumetric spikes before they reach the origin server |
| **WAF (Web Application Firewall)** | Filters malicious application-layer requests (e.g., malformed HTTP, known bot signatures) before they reach the app |
| **Anycast routing** | Spreads incoming traffic across multiple geographically distributed data centers, diluting attack concentration |
| **Scrubbing centers** | Specialized infrastructure that inspects and filters traffic during an attack, forwarding only clean traffic to the origin |

### 5.4 Before/After Comparison

| Metric | Before Mitigation | After Mitigation |
|---|---|---|
| SYN packets/sec accepted | [fill from lab capture] | [fill from lab capture] |
| Server CPU utilization during attack | [fill from lab capture] | [fill from lab capture] |
| HTTP response time under load | [fill from lab capture] | [fill from lab capture] |
| Successful completed connections | [fill from lab capture] | [fill from lab capture] |

---

## 6. Monitoring & Validation

### 6.1 Re-Running the Simulated Attack Post-Mitigation

Repeat the same traffic-generation commands from Section 3 **after** applying the iptables/nftables rules and web server hardening, and compare results side by side.

### 6.2 Comparing Performance/Availability

```bash
watch -n1 'ss -s'
ab -n 5000 -c 200 http://192.168.100.20/
```
Compare Apache Bench's reported requests-per-second and failed-request counts before vs. after mitigation — this gives a quantifiable effectiveness measure for the report.

### 6.3 Evidence Gathering

For each test run, capture:
- Wireshark/tcpdump `.pcap` file
- Screenshot of `htop`/`ss -s` output during the attack
- Apache Bench summary output
- Timestamped before/after comparison table (see 5.4)
- Any Suricata alerts triggered

---

## 7. Reporting

### 7.1 Professional Report Template

```markdown
# DDoS Simulation & Mitigation Report — [Lab Project Name]

## 1. Executive Summary
Non-technical summary of the testing objective, key findings, and 
overall resilience posture of the tested system before/after mitigation.

## 2. Scope & Methodology
- Testing dates
- In-scope systems (isolated lab IPs only)
- Methodology followed
- Tools used

## 3. Attack Scenarios Tested

| Scenario ID | Attack Type | Tool Used | Target |
|---|---|---|---|
| A-01 | SYN Flood (Protocol-layer) | hping3 | 192.168.100.20:80 |
| A-02 | HTTP Load Flood (App-layer) | Apache Bench | 192.168.100.20:80 |
| A-03 | ICMP Flood (Volumetric) | hping3 | 192.168.100.20 |

## 4. Impact Analysis (Pre-Mitigation)
- Connection table exhaustion observations
- CPU/memory impact
- Service availability impact

## 5. Mitigation Measures Applied
- iptables/nftables rate limiting
- SYN cookies enabled
- Web server rate-limiting modules configured

## 6. Effectiveness Results (Post-Mitigation)

| Metric | Before | After | % Improvement |
|---|---|---|---|
| Requests/sec handled | | | |
| Failed connections | | | |
| CPU utilization peak | | | |

## 7. Risk Rating & Recommendations
Qualitative or CVSS-style risk rating for each attack scenario, plus 
prioritized recommendations (e.g., enable SYN cookies by default, 
add WAF/CDN for internet-facing production systems).

## 8. Appendix
- Full pcap references
- Suricata alert logs
- Screenshot gallery
```

### 7.2 Risk Rating Guidance (Example)

| Severity | Description | Example |
|---|---|---|
| Critical | Service becomes fully unavailable with minimal attacker effort | Unmitigated SYN flood causing total outage |
| High | Significant degradation, partial service disruption | HTTP flood causing major slowdowns |
| Medium | Noticeable but recoverable performance impact | Moderate ICMP flood, some packet loss |
| Low | Minimal measurable impact due to effective mitigation | Attack traffic absorbed by rate limiting |

---

## 8. Deliverables

This project produces the following portfolio-ready artifacts:

1. **Lab Setup Guide** — Section 2 of this document (network + VM build steps)
2. **Final Report** — populated version of the Section 7 template, exported to Markdown or Word (.docx)
3. **Evidence Package** — pcap captures, screenshots, before/after metrics
4. **This Project Document** — serves as a portfolio showcase piece

### 8.1 Lessons Learned / Skills Demonstrated (Portfolio Summary)

**Skills Demonstrated:**
- Isolated lab network design and segmentation for safe security testing
- Understanding of DDoS attack categories (volumetric, protocol, application-layer)
- Controlled traffic generation and impact analysis using hping3 and Apache Bench
- Traffic analysis and attack-pattern detection with Wireshark/tcpdump
- Firewall-based mitigation (iptables/nftables rate limiting, SYN cookies)
- Web server hardening (Apache/Nginx rate limiting, timeout tuning)
- Conceptual understanding of enterprise-scale mitigation (CDN, WAF, Anycast, scrubbing centers)
- Quantitative before/after effectiveness reporting

**Example Resume Bullet:**
> *"Designed and executed a DDoS simulation and mitigation exercise in an isolated lab environment, generating controlled attack traffic to test server resilience, implementing firewall- and application-level mitigations, and producing a quantitative before/after effectiveness report."*

---

*End of document. Remember: always operate within an isolated, authorized lab with no route to the public internet. Never run DDoS/stress-testing tools against any system without explicit ownership and authorization, and always check your cloud provider's stress-testing policy before testing cloud-hosted infrastructure.*
