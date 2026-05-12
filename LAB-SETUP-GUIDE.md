# Test Lab Environment Setup Guide

**Purpose:** Step-by-step setup of the isolated test lab infrastructure

---

## Testing Sites

Two physical locations are used in this project. Both are Ethernet-only — no WiFi is used for device connectivity at either site.

| Site | Devices | Infrastructure | Internet uplink |
|------|---------|----------------|-----------------|
| **Corporate office (showroom)** | Main fleet under test | Corporate firewall + managed switch or Linux gateway | Corporate LAN → corp firewall → internet |
| **Home lab** | Subset of devices for detailed analysis | Linux box acting as gateway | 4G router (NAT, no WiFi) |

The **home lab** is the primary deep-analysis environment because it offers full control of the internet uplink and the Linux gateway machine. The **office showroom** is used for initial inventory, physical inspection, and long-running passive capture on the full device fleet.

**Corporate firewall — value for this project:**

The corporate firewall at the office site is a significant asset. Coordinate with IT/NOC to leverage the following:

- **Firewall logs** — request access to outbound connection logs for the VLAN or switch port segment where the signage devices are connected. These logs are independent of any monitoring you run on-device or on the Linux gateway, providing a second source of truth.
- **DNS query logging** — if the corporate DNS resolver logs queries, those records will show all hostnames the devices attempt to resolve, even if the connection itself is encrypted.
- **Geo-blocking visibility** — the firewall may already flag or block connections to certain country-code IP ranges. Check whether any signage device traffic triggers existing geo-block rules.
- **Traffic isolation** — ask IT to place the signage devices on a dedicated VLAN or firewall zone so their traffic is trivially separable from other office traffic in the logs.
- **Alert forwarding** — if the firewall feeds a SIEM, request that alerts for the signage VLAN be routed to you for the duration of the test.

> **Office network path:** DUT → Ethernet → switch (showroom) → corporate LAN → **corporate firewall** → internet. The firewall sees all outbound traffic from the devices. No Linux gateway is strictly required at the office site if firewall log access is granted — the gateway is optional here for additional PCAP depth.

> **Home lab network path:** DUT → Ethernet → Linux gateway (enp2s0) → Linux gateway (enp3s0) → 4G router (Ethernet WAN port) → 4G mobile internet. Every packet passes through the Linux host before hitting the 4G router. The 4G router needs no special configuration.

---

## Architecture Overview

Two architecture options are described. **Option B (Linux gateway) is preferred** — it eliminates the need for a managed switch with port mirroring and consolidates all analysis tools on one host.

### Option A — Managed Switch + Separate Hosts (original)

```text
  [Device Under Test] ── [Managed Switch / Port Mirror] ── [Packet Capture Host]
                                    │
                             [Test Router]
                                    │
                         [DNS Sinkhole + Proxy]
                                    │
                          [Simulated Internet]
```

### Option B — Linux Gateway + Malcolm (preferred)

```text
  [Device Under Test (DUT)]
          │  (Ethernet only)
          │
  ┌───────┴──────────────────────────────────────────────────────────────┐
  │                  LINUX GATEWAY HOST (2 NICs)                          │
  │                                                                        │
  │   NIC 1: enp2s0  ←── DUT-facing (10.99.1.1/24)                      │
  │   NIC 2: enp3s0  ──→  WAN / uplink (4G router)                       │
  │                                                                        │
  │   ── Network infrastructure (Linux, always-on) ──                     │
  │   ├── dnsmasq          DHCP + DNS logging                             │
  │   ├── nftables         NAT + per-connection logging (monitor-only)    │
  │   └── mitmproxy        Transparent HTTPS interception                 │
  │                                                                        │
  │   ── Analysis backend (Malcolm Docker stack) ──                       │
  │   ├── Zeek             Live protocol analysis (conn/dns/ssl/http/...)  │
  │   ├── Suricata         IDS signature alerts                           │
  │   ├── Arkime           Full PCAP storage + browser-based session search│
  │   ├── OpenSearch       Log indexing + anomaly detection (ML)          │
  │   └── Dashboards       40+ prebuilt dashboards, alerting, GeoIP       │
  └────────────────────────────────────────────────────────────────────────┘
          │
  [Internet] — full access, all traffic logged
```

All device traffic is **forced through the Linux host** at L3. Malcolm runs as Docker containers on the same host and listens on `enp2s0` — no separate analysis machine needed. The infrastructure layer (`dnsmasq`, `nftables`, `mitmproxy`) runs on the host OS alongside the Docker stack.

---

## Option B — Linux Gateway Setup (Preferred)

**Hardware needed:** Any Linux PC or mini PC with two NICs. Both the DUT and the gateway communicate over Ethernet — no WiFi involved.
**OS:** Ubuntu 24.04 LTS or Debian 12.
**Assumed interface names:** `enp2s0` = DUT-facing LAN (Ethernet cable from device), `enp3s0` = WAN/uplink.

**Home lab WAN uplink:** `enp3s0` connects by Ethernet to the 4G router's LAN port. The 4G router handles mobile internet — no configuration changes are needed on the router itself. Keep the router's NAT and DHCP active; the Linux gateway sits between it and the DUTs.

```text
  [DUT] ──Ethernet──▶ enp2s0 [LINUX GATEWAY] enp3s0 ──Ethernet──▶ [4G ROUTER] ──▶ Internet
```

