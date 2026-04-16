# Android Digital Signage Device Security Test Protocol

**Version:** 1.0  
**Created:** 2026-04-16  
**Classification:** Internal / Confidential  
**Scope:** Chinese-manufactured rooted Android digital signage devices prior to customer deployment

---

## Table of Contents

1. [Purpose & Scope](#1-purpose--scope)
2. [Threat Model](#2-threat-model)
3. [Test Environment Setup](#3-test-environment-setup)
4. [Phase 1 — Pre-Network Physical & Static Analysis](#4-phase-1--pre-network-physical--static-analysis)
5. [Phase 2 — Isolated Network Baseline (Days 1–14)](#5-phase-2--isolated-network-baseline-days-114)
6. [Phase 3 — Deep Traffic Analysis (Weeks 2–8)](#6-phase-3--deep-traffic-analysis-weeks-28)
7. [Phase 4 — CMS & Help Desk Integration Testing (Weeks 4–8)](#7-phase-4--cms--help-desk-integration-testing-weeks-48)
8. [Phase 5 — Long-Term Monitoring (Months 2–3+)](#8-phase-5--long-term-monitoring-months-23)
9. [Phase 6 — Customer Network Simulation](#9-phase-6--customer-network-simulation)
10. [Risk Classification & Acceptance Criteria](#10-risk-classification--acceptance-criteria)
11. [Reporting & Documentation](#11-reporting--documentation)
12. [Appendices](#12-appendices)

---

## 1. Purpose & Scope

### Purpose

This protocol establishes a systematic, reproducible process for security-validating Chinese-manufactured rooted Android digital signage devices before deployment into customer networks. The primary concerns are:

- **"Calling home"** — unauthorized outbound communication to manufacturer servers, Chinese CDNs, or telemetry endpoints
- **Persistent backdoors** — pre-installed services that survive factory reset or firmware updates
- **Supply chain compromise** — malicious code injected at manufacture or during firmware signing
- **Network pivoting risk** — the device being used as a foothold into the customer LAN
- **Data exfiltration** — leakage of network topology, credentials, or customer data

### Scope

- All device models and firmware versions purchased for deployment
- Each major firmware update/batch shipment requires a new test cycle
- Both out-of-the-box state AND after CMS/helpdesk agent installation

### Out of Scope

- Physical hardware tamper testing (covered separately if required)
- Application-layer CMS software security (separate secure SDLC process)

---

## 2. Threat Model

| Threat Actor | Motivation | Likely Vector |
|---|---|---|
| Device manufacturer | Data collection / telemetry revenue | Pre-installed system apps, ROM-level agents |
| Nation-state (supply chain) | Intelligence / infrastructure access | Firmware backdoor, signed malicious APK |
| Firmware signing chain compromise | Persistent access | Tampered OTA update package |
| Rogue APK in ROM | Ad revenue / botnet | Pre-installed adware with root access |
| Weak default credentials | Opportunistic attackers on LAN | ADB open over network, default SSH/Telnet |

### Key Questions This Protocol Answers

1. Does the device communicate with any host outside our approved allowlist?
2. Do those communications contain device identifiers, PII, or network data?
3. Are there services listening on the LAN that should not be?
4. Does the device survive attempts to remain persistent after wipe/reflash?
5. Is the device's root implementation itself a security risk?

---

## 3. Test Environment Setup

### 3.1 Required Hardware & Infrastructure

| Item | Purpose |
|---|---|
| Dedicated test VLAN / physical switch | Total isolation from production and internet |
| Transparent proxy server (e.g., mitmproxy, Squid with SSL bump) | Decrypt and inspect HTTPS traffic |
| Network TAP or managed switch with port mirroring | Passive full packet capture without altering device behavior |
| Packet capture workstation (Wireshark / tcpdump / Zeek) | Long-term PCAP storage and analysis |
| DNS sinkhole (Pi-hole or BIND RPZ) | Log all DNS queries, block unwanted domains |
| NTP server (local) | Ensure accurate timestamps across all logs |
| SIEM or log aggregator (ELK / Graylog / Splunk) | Correlation, alerting, long-term storage |
| ADB workstation (Linux preferred) | Static analysis, package enumeration |
| Second clean Android device (reference) | Baseline comparison for traffic |
| Dedicated router with full NAT logging | Track all outbound connection attempts |

### 3.2 Network Topology for Testing

```text
[Device Under Test]
       |
  [Managed Switch / Mirror Port]
       |          |
  [TAP Sensor]  [Test Router / Firewall]
       |          |
  [Packet Capture]  [DNS Sinkhole]
                    [Transparent Proxy]
                    |
              [Simulated Internet] ← controlled egress, all traffic logged
                    |
              [REAL Internet] ← allowed only after explicit decision per domain
```

### 3.3 Baseline Setup Steps

- [ ] Flash test router to known-good firmware (OpenWrt recommended)
- [ ] Configure DNS sinkhole — log ALL queries, initially allow all resolution
- [ ] Configure transparent proxy with self-signed CA — install CA on device if possible
- [ ] Configure port mirror to packet capture workstation
- [ ] Set up NTP — all systems synchronized
- [ ] Set up log rotation — minimum 90-day retention
- [ ] Assign device a static DHCP lease — log by MAC address
- [ ] Document test VLAN subnet, gateway, DNS server IPs clearly
- [ ] Verify no path exists from test VLAN to production LAN

### 3.4 Device Identification & Baseline Documentation

Before ANY network connection, document:

```text
Device Model:
Firmware Version (Settings → About):
Build Number:
Android Security Patch Level:
Serial Number:
IMEI (if applicable):
MAC Address (WiFi):
MAC Address (Ethernet):
Bootloader Version:
Root Method / Version:
Pre-installed system apps (adb shell pm list packages -s):
```

---

## 4. Phase 1 — Pre-Network Physical & Static Analysis

**Duration:** 1–3 days  
**Network:** NONE — air-gapped

### 4.1 ADB Static Enumeration

Connect via USB only. WiFi ADB must be disabled.

```bash
# List ALL installed packages (system + user)
adb shell pm list packages -f > packages_full.txt

# List system packages only
adb shell pm list packages -s > packages_system.txt

# List packages with UID
adb shell pm list packages -U > packages_uid.txt

# Dump all app permissions
adb shell dumpsys package > dumpsys_package.txt

# List all running services
adb shell dumpsys activity services > running_services.txt

# List scheduled jobs (potential beaconing jobs)
adb shell dumpsys jobscheduler > scheduled_jobs.txt

# List all broadcast receivers
adb shell dumpsys activity broadcasts > broadcasts.txt

# Check for listening ports
adb shell ss -tlnp > listening_ports.txt
adb shell netstat -tlnp >> listening_ports.txt

# Check for open ADB over network
adb shell getprop service.adb.tcp.port

# Enumerate /system/app and /system/priv-app
adb shell ls -la /system/app/ > system_apps.txt
adb shell ls -la /system/priv-app/ > system_priv_apps.txt

# Check SELinux enforcement
adb shell getenforce

# Check root implementation
adb shell which su
adb shell ls -la /system/xbin/su
adb shell ls -la /system/bin/su

# Dump build properties
adb shell getprop > getprop_full.txt

# Check for Magisk / SuperSU / KingRoot remnants
adb shell ls /data/adb/
adb shell ls /sbin/.magisk/
```

- [ ] Review package list against known malicious/adware package names (see Appendix A)
- [ ] Flag any packages from `com.mediatek`, `com.rockchip`, `com.allwinner` that have INTERNET permission
- [ ] Flag any packages with `android.permission.READ_CONTACTS`, `READ_CALL_LOG`, `RECORD_AUDIO`, `ACCESS_FINE_LOCATION` for system apps
- [ ] Check for VPN service pre-installed
- [ ] Verify ADB over TCP is NOT enabled (`service.adb.tcp.port` should be empty or -1)
- [ ] Verify Telnet / SSH daemons are not running

### 4.2 APK Extraction & Static Analysis

Extract suspicious APKs and analyze:

```bash
# Extract APK
adb pull /system/priv-app/<AppName>/<AppName>.apk .

# Decompile with apktool
apktool d <AppName>.apk

# Search for hardcoded IPs and domains
grep -rE "([0-9]{1,3}\.){3}[0-9]{1,3}" <AppName>/
grep -rE "(http|https|ftp)://[^\"]+" <AppName>/

# Check certificate
apksigner verify --print-certs <AppName>.apk

# VirusTotal hash check (do NOT upload if customer data concern)
sha256sum <AppName>.apk
```

**Tools:** apktool, jadx, MobSF (Mobile Security Framework) — run MobSF locally in Docker.

```bash
docker run -it --rm -p 8000:8000 opensecurity/mobile-security-framework-mobsf
```

Upload each suspicious APK. Review:

- [ ] Network endpoints referenced in code
- [ ] Cryptographic operations (look for custom certificate pinning bypass)
- [ ] Dynamic code loading (`DexClassLoader`)
- [ ] Native library calls (`System.loadLibrary`)
- [ ] Obfuscated strings (base64, XOR encoded URLs)

### 4.3 Firmware Image Analysis (if accessible)

If you can extract the firmware image:

```bash
# Extract OTA zip or firmware image
binwalk -e firmware.img

# Scan extracted filesystem
find . -name "*.apk" | while read f; do sha256sum "$f"; done

# Check for known bad hashes against threat intel feeds
# Check /etc/hosts for manipulation
cat etc/hosts

# Check /system/etc/security/cacerts/ for rogue CA certificates
ls etc/security/cacerts/
```

- [ ] Compare `/etc/hosts` — any entries redirecting common domains?
- [ ] Compare CA certificate store against AOSP baseline — any extra CAs?
- [ ] Check for any cron-like jobs (`/etc/cron`, `/etc/init.d`, custom init scripts in `/init.rc`)

---

## 5. Phase 2 — Isolated Network Baseline (Days 1–14)

**Duration:** 14 days minimum  
**Network:** Test VLAN with full egress logging, DNS sinkhole active, transparent proxy active  
**Device State:** Out of box, no CMS software installed, device powered on and left running

### 5.1 Initial Connection

- [ ] Connect device to test VLAN
- [ ] Record first DHCP request timestamp and offered IP
- [ ] Record first DNS query and first outbound connection attempt
- [ ] Note: first 10 minutes of traffic is critical — many beaconing behaviors trigger on first network availability

### 5.2 Traffic Collection Configuration

On capture workstation:

```bash
# Continuous packet capture, rotate every hour
tcpdump -i eth0 -w /captures/device-%Y%m%d-%H%M.pcap -G 3600 -Z root

# Zeek for protocol analysis
zeek -i eth0 local

# DNS log analysis (Pi-hole)
tail -f /var/log/pihole.log | grep <device_IP>
```

### 5.3 Daily Checks (Days 1–14)

Each day, review and record:

| Check | Tool | Expected Result |
|---|---|---|
| Unique external IPs contacted | `zeek conn.log` / Wireshark | Only approved IPs |
| Unique external domains resolved | DNS sinkhole log | Only approved domains |
| Total outbound data volume (bytes) | `zeek conn.log` | Document baseline |
| Any connections on non-standard ports | `zeek conn.log` filter | None unexpected |
| TLS certificate inspection | mitmproxy log | No certificate pinning bypass needed |
| Any UDP beaconing | Wireshark filter `udp && ip.dst != <LAN>` | Document all |
| ICMP to external hosts | Wireshark | None expected |
| mDNS / SSDP / LLMNR broadcasts | Wireshark | Document scope |
| ARP anomalies | Wireshark / arpwatch | No unexpected ARP |

### 5.4 Identifying "Calling Home" Behavior

Flag and investigate ANY connection to:

- IP addresses in Chinese ASNs (AS4134 ChinaNet, AS4837 China Unicom, AS4812 China Telecom, AS9808 China Mobile, AS45090 Tencent, AS55967 Baidu, AS37963 Alibaba)
- Domains resolving to Chinese IP ranges
- Domains with `.cn` TLD
- Known telemetry domains (see Appendix B)
- Any connection sending device identifiers (IMEI, serial, MAC) in plaintext

**ASN lookup:**

```bash
# Check IP ownership
whois <IP> | grep -i "orgname\|country\|netname"
# Or use: https://ipinfo.io/<IP>  (offline: use MaxMind GeoLite2)
```

### 5.5 Protocol-Specific Analysis

#### DNS

```text
# Flag: DNS queries to non-configured DNS servers (DNS tunneling)
# Flag: Unusually long DNS query strings (>100 chars subdomain = possible tunneling)
# Flag: High-frequency DNS queries to same domain (beaconing via DNS)
# Flag: TXT record queries (common C2 channel)
```

#### TLS/HTTPS

```text
# Document all TLS certificate subjects and issuers
# Flag: Self-signed certificates to external hosts
# Flag: Certificates issued by Chinese CAs (CNNIC, WoSign, StartCom)
# Flag: Certificate transparency log absence for repeated connections
# Flag: Unusual cipher suites or TLS 1.0 usage
```

#### Non-HTTP Traffic

```text
# Flag: Any outbound SMTP/IMAP
# Flag: Any outbound FTP
# Flag: IRC-like protocols (port 6667, 6697)
# Flag: Connections on ports 4444, 5555, 31337 (common backdoor ports)
# Flag: ADB over TCP to external (port 5555 outbound)
```

---

## 6. Phase 3 — Deep Traffic Analysis (Weeks 2–8)

**Duration:** 6 weeks  
**Network:** Same isolated environment, but now with controlled internet access per approved domain list

### 6.1 Controlled Internet Access

Grant internet access domain by domain. For each domain:

1. Allow DNS resolution
2. Allow HTTPS connection
3. Capture and decrypt (where possible) full session
4. Document what data is sent and received
5. Decide: **Block** / **Allow** / **Allow with monitoring**

### 6.2 Periodic Trigger Testing

Some "calling home" behaviors trigger on specific events. Simulate:

| Trigger | How to Simulate | Watch For |
|---|---|---|
| Device power cycle | Reboot device | Burst of connections within first 60s |
| Time-based beacon | Leave running 24/7 for 4 weeks | Connections at same time daily/weekly |
| Network loss/recovery | Disconnect/reconnect ethernet | Re-registration attempts |
| Low storage event | Fill storage to >90% | Any telemetry spike |
| Date/time triggers | Set device clock to specific dates (e.g., Chinese holidays, anniversaries) | Behavioral change |
| Firmware update check | Observe OTA check intervals | What server is checked, what is sent |

### 6.3 OTA Update Analysis

**CRITICAL:** Never apply OTA updates without prior analysis.

- [ ] Capture the OTA check request — what device identifiers are sent?
- [ ] If an update is offered, download but DO NOT apply on a production device
- [ ] Analyze the update package on an isolated test device
- [ ] Verify update package signature — who signed it?
- [ ] `binwalk` + full analysis of update content
- [ ] After applying update on test device, repeat full traffic baseline (Phase 2)

### 6.4 Root Implementation Security Review

The device is rooted. Assess:

- [ ] Which root method is used (Magisk, SuperSU, KingRoot, manufacturer root, etc.)?
- [ ] Is the root implementation from the manufacturer? (Red flag — manufacturer-rooted devices may have backdoors baked in)
- [ ] Does any pre-installed app have root access without user prompt?
- [ ] Review Magisk module list if Magisk: `adb shell ls /data/adb/modules/`
- [ ] Verify su binary is not a trojanized version:

  ```bash
  adb pull /system/xbin/su ./su_binary
  sha256sum su_binary
  # Compare against known-good hash for your root method version
  ```

- [ ] Test: install a user app, verify it CANNOT get root without explicit grant

### 6.5 LAN Exposure Assessment

The device runs on a customer LAN. Assess what it exposes:

```bash
# Full port scan from another host on same VLAN
nmap -sV -p- -A <device_IP>

# UDP scan
nmap -sU --top-ports 200 <device_IP>

# Check for HTTP admin interfaces
curl -v http://<device_IP>/
curl -v http://<device_IP>:8080/
curl -v http://<device_IP>:8443/

# Test ADB over TCP
adb connect <device_IP>:5555

# Test default credentials if HTTP/SSH/Telnet found
```

- [ ] Document every open port and service
- [ ] Test all services for default credentials
- [ ] Test all HTTP admin interfaces for known vulnerabilities (directory traversal, auth bypass)
- [ ] Verify device does not respond to SNMP queries (community string `public`)

---

## 7. Phase 4 — CMS & Help Desk Integration Testing (Weeks 4–8)

**Duration:** 4 weeks, overlapping with Phase 3  
**Purpose:** Verify that YOUR software does not introduce or worsen security posture

### 7.1 Pre-Installation Baseline

- [ ] Capture a 48-hour traffic baseline BEFORE installing your CMS agent
- [ ] Document all open ports BEFORE installation
- [ ] Snapshot installed package list BEFORE installation

### 7.2 CMS Agent Installation

- [ ] Install CMS agent via your standard deployment method
- [ ] Record all changes to installed packages
- [ ] Record all new open ports
- [ ] Record all new services started

### 7.3 Post-Installation Comparison

- [ ] Compare traffic patterns — does CMS agent initiate expected connections only?
- [ ] Verify CMS agent uses TLS 1.2+ with valid certificates
- [ ] Verify CMS agent does not use credentials hardcoded in APK (decompile and check)
- [ ] Verify CMS communication endpoints are under your control (not third-party)
- [ ] Verify remote management commands cannot be injected by a third party
- [ ] Test: Can an unauthorized party send commands to the CMS agent?

### 7.4 Help Desk Remote Access Testing

- [ ] What protocol is used for remote access? (ADB, VPN tunnel, proprietary?)
- [ ] Is remote access always-on or on-demand?
- [ ] Test: Is remote access session logged and auditable?
- [ ] Test: Can a session be hijacked by a third party on the LAN?
- [ ] Verify remote access credentials are not shared across devices
- [ ] Verify remote access can be disabled per device

### 7.5 Credential & Secret Hygiene

- [ ] No hardcoded passwords in any config files pushed to device
- [ ] API keys stored in Android Keystore or equivalent secure storage
- [ ] Verify no credentials transmitted in URL parameters
- [ ] Verify no credentials logged in plaintext in logcat

---

## 8. Phase 5 — Long-Term Monitoring (Months 2–3+)

**Purpose:** Catch time-delayed or low-frequency beaconing, seasonal triggers, and dormant malware

### 8.1 Ongoing Automated Monitoring

Set up automated alerting for:

| Alert | Threshold | Action |
|---|---|---|
| New external IP contacted | Any new IP not in approved list | Immediate investigation |
| Data volume spike | >2x 7-day rolling average | Investigate within 4 hours |
| New DNS domain resolved | Any new domain | Log and review daily |
| New open port detected | Any port change | Immediate investigation |
| Certificate change on known host | Any | Investigate — possible MitM or server change |
| Device connects to non-approved DNS | Any | Block and alert |
| Outbound connection during maintenance window | Any | Investigate |

### 8.2 Weekly Reviews

- [ ] Review all new external IPs/domains contacted this week
- [ ] Review total data volume by destination — flag anomalies
- [ ] Review DNS query frequency distribution — flag high-frequency domains
- [ ] Review any security alerts generated
- [ ] Check for device firmware update checks — any new update available?
- [ ] Update approved allowlist with justified additions

### 8.3 Monthly Deep Dives

- [ ] Full `nmap` rescan — any new open ports vs. baseline?
- [ ] ADB package list diff — any new packages installed?
- [ ] ADB scheduled jobs diff — any new jobs registered?
- [ ] Review 30-day PCAP samples — do random 1-hour samples contain unexpected traffic?
- [ ] Review SIEM correlation rules — add new rules based on observed patterns
- [ ] Update threat intel feeds — check known malicious domains against DNS logs

### 8.4 Time-Trigger Testing

- [ ] Set device clock forward to: Chinese National Day (Oct 1), Chinese New Year, June 4 (historical significance) — observe behavior changes
- [ ] Set device clock to exactly 3-month, 6-month, 1-year marks — observe
- [ ] Simulate device being offline for 30 days, then reconnect — what does it immediately contact?

---

## 9. Phase 6 — Customer Network Simulation

**Purpose:** Test the device as an attacker would use it to pivot into a customer LAN

### 9.1 Network Isolation Validation

Simulate a typical customer network scenario:

```text
[Internet]
    |
[Customer Firewall/Router]
    |
[Customer LAN switch]
    |----[Digital Signage Device (DUT)]
    |----[Simulated Customer Workstation]
    |----[Simulated Customer File Server (Windows SMB)]
    |----[Simulated Customer IP Camera]
```

### 9.2 Lateral Movement Tests

From the device's perspective (assuming it IS compromised):

- [ ] Can the device scan the LAN? (`adb shell nmap` or built-in tools)
- [ ] Can the device reach customer workstations via SMB?
- [ ] Can the device enumerate NetBIOS / LDAP?
- [ ] Can the device reach management interfaces (router admin panel)?
- [ ] Does the device attempt to reach any of these on its own (without being instructed)?
- [ ] Can the device be used as a proxy/relay by an external attacker?

### 9.3 VLAN / Segmentation Recommendations for Customers

Based on findings, document:

- [ ] Recommended VLAN configuration for customers
- [ ] Required firewall rules (inbound and outbound) for device to function
- [ ] DNS allowlist for deployment
- [ ] Any ports that MUST be blocked at customer firewall

---

## 10. Risk Classification & Acceptance Criteria

### Risk Matrix

| Finding | Severity | Disposition |
|---|---|---|
| Active connection to Chinese ASN carrying device identifiers | CRITICAL | Do not deploy — reject device |
| Manufacturer CA certificate in system store | HIGH | Block cert + document |
| Pre-installed app with location/mic/camera permission + INTERNET | HIGH | Disable app + retest |
| ADB TCP open by default | HIGH | Must be disabled before deployment |
| Any open port with default credentials | HIGH | Must be remediated |
| Periodic beaconing to unknown host (no PII visible) | MEDIUM | Investigate + decide per case |
| OTA update check sends device serial | MEDIUM | Block OTA endpoint or control update path |
| Pre-installed adware-class APK without C2 | MEDIUM | Remove + document |
| mDNS/SSDP broadcasts on LAN | LOW | Document for customer |
| Unnecessarily open local ports (no default creds) | LOW | Document for customer |
| Google/standard Android telemetry only | ACCEPTED | Document + include in customer disclosure |

### Acceptance Criteria for Deployment

A device model/firmware version is approved for deployment ONLY when:

- [ ] No CRITICAL findings
- [ ] All HIGH findings remediated OR mitigated with documented compensating controls
- [ ] All MEDIUM findings triaged with documented disposition
- [ ] All LOW findings documented for customer disclosure
- [ ] Approved network allowlist finalized
- [ ] Customer deployment network configuration document complete
- [ ] Monitoring runbook for deployed devices complete

---

## 11. Reporting & Documentation

### 11.1 Per-Device Records

Maintain for each physical test device:

- `device-info.txt` — static device documentation (Section 3.4)
- `packages-YYYYMMDD.txt` — package list snapshots (diff weekly)
- `pcap/` — full packet captures (retain 90 days minimum, indefinitely for flagged sessions)
- `dns-log.csv` — all DNS queries with timestamp, domain, resolved IP
- `connections-log.csv` — all external connections: timestamp, dst IP, dst port, bytes, protocol
- `findings-log.md` — running log of all findings with severity and disposition
- `allowlist.txt` — approved domains/IPs with business justification

### 11.2 Findings Log Format

```markdown
## Finding #001
Date: YYYY-MM-DD
Severity: CRITICAL / HIGH / MEDIUM / LOW
Phase: 2
Description: Device contacted 114.113.x.x (ChinaNet AS4134) every 6 hours.
             Connection contained base64-encoded payload including device serial number.
Evidence: PCAP file: pcap/device01-20260418-0600.pcap, frame 4421
DNS: dns-log.csv line 892
Disposition: REJECT - device model not suitable for deployment
Remediation: N/A - reject
```

### 11.3 Customer Deployment Report

For approved devices, produce:

1. **Executive Summary** — risk level, what was tested, outcome
2. **Approved Network Configuration** — firewall rules, VLAN requirements, DNS allowlist
3. **Residual Risks** — accepted findings and why
4. **Monitoring Requirements** — what customer/NOC should log ongoing
5. **Update Policy** — how firmware updates are managed and re-tested

---

## 12. Appendices

### Appendix A — Known Suspicious Package Names

```text
com.adups.fota              # Adups spyware (backdoor found 2016)
com.adups.fota.sysoper
com.adsunflower             # Adware
com.android.provision       # Check if modified from AOSP
com.mediatek.mdmconfig      # MediaTek diagnostics - check permissions
com.mediatek.mtklogger      # MediaTek logger - disable
com.qualcomm.qti.qdss.agent # Qualcomm diagnostics
com.rockchip.update         # Rockchip OTA - monitor closely
com.allwinner.update        # Allwinner OTA - monitor closely
com.softwinner.update
tv.ouya.console             # If present unexpectedly
com.kubi                    # Known problematic in some devices
com.nq.android.antivirus    # Fake AV, known data harvester
```

### Appendix B — Known Chinese Telemetry / Analytics Domains

```text
# Baidu
bdimg.com, bdstatic.com, baidu.com, hm.baidu.com, pos.baidu.com

# Alibaba / Umeng analytics
umeng.com, umengcloud.com, amap.com, alicdn.com

# Tencent
bugly.qq.com, aegis.qq.com, beacon.qq.com, analytics.qq.com

# Xiaomi
tracking.miui.com, data.mistat.xiaomi.com, api.ad.xiaomi.com

# ByteDance / TikTok
rangersapplog.com, byteoversea.com, log.byteoversea.com

# Common Chinese ad networks
mintegral.com, mobvista.com, zplayads.com, yumimobi.com

# MediaTek / SoC vendors
log.mediatek.com, crash.mediatek.com

# Generic Chinese cloud
*.aliyuncs.com (distinguish from legitimate CDN use)
*.qcloud.com, *.myqcloud.com
```

### Appendix C — Recommended Firewall Egress Rules for Customer Deployment

```text
# ALLOW — Required for device function
ALLOW TCP 443 to <your_CMS_servers>
ALLOW TCP 443 to <your_helpdesk_servers>
ALLOW UDP 123 to <your_NTP_server>
ALLOW TCP/UDP 53 to <your_DNS_server>

# ALLOW — Standard Android/Google services (review per customer policy)
ALLOW TCP 443 to 142.250.0.0/15   # Google
ALLOW TCP 443 to 172.217.0.0/16   # Google
ALLOW TCP 443 to 74.125.0.0/16    # Google Play, GCM

# DENY — High risk, no legitimate signage purpose
DENY ALL to 0.0.0.0/0 from port 5555   # ADB TCP
DENY ALL to CN (country block)          # Block Chinese IP ranges
DENY ALL to HK (review per policy)      # Hong Kong — review
DENY TCP 23                              # Telnet
DENY TCP 21                              # FTP outbound

# DENY — By default, deny all other egress
DENY ALL (default deny, log all hits)
```

### Appendix D — Tool Reference

| Tool | Purpose | Install |
|---|---|---|
| Wireshark | Packet analysis | wireshark.org |
| tcpdump | CLI packet capture | Linux package |
| Zeek | Network visibility platform | zeek.org |
| mitmproxy | HTTPS interception | mitmproxy.org |
| Pi-hole | DNS sinkhole | pi-hole.net |
| nmap | Port scanning | nmap.org |
| apktool | APK decompilation | apktool.ibotpeaches.me |
| jadx | APK decompile to Java | github.com/skylot/jadx |
| MobSF | Automated mobile app analysis | mobsf.github.io (Docker) |
| ADB | Android debug bridge | Android SDK Platform Tools |
| binwalk | Firmware analysis | github.com/ReFirmLabs/binwalk |
| MaxMind GeoLite2 | Offline IP geolocation | maxmind.com |
| ELK Stack | Log aggregation / SIEM | elastic.co |

### Appendix E — Relevant Standards & References

- NIST SP 800-163: Vetting the Security of Mobile Applications
- ENISA: Good Practices for Security of IoT
- OWASP Mobile Security Testing Guide (MASTG)
- BSI TR-03161: Security Requirements for Digital Signage
- Adups spyware disclosure (BLU phones, 2016) — case study reference
- CISA Advisory AA22-257A: Iranian Islamic Revolutionary Guard Corp (supply chain)
- Supply chain security: NIST SP 800-161r1

---

*Document maintained by: [Security Team / NOC]*  
*Next scheduled review: [Date of next batch shipment]*  
*Approval required from: [Security Lead, Operations Lead]*