> **Note:** Devices in this initial batch are Ethernet-only (no WiFi). This means there is exactly one network path from device to internet — through `enp2s0` on the Linux gateway. All traffic is visible on that single interface with no risk of a parallel WiFi egress path.

### B.1 — Basic IP Forwarding & NAT

```bash
# Enable IP forwarding permanently
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-ip-forward.conf
sudo sysctl -p /etc/sysctl.d/99-ip-forward.conf

# Assign static IP to DUT-facing NIC
sudo ip addr add 10.99.1.1/24 dev enp2s0
sudo ip link set enp2s0 up

# Basic NAT so DUT can reach internet via WAN NIC
sudo iptables -t nat -A POSTROUTING -o enp3s0 -j MASQUERADE
sudo iptables -A FORWARD -i enp2s0 -o enp3s0 -j ACCEPT
sudo iptables -A FORWARD -i enp3s0 -o enp2s0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Make the iptables rules survive reboot:

```bash
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

### B.2 — nftables: NAT + Full Connection Logging (monitor-only)

> **Monitoring only** — all traffic is allowed and forwarded. The goal is to observe unmodified device behavior. Blocking is done only in the customer deployment firewall, not here.

```bash
sudo apt install nftables -y
sudo systemctl enable nftables
```

Create `/etc/nftables.conf`:

```nft
#!/usr/sbin/nft -f
flush ruleset

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;
        # NAT all DUT traffic out through WAN NIC
        oifname "enp3s0" masquerade
    }
}

table ip monitor {
    chain forward {
        type filter hook forward priority 0; policy accept;

        # Log every outbound connection from DUT — new connections only
        ip saddr 10.99.1.100 ct state new \
          log prefix "DUT-OUTBOUND: " flags all

        # Log every inbound connection to DUT
        ip daddr 10.99.1.100 ct state new \
          log prefix "DUT-INBOUND: "  flags all
    }
}
```

```bash
sudo nft -f /etc/nftables.conf

# Verify rules loaded
sudo nft list ruleset

# Watch live connection log
sudo journalctl -f -k | grep "DUT-"

# Save to file for later analysis
sudo journalctl -k | grep "DUT-OUTBOUND" > /logs/nftables-connections.log
```

### B.3 — Annotate Chinese IP Ranges for Log Analysis

Rather than blocking, download the Chinese IP ranges and use them to **annotate** the connection log after the fact:

```bash
sudo apt install ipset curl -y

# Download aggregated Chinese IP ranges (update monthly)
curl -sL "https://www.ipdeny.com/ipblocks/data/aggregated/cn-aggregated.zone" \
  -o /etc/cn-ranges.txt

# Script to flag any DUT connection log entries hitting Chinese IPs
# Run this against your daily nftables/Zeek connection log
python3 << 'EOF'
import ipaddress, sys

# Load Chinese IP ranges
cn_nets = []
with open("/etc/cn-ranges.txt") as f:
    for line in f:
        line = line.strip()
        if line:
            try:
                cn_nets.append(ipaddress.ip_network(line))
            except ValueError:
                pass

def is_chinese_ip(ip):
    addr = ipaddress.ip_address(ip)
    return any(addr in net for net in cn_nets)

# Feed IPs from zeek conn.log or nftables log
for line in sys.stdin:
    # Adjust parsing to your log format
    parts = line.strip().split()
    for part in parts:
        try:
            if is_chinese_ip(part):
                print(f"[CHINA-IP] {line.strip()}")
                break
        except ValueError:
            continue
EOF
```

**Note:** Any line flagged `[CHINA-IP]` in the connection log is an immediate 🔴 CRITICAL finding to investigate.

### B.4 — dnsmasq as DHCP + DNS Logger

```bash
sudo apt install dnsmasq -y
```

Edit `/etc/dnsmasq.conf`:

```ini
# Only listen on DUT-facing NIC
interface=enp2s0
bind-interfaces

# DHCP range — static lease for device
dhcp-range=10.99.1.100,10.99.1.200,12h
dhcp-host=AA:BB:CC:DD:EE:FF,device-under-test,10.99.1.100

# Force all DNS through this host
# Upstream resolver (use your preferred one, or local unbound)
server=8.8.8.8

# Log ALL queries with timestamps
log-queries=extra
log-facility=/var/log/dnsmasq.log

# Set this host as NTP server for device
dhcp-option=42,10.99.1.1
```

```bash
sudo systemctl restart dnsmasq

# Live DNS query monitoring
sudo tail -f /var/log/dnsmasq.log | grep "10.99.1.100"

# Detect (but do NOT block) DNS queries bypassing dnsmasq to a hardcoded resolver
# These will appear in the nftables log as DUT-OUTBOUND on port 53 to a non-10.99.1.1 destination
# Look for them with:
sudo journalctl -k | grep 'DUT-OUTBOUND' | grep 'dport 53'
```

> **Important:** If the device sends DNS queries to an IP other than `10.99.1.1`, this means it has a hardcoded resolver (common in Chinese firmware). Do not block it — document the destination IP and capture the queries with Zeek/tcpdump.

### B.5 — mitmproxy Transparent Mode on Linux Gateway

```bash
pip install mitmproxy

# Redirect HTTP and HTTPS from DUT to mitmproxy (runs on port 8080)
sudo iptables -t nat -A PREROUTING -i enp2s0 -p tcp --dport 80 \
  -j REDIRECT --to-port 8080
sudo iptables -t nat -A PREROUTING -i enp2s0 -p tcp --dport 443 \
  -j REDIRECT --to-port 8080

# Run mitmproxy in transparent mode, log all flows to file
mitmdump \
  --mode transparent \
  --showhost \
  --save-stream-file /logs/mitmproxy-$(date +%Y%m%d).mitm \
  --set block_global=false \
  --listen-host 0.0.0.0 \
  --listen-port 8080 &

# To inspect live in the web UI instead:
mitmweb \
  --mode transparent \
  --web-host 0.0.0.0 \
  --web-port 8081 \
  --listen-port 8080 &
# Open http://localhost:8081 in browser
```

Install mitmproxy CA as system CA on the (rooted) device so TLS is decrypted:

```bash
CERT_PATH=$(python3 -c "import mitmproxy; import os; print(os.path.join(os.path.expanduser('~'), '.mitmproxy', 'mitmproxy-ca-cert.cer'))")
CERT_HASH=$(openssl x509 -subject_hash_old -in "$CERT_PATH" | head -1)

adb root
adb remount
adb push "$CERT_PATH" /system/etc/security/cacerts/${CERT_HASH}.0
adb shell chmod 644 /system/etc/security/cacerts/${CERT_HASH}.0
adb reboot
```

### B.6 — Zeek on the DUT-Facing Interface

```bash
# Install Zeek
sudo apt install zeek -y
# Or: https://docs.zeek.org/en/master/install.html

# Run Zeek on the DUT-facing NIC
# This captures all L2-L7 traffic before NAT modifies it
sudo zeek -i enp2s0 local &

# Key log files (in current directory or /var/log/zeek/):
#   conn.log    — every TCP/UDP/ICMP flow: src, dst, port, bytes, duration
#   dns.log     — every DNS query and response
#   ssl.log     — TLS handshakes: server cert subject, issuer, cipher suite
#   http.log    — HTTP host, URI, user-agent, response code
#   files.log   — any file transferred (by hash)
#   weird.log   — protocol anomalies (very useful)
```

Useful Zeek one-liners for device analysis:

```bash
# All unique external IPs the device connected to today
zeek-cut id.resp_h < conn.log | grep -v "^10\." | sort | uniq -c | sort -rn

# Total bytes sent by device per destination IP
zeek-cut id.resp_h orig_bytes < conn.log | \
  awk '{sum[$1]+=$2} END {for(ip in sum) print sum[ip], ip}' | \
  sort -rn | head 20

# All domains resolved
zeek-cut query < dns.log | sort | uniq -c | sort -rn

# TLS certificate subjects seen
zeek-cut server_name certificate.subject < ssl.log | sort | uniq

# Anomalies and protocol errors
zeek-cut ts id.orig_h id.resp_h name msg < weird.log
```

### B.7 — Wireshark on the Linux Gateway

```bash
sudo apt install wireshark -y
# Add your user to wireshark group to capture without root
sudo usermod -aG wireshark $USER
newgrp wireshark

# Capture on DUT-facing NIC (all device traffic visible here before NAT)
wireshark -i enp2s0 &

# Useful Wireshark display filters for this use case:
#   ip.addr == 10.99.1.100              — all device traffic
#   dns                                  — all DNS
#   ssl.handshake.type == 1              — TLS ClientHello (see SNI)
#   http                                 — unencrypted HTTP
#   tcp && !ssl && ip.dst != 10.99.1.1  — non-TLS external TCP (red flag)
#   frame contains "SERIALNUMBER"        — replace with actual serial
#   ip.geoip.country == "China"          — requires GeoIP DB installed
```

Install MaxMind GeoIP for Wireshark country-level filtering:

```bash
# Register for free at maxmind.com, download GeoLite2-Country.mmdb
sudo mkdir -p /usr/share/GeoIP
sudo cp GeoLite2-Country.mmdb /usr/share/GeoIP/
# Wireshark → Edit → Preferences → Name Resolution → enable MaxMind database
```

### B.8 — Rotating tcpdump on Linux Gateway

```bash
# Capture all DUT traffic, rotate hourly, compress, keep 90 days
sudo tcpdump -i enp2s0 \
  -w /captures/dut-%Y%m%d-%H%M.pcap \
  -G 3600 \
  host 10.99.1.100 &

# Compress captures older than 2 hours (cron job)
# Add to /etc/cron.d/pcap-rotate:
0 * * * * root find /captures -name "*.pcap" -mmin +120 -exec gzip {} \;
0 2 * * * root find /captures -name "*.pcap.gz" -mtime +90 -delete
```

### B.9 — Alternate Egress Path Risk (Ethernet-only devices)

Devices in this batch are **Ethernet-only** — no WiFi chip. This eliminates the WiFi/Ethernet bridge risk and means there is a single, fully monitored network path through `enp2s0`. However, two secondary egress vectors are still worth checking:

#### Static IP Assignment (bypassing DHCP)

A rooted device could ignore DHCP and assign itself a hardcoded static IP, either on the same subnet or a different one entirely, and attempt to reach a hardcoded gateway.

```bash
# Check what IP the device has actually configured
adb shell ip addr show eth0

# Check the routing table — does it use 10.99.1.1 as default gateway?
adb shell ip route show
# Expected: default via 10.99.1.1 dev eth0
# Flag if: any other gateway IP or any additional route entry
```

- [ ] Device uses DHCP and respects assigned gateway: ⚪ LOG — normal
- [ ] Device assigns itself a **static IP** without DHCP: 🟠 WARN — hardcoded network target
- [ ] Device sets a **different default gateway**: 🔴 CRITICAL — traffic will not pass through your monitoring host

> If a device bypasses your gateway via static routes, traffic will not appear in Zeek or nftables logs at all. Verify with ADB after every reboot that the routing table still points to `10.99.1.1`.

#### USB Tethering as a Secondary Egress Path

Even without WiFi, a rooted device can silently accept USB tethering from a connected phone and use the phone's mobile data connection — completely bypassing the Ethernet gateway.

**Rule: do not connect any phone or USB device to the DUT during testing unless it is part of a specific test case.**

```bash
# Check current tethering state
adb shell dumpsys connectivity | grep -i tether

# Check USB config — should be 'adb' only, not 'rndis' (rndis = USB tethering)
adb shell getprop persist.sys.usb.config
# Flag if output contains 'rndis'

# Check if any app has MANAGE_USB permission
grep -i "MANAGE_USB\|CHANGE_NETWORK_STATE\|MANAGE_NETWORK_POLICY" dumpsys_package.txt
```

| Observation | Severity | Meaning |
|---|---|---|
| DHCP used, gateway is `10.99.1.1`, single route | ⚪ LOG | Expected — all traffic monitored |
| Device assigns itself a static IP | 🟠 WARN | Hardcoded network knowledge |
| Default gateway is not `10.99.1.1` | 🔴 CRITICAL | Traffic bypasses monitoring entirely |
| USB config includes `rndis` | 🟠 WARN | USB tethering enabled — do not plug phones in |
| USB tethering activates automatically on USB connect | 🔴 CRITICAL | Second egress path via phone mobile data |

#### USB Tethering as a Third Egress Path

Also check whether the device silently enables USB tethering if a phone is connected (another pivot vector):

```bash
# Check USB tethering state
adb shell dumpsys connectivity | grep -i tether

# Check if device accepts incoming USB tethering
adb shell getprop persist.sys.usb.config

# Monitor USB events during testing
adb shell getevent | grep -i usb
```

---

### B.11 — OSI Layer Coverage Summary

| OSI Layer | Tool | What Is Captured |
|---|---|---|
| L2 — Data Link | Wireshark / tcpdump on `enp2s0` | ARP, MAC addresses, broadcast storms |
| L3 — Network | nftables logs, Zeek `conn.log` | IP geolocation, ASN, ICMP, routing anomalies |
| L4 — Transport | Zeek `conn.log`, nftables | TCP/UDP ports, connection state, beaconing intervals |
| L5/L6 — Session/Presentation | mitmproxy, Zeek `ssl.log` | TLS cert inspection, cipher suites, certificate pinning |
| L7 — Application | mitmproxy, Zeek `dns.log` `http.log` | DNS queries, HTTP payloads, user-agents, API calls, uploaded data |

| L2 — Data Link | Wireshark / tcpdump on `enp2s0` | ARP, MAC addresses, broadcast storms |
| L3 — Network | nftables logs, Zeek `conn.log` | IP geolocation, ASN, ICMP, routing anomalies |
| L4 — Transport | Zeek `conn.log`, nftables | TCP/UDP ports, connection state, beaconing intervals |
| L5/L6 — Session/Presentation | mitmproxy, Zeek `ssl.log` | TLS cert inspection, cipher suites, certificate pinning |
| L7 — Application | mitmproxy, Zeek `dns.log` `http.log` | DNS queries, HTTP payloads, user-agents, API calls, uploaded data |

All layers are visible **before NAT** on `enp2s0` — this is the key advantage of the Linux gateway over router-only logging.

---

### B.10 — Malcolm: Integrated Analysis Backend

[Malcolm](https://github.com/cisagov/Malcolm) (CISA / Idaho National Lab, Apache 2.0) replaces the manually assembled Zeek + tcpdump + Elasticsearch + Kibana stack with a single pre-wired Docker Compose deployment. It adds Suricata IDS, Arkime full-PCAP browser, OpenSearch anomaly detection, DGA/entropy detection, YARA file scanning, threat intel feed integration, and 40+ prebuilt dashboards — all configured out of the box.

**What Malcolm replaces from the individual-component plan:**

| Individual component | Malcolm equivalent |
|---|---|
| `tcpdump` rotating PCAP | Arkime capture (PCAP stored + indexed) |
| Zeek (manual) | Zeek (pre-configured, 30+ protocol parsers) |
| ❌ none | Suricata IDS — signature-based alerts |
| Kibana (manual dashboards) | OpenSearch Dashboards — 40+ prebuilt dashboards |
| Manual log parsing | Logstash + Filebeat pipelines, all pre-wired |
| ❌ none | JA4+ TLS fingerprinting (via Zeek plugin) |
| ❌ none | freq server — entropy/DGA detection on DNS queries |
| ❌ none | MISP/TAXII threat intel auto-correlation |
| ❌ none | Strelka file scanner (YARA + Capa + ClamAV) |
| Manual GeoIP | MaxMind GeoLite2 built-in |

**What Malcolm does NOT replace (keep running on the host OS):**

- `dnsmasq` — DHCP server and DNS logger for the DUT subnet
- `nftables` — NAT and per-connection kernel-level logging
- `mitmproxy` — transparent TLS interception (Malcolm sees metadata only; mitmproxy sees plaintext)

#### B.10.1 — Hardware Requirements

Malcolm is heavier than individual tools. Minimum for a single-device lab:

| Resource | Minimum | Recommended |
|---|---|---|
| RAM | 16 GB | 32 GB |
| CPU | 4 cores | 8 cores |
| Storage | 250 GB SSD | 500 GB+ SSD |
| OS | Ubuntu 22.04/24.04, Debian 12 | same |

#### B.10.2 — Installation

```bash
# Install Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker

# Clone Malcolm
git clone --depth 1 https://github.com/cisagov/Malcolm.git
cd Malcolm

# Run the interactive setup wizard
# Accept defaults where unsure; key choices:
#   - capture interface: enp2s0
#   - live capture: yes
#   - Suricata: yes
#   - mitmproxy integration: no (mitmproxy runs separately on host)
python3 scripts/install.py

# Pull container images (~10–15 GB, takes time on first run)
docker compose pull

# Start Malcolm
docker compose up -d
```

Malcolm's web UI is available at `https://localhost` after startup (self-signed cert, accept the warning).
Default credentials are set during `install.py`.

#### B.10.3 — Point Malcolm at the DUT Interface

During `install.py` or by editing `malcolm-environment.env`:

```bash
# In malcolm-environment.env:
PCAP_IFACE=enp2s0          # capture on DUT-facing NIC
ZEEK_LIVE_CAPTURE=true
SURICATA_LIVE_CAPTURE=true
```

Malcolm's Zeek and Suricata containers will attach directly to `enp2s0` and begin capturing. No tcpdump wrapper needed — Arkime stores and indexes the raw PCAP automatically.

#### B.10.4 — Key Dashboards for This Use Case

Once Malcolm is running, open the OpenSearch Dashboards interface (linked from the main Malcolm UI) and look at:

| Dashboard | What to watch |
|---|---|
| **Connections** | Destination IPs, countries, ASNs — filter by `enp2s0` source |
| **DNS** | All queried domains, especially those not matching known CDN patterns |
| **TLS** | Certificate subjects, issuers, JA4+ fingerprints, expired/self-signed certs |
| **Suricata Alerts** | Any signature hits — even low-severity ones are worth noting |
| **File Transfers** | Zeek-extracted files scanned by Strelka/YARA |
| **Connections by Country** | GeoIP map — flag any Chinese ASN traffic |

#### B.10.5 — Uploading Offline PCAPs

If you capture a PCAP elsewhere (e.g., at the office with tcpdump), upload it to Malcolm for indexing:

```bash
# Via the Malcolm web UI → Upload PCAP / Zeek logs
# Or via the upload container directly:
curl -u analyst:password -F "filedata=@/path/to/capture.pcap" \
  https://localhost/upload
```

Malcolm will process the PCAP through Zeek, Suricata, and Arkime and add it to the searchable index — making it possible to correlate office captures with home lab captures in one interface.

---

### B.12 — Known-Good Traffic Filtering (BCN CMS + Schedule-Aware Analysis)

#### B.12.1 — Why Filter CMS Traffic from PCAP Storage (But Never from Metadata)

Content delivery from Doohlab BCN is the highest-volume traffic source — potentially 80%+ of total bytes. Storing full PCAP for it wastes storage without adding security value.

**Rule: filter CMS traffic from PCAP payload storage only. Never filter it from Zeek logs or connection metadata.** The connection still appears in `conn.log`, `ssl.log`, and dashboards — only the raw packet bytes are not saved.

Risks that only show up in CMS traffic metadata (reasons to keep watching it):

- **Data piggybacking** — exfil payload embedded in a content sync request (upload bytes to CMS endpoint much larger than expected)
- **DNS substitution** — device resolves `bcn.doohlab.com` but connects to a different IP than BCN's known CIDR
- **TLS certificate mismatch** — cert subject in `ssl.log` does not match `*.doohlab.com`
- **Out-of-schedule connections** — device contacts CMS at a time when no content is scheduled (see B.12.3)

#### B.12.2 — Arkime PCAP Filter for Known-Good CMS Traffic

First, resolve and document BCN's origin IP ranges. Update this list whenever BCN infrastructure changes:

```bash
# /etc/known-good-cidrs.conf
# Doohlab BCN CMS — update with real IPs from DNS/NOC
# dig +short bcn.doohlab.com
# Check with your BCN account contact for full CDN/origin CIDR list
BCN_CIDRS="203.0.113.0/24"    # placeholder — replace with real BCN CIDRs
```

Add the filter to Arkime's config so it skips PCAP storage for matching sessions
while still indexing session metadata:

```ini
# /opt/malcolm/config/arkime.ini  (or arkime/config/config.ini in Malcolm)

# Drop PCAP payload for known-good CMS traffic
# Metadata (conn.log, ssl.log, session record) is still indexed — only bytes not stored
dontSaveBPFs=dst net 203.0.113.0/24;1
# Format: <BPF expression>;<sampleRate>
# sampleRate 1 = drop 100% of matching session PCAPs
# Add more semicolon-separated rules for additional known-good CIDRs:
# dontSaveBPFs=dst net 203.0.113.0/24;1 or dst net 198.51.100.0/24;1
```

Restart Arkime capture after editing:

```bash
cd /opt/malcolm
docker compose restart arkime
```

#### B.12.3 — Schedule-Aware Anomaly Detection

Because content is scheduled in BCN UI, you have a ground-truth timetable of when the devices *should* be downloading content. Any CMS connection outside that window is suspicious.

**Step 1 — Export the content schedule from BCN**

Export or manually record the scheduled delivery windows for each device (or device group) as a simple CSV:

```text
# /etc/bcn-schedule.csv
# device_mac, window_start (HH:MM UTC), window_end (HH:MM UTC), days (mon-fri / daily / etc)
AA:BB:CC:DD:EE:01, 06:00, 08:00, daily
AA:BB:CC:DD:EE:01, 14:00, 15:00, mon-fri
AA:BB:CC:DD:EE:02, 07:00, 09:00, daily
```

**Step 2 — Cross-reference Zeek `conn.log` against the schedule**

Run this after each capture day:

```python
#!/usr/bin/env python3
# check-bcn-schedule.py — flag CMS connections outside scheduled windows
import csv
import sys
from datetime import datetime, time

BCN_CIDRS = ["203.0.113.0/24"]  # replace with real BCN CIDRs
SCHEDULE_FILE = "/etc/bcn-schedule.csv"

import ipaddress
bcn_nets = [ipaddress.ip_network(c) for c in BCN_CIDRS]

def is_bcn(ip):
    try:
        addr = ipaddress.ip_address(ip)
        return any(addr in net for net in bcn_nets)
    except ValueError:
        return False

def in_window(ts, start_str, end_str):
    t = datetime.utcfromtimestamp(float(ts)).time()
    s = time.fromisoformat(start_str)
    e = time.fromisoformat(end_str)
    return s <= t <= e

# Load schedule: {mac: [(start, end, days), ...]}
schedule = {}
with open(SCHEDULE_FILE) as f:
    for row in csv.reader(f):
        if not row or row[0].startswith("#"):
            continue
        mac, start, end, days = [x.strip() for x in row]
        schedule.setdefault(mac.lower(), []).append((start, end, days))

# Read zeek conn.log from stdin (zeek-cut ts id.orig_h id.resp_h)
# Usage: zeek-cut ts id.orig_h id.resp_h < conn.log | python3 check-bcn-schedule.py
for line in sys.stdin:
    parts = line.strip().split("\t")
    if len(parts) < 3:
        continue
    ts, src, dst = parts[0], parts[1], parts[2]
    if not is_bcn(dst):
        continue
    # Find device MAC from ARP/DHCP log or use IP as proxy
    # For simplicity, match by source IP mapped to MAC via dnsmasq lease
    matched_window = False
    for mac, windows in schedule.items():
        for start, end, days in windows:
            if in_window(ts, start, end):
                matched_window = True
                break
    if not matched_window:
        dt = datetime.utcfromtimestamp(float(ts)).strftime("%Y-%m-%d %H:%M:%S UTC")
        print(f"[OUT-OF-SCHEDULE] {dt}  {src} → {dst}  (CMS contact outside any scheduled window)")
```

Run daily:

```bash
zeek-cut ts id.orig_h id.resp_h < /var/log/zeek/conn.log | \
  python3 /opt/scripts/check-bcn-schedule.py >> /logs/schedule-anomalies.log
```

Any `[OUT-OF-SCHEDULE]` line is a 🟠 WARN finding — investigate whether it was a legitimate background sync (BCN may do health checks), an unscheduled firmware pull, or unexpected exfil.

#### B.12.4 — CMS Traffic Integrity Checks (Always-On)

These checks run against Zeek metadata regardless of PCAP filtering:

```bash
# 1. BCN destination IP is inside known CIDR (flag anything going elsewhere)
zeek-cut id.resp_h < conn.log | sort -u | while read ip; do
  python3 -c "
import ipaddress, sys
nets = [ipaddress.ip_network('203.0.113.0/24')]  # replace
ip = ipaddress.ip_address('$ip')
if not any(ip in n for n in nets):
    print(f'[UNEXPECTED-CMS-IP] $ip')
"
done

# 2. TLS cert subject matches BCN domain
zeek-cut server_name certificate.subject < ssl.log | \
  grep -v "doohlab.com" | \
  grep -i "bcn\|cms\|content" | \
  awk '{print "[CERT-MISMATCH]", $0}'

# 3. Upload bytes to CMS endpoint above baseline (flag > 1 MB upload to CMS)
zeek-cut id.resp_h orig_bytes < conn.log | awk -v threshold=1048576 '
  $2 > threshold {print "[LARGE-UPLOAD-TO-CMS]", $1, $2, "bytes"}'
```

#### B.12.5 — Summary: What to Store and What to Skip

| Traffic type | PCAP payload | Zeek metadata | Reason |
|---|---|---|---|
| BCN CMS (in-schedule) | ❌ skip | ✅ keep | Known-good source; metadata sufficient |
| BCN CMS (out-of-schedule) | ✅ store | ✅ keep | Anomalous — full payload needed |
| All other destinations | ✅ store | ✅ keep | Unknown — full capture required |
| nftables / dnsmasq local | ❌ skip | ✅ keep | Management traffic, no exfil risk |

---

### OpenWrt Setup

```bash
# Install OpenWrt on router
# Enable full connection logging

# /etc/config/firewall - add logging to all rules
config rule
    option name 'log-all-outbound'
    option src 'lan'
    option dest 'wan'
    option target 'ACCEPT'
    option log '1'
    option log_ip '1'

# Enable DHCP static lease for device
# /etc/config/dhcp
config host
    option name 'device-under-test'
    option mac 'AA:BB:CC:DD:EE:FF'  # device MAC
    option ip '10.99.1.100'

# NTP server - serve local time
# /etc/config/system
config timeserver 'ntp'
    option enabled '1'
    list server '0.pool.ntp.org'

# Firewall: log all denied connections
uci set firewall.@defaults[0].drop_invalid=1
uci commit firewall
```

### pfSense / OPNsense Setup

1. Create new VLAN for test lab (e.g., VLAN 99, 10.99.1.0/24)
2. Enable **firewall logging** on all LAN-to-WAN rules
3. Enable **pfBlockerNG** with GeoIP blocking (configure to block CN, optionally HK)
4. Configure **DHCP** with static mapping for device MAC
5. Enable **ntpd** — serve NTP to LAN
6. Export firewall logs to SIEM/ELK

---

## 2. DNS Sinkhole — Pi-hole

### Install Pi-hole (Docker)

```bash
# docker-compose.yml for Pi-hole
version: "3"
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    network_mode: host
    environment:
      TZ: 'Europe/Helsinki'
      WEBPASSWORD: 'changeme-strong-password'
      PIHOLE_DNS_: '8.8.8.8;8.8.4.4'
      QUERY_LOGGING: 'true'
    volumes:
      - ./pihole/etc-pihole:/etc/pihole
      - ./pihole/etc-dnsmasq.d:/etc/dnsmasq.d
    restart: unless-stopped
```

```bash
docker-compose up -d

# Verify running
docker logs pihole
```

### Configure Pi-hole for Maximum Logging

1. Admin UI → Settings → DNS → Enable "Log queries"
2. Admin UI → Settings → DNS → Set upstream DNS to your preferred resolver
3. **Do NOT enable any blocking lists initially** — you want to see all queries first

### Export Pi-hole Logs

```bash
# Continuous export for SIEM
docker exec pihole tail -f /var/log/pihole.log | \
  awk '{print strftime("%Y-%m-%dT%H:%M:%S"), $0}' >> /logs/dns-queries.log
```

---

## 3. Transparent Proxy — mitmproxy

### Install mitmproxy

```bash
# Install via pip
pip install mitmproxy

# Or Docker
docker run --rm -it -p 8080:8080 -p 8081:8081 \
  -v ~/.mitmproxy:/home/mitmproxy/.mitmproxy \
  mitmproxy/mitmproxy mitmweb --web-host 0.0.0.0
```

### Run as Transparent Proxy

```bash
# Start transparent proxy on capture host
mitmproxy --mode transparent --showhost \
  --save-stream-file /logs/mitmproxy-$(date +%Y%m%d).mitm \
  --set block_global=false

# Or headless with logging
mitmdump --mode transparent --showhost \
  --save-stream-file /logs/mitmproxy-$(date +%Y%m%d).mitm \
  -w /logs/flows-$(date +%Y%m%d).dump
```

### Router Rules to Redirect Traffic to Proxy

```bash
# On OpenWrt / iptables
iptables -t nat -A PREROUTING -i br-lan -p tcp --dport 80 \
  -j REDIRECT --to-port 8080
iptables -t nat -A PREROUTING -i br-lan -p tcp --dport 443 \
  -j REDIRECT --to-port 8080
```

### Install mitmproxy CA on Device

```bash
# Copy CA cert to device
adb push ~/.mitmproxy/mitmproxy-ca-cert.cer /sdcard/mitmproxy-ca.cer

# Install via Android settings
adb shell am start -n com.android.certinstaller/.CertInstallerMain \
  -a android.intent.action.VIEW \
  -d file:///sdcard/mitmproxy-ca.cer
```

**Note:** If device uses certificate pinning, you will need to bypass it.
On a rooted device you can use the Magisk module `MagiskTrustUserCerts` or
install the cert as a system CA:

```bash
# Install mitmproxy cert as system CA (requires root)
adb root
# Hash the cert
CERT_HASH=$(openssl x509 -subject_hash_old -in ~/.mitmproxy/mitmproxy-ca-cert.cer | head -1)
adb push ~/.mitmproxy/mitmproxy-ca-cert.cer /system/etc/security/cacerts/${CERT_HASH}.0
adb shell chmod 644 /system/etc/security/cacerts/${CERT_HASH}.0
```

---

## 4. Packet Capture — Long-Term

### Managed Switch / Port Mirror

Configure your managed switch to mirror the port connected to the DUT to the capture host port.
Example (Cisco-like):

```text
monitor session 1 source interface Gi0/1
monitor session 1 destination interface Gi0/10
```

### tcpdump — Rotating Captures

```bash
# Rotate every hour, keep 90 days worth
# Naming: device-YYYYMMDD-HHMM.pcap
tcpdump -i eth0 \
  -w /captures/device-%Y%m%d-%H%M.pcap \
  -G 3600 \
  -Z pcap \
  host 10.99.1.100

# Compress old captures (run daily via cron)
find /captures -name "*.pcap" -mtime +1 -exec gzip {} \;

# Delete captures older than 90 days
find /captures -name "*.pcap.gz" -mtime +90 -delete
```

### Zeek — Automated Protocol Analysis

```bash
# Install Zeek
# Ubuntu/Debian:
sudo apt install zeek

# Run on capture interface
zeek -i eth0 local

# Key log files generated:
# /var/log/zeek/conn.log     - all connections
# /var/log/zeek/dns.log      - DNS queries
# /var/log/zeek/ssl.log      - TLS sessions + certificate info
# /var/log/zeek/http.log     - HTTP requests
# /var/log/zeek/files.log    - file transfers detected

# Query conn.log for device
grep "10.99.1.100" /var/log/zeek/conn.log | \
  awk '{print $9, $7, $10, $11}' | \  # timestamp, dst_ip, bytes sent, bytes recv
  sort | uniq -c | sort -rn | head 20
```

---

## 5. SIEM / Log Aggregator

### ELK Stack (Elasticsearch + Logstash + Kibana) — Docker Compose

```yaml
# docker-compose.yml
version: '3'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=changeme
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:8.12.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_PASSWORD=changeme
    depends_on:
      - elasticsearch

volumes:
  esdata:
```

### Logstash Pipeline — Ingest Zeek and Pi-hole Logs

```text
# /logstash/pipeline/zeek.conf
input {
  file {
    path => "/var/log/zeek/conn.log"
    type => "zeek-conn"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
  file {
    path => "/var/log/zeek/dns.log"
    type => "zeek-dns"
    start_position => "beginning"
  }
}

filter {
  if [type] == "zeek-conn" {
    csv {
      separator => " "
      columns => ["ts","uid","id.orig_h","id.orig_p","id.resp_h","id.resp_p","proto","service","duration","orig_bytes","resp_bytes","conn_state","local_orig","local_resp","missed_bytes","history","orig_pkts","orig_ip_bytes","resp_pkts","resp_ip_bytes","tunnel_parents"]
    }
    date { match => ["ts", "UNIX"] }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "zeek-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "changeme"
  }
}
```

---

## 6. ADB Workstation Setup

```bash
# Install Android SDK Platform Tools (Linux)
wget https://dl.google.com/android/repository/platform-tools-latest-linux.zip
unzip platform-tools-latest-linux.zip
export PATH=$PATH:$(pwd)/platform-tools

# Verify device connected
adb devices

# Enable verbose logging to file
adb logcat > /logs/logcat-$(date +%Y%m%d).log &

# Dump logcat with timestamps
adb logcat -v time > /logs/logcat-timestamped-$(date +%Y%m%d).log &
```

### MobSF (Mobile Security Framework) — Docker

```bash
docker run -it --rm \
  -p 8000:8000 \
  -v /tmp/mobsf/:/home/mobsf/.MobSF \
  opensecurity/mobile-security-framework-mobsf:latest

# Access at http://localhost:8000
# Upload APK files for automated analysis
# Key sections to review:
#   - MANIFEST analysis (permissions)
#   - Network security config
#   - Hardcoded secrets
#   - API endpoint scan
#   - VirusTotal scan (disable if APK may contain customer data)
```

---

## 7. Baseline Documentation Scripts

Run these and save output before each test phase:

```bash
#!/bin/bash
# baseline.sh — run at start of each phase, saves to ./baseline/YYYYMMDD/

DATE=$(date +%Y%m%d_%H%M)
DIR="./baseline/$DATE"
mkdir -p "$DIR"

echo "[*] Collecting device baseline..."

adb shell pm list packages -f > "$DIR/packages_full.txt"
adb shell pm list packages -s > "$DIR/packages_system.txt"
adb shell pm list packages -U > "$DIR/packages_uid.txt"
adb shell dumpsys package > "$DIR/dumpsys_package.txt"
adb shell dumpsys activity services > "$DIR/running_services.txt"
adb shell dumpsys jobscheduler > "$DIR/scheduled_jobs.txt"
adb shell ss -tlnp > "$DIR/listening_ports.txt"
adb shell getprop > "$DIR/getprop.txt"
adb shell ls -la /system/app/ > "$DIR/system_apps.txt"
adb shell ls -la /system/priv-app/ > "$DIR/system_priv_apps.txt"
adb shell ls -la /data/adb/ 2>/dev/null > "$DIR/magisk_modules.txt"
adb shell settings get secure android_id > "$DIR/android_id.txt"
adb shell getprop ro.serialno > "$DIR/serial.txt"

echo "[*] Baseline saved to $DIR"
echo "[*] Diff against previous baseline:"
if [ -d "./baseline/previous" ]; then
    diff -r ./baseline/previous "$DIR"
fi

# Update 'previous' symlink
rm -f ./baseline/previous
ln -s "$DIR" ./baseline/previous
```

---

## 8. Quick Reference Commands

```bash
# Find all connections from device today in Zeek
grep "10.99.1.100" /var/log/zeek/conn.log | \
  grep -v "10.99.1." | \
  awk '{print $5}' | sort | uniq -c | sort -rn

# Find all domains resolved by device today
grep "10.99.1.100" /var/log/zeek/dns.log | \
  awk '{print $10}' | sort | uniq -c | sort -rn

# Check TLS certificates seen
grep "10.99.1.100" /var/log/zeek/ssl.log | \
  awk '{print $9, $10, $15}' | sort | uniq

# Search PCAP for device serial number
tshark -r capture.pcap -Y "frame contains \"$(adb shell getprop ro.serialno | tr -d '\r')\""

# Volume by destination IP (top 10)
grep "10.99.1.100" /var/log/zeek/conn.log | \
  awk '{sum[$5]+=$10} END {for(ip in sum) print sum[ip], ip}' | \
  sort -rn | head 10

# Real-time DNS monitoring
docker exec pihole tail -f /var/log/pihole.log | grep --line-buffered "10.99.1.100"
```
