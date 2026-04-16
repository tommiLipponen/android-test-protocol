# Traffic Analysis Daily/Weekly Checklist

**Purpose:** Quick-reference checklist for analysts reviewing captured traffic from test devices

---

## Color Codes

- 🔴 **STOP** — Critical finding, halt deployment, escalate immediately
- 🟠 **WARN** — High severity, investigate within 24 hours
- 🟡 **NOTE** — Medium severity, investigate within 1 week
- 🟢 **OK** — Expected / acceptable
- ⚪ **LOG** — Log and monitor, no immediate action

---

## Daily Checklist

### DNS Layer

- [ ] Are all DNS queries going to the configured sinkhole/resolver only?
  - Any queries to other IPs: 🟠 DNS hijacking or hardcoded resolver
- [ ] Any DNS query with subdomain length > 100 characters?
  - Yes: 🔴 Possible DNS tunneling
- [ ] Any TXT record queries to external domains?
  - Yes: 🟠 Possible C2 command channel
- [ ] High frequency queries (>10/min) to same domain?
  - Yes: 🟡 Possible beaconing or DNS tunneling
- [ ] Any `.cn` domain queries?
  - Yes: 🟠 Investigate immediately
- [ ] New domains not seen before?
  - Yes: ⚪ Log and review weekly

### IP / Connection Layer

- [ ] Any connection to Chinese ASNs (see main protocol Appendix B)?
  - Yes: 🔴 Critical — document exact payload if possible
- [ ] Any connection on port 5555 (ADB TCP)?
  - Yes: 🔴 Remote ADB access — reject device
- [ ] Any connection on port 23 (Telnet) or port 22 (SSH) outbound?
  - Yes: 🔴 Command channel
- [ ] Any connection on ports 4444, 6667, 31337, 1337?
  - Yes: 🔴 Known backdoor/C2 ports
- [ ] Any raw TCP connection (non-HTTPS) to external IP?
  - Yes: 🟠 Investigate — what data is sent in plaintext?
- [ ] Any connection to IP without prior DNS resolution (hardcoded IP)?
  - Yes: 🟠 Hardcoded C2 or update server — whois lookup immediately
- [ ] Total outbound bytes today vs. yesterday — >50% increase?
  - Yes: 🟡 Investigate cause
- [ ] Any ICMP to external hosts?
  - Yes: 🟡 Unusual — document

### TLS / HTTPS Layer

- [ ] Any TLS handshake to external host failing cert validation?
  - Yes: 🟡 Investigate — possible certificate pinning or self-signed C2
- [ ] Any certificate issued by CNNIC, WoSign, StartCom, or unknown Chinese CA?
  - Yes: 🔴 Rogue certificate authority
- [ ] Any TLS 1.0 or SSL 3.0 connection?
  - Yes: 🟡 Weak protocol — document
- [ ] Any new TLS certificate subject not seen before?
  - Yes: ⚪ Log — check CT log, whois registrar

### LAN / Local Layer

- [ ] Any ARP requests to IPs outside assigned subnet?
  - Yes: 🟡 Possible ARP scanning
- [ ] Excessive mDNS/SSDP broadcasts?
  - Yes: 🟡 Document — inform customer of LAN noise
- [ ] Any LLMNR broadcasts?
  - Yes: 🟡 Document — LLMNR can be abused by attackers on LAN
- [ ] Any attempted connection to other hosts on the LAN test segment?
  - Yes: 🟠 Lateral movement attempt

---

## Weekly Analysis

### Volume & Behavior Trends

- [ ] Plot hourly outbound traffic volume for the week — any regular spikes at fixed times?
  - Yes: 🟠 Time-based beaconing — find the exact time and correlate with destination
- [ ] Total unique external IPs this week vs. last week — significant increase?
  - Yes: 🟡 Investigate new destinations
- [ ] Compare DNS query count per domain — top 10 most queried domains expected?
  - Unexpected top domains: 🟠 Investigate

### Package & Service Drift

```bash
# Run and diff against previous week
adb shell pm list packages -f > packages_$(date +%Y%m%d).txt
diff packages_LASTWEEK.txt packages_$(date +%Y%m%d).txt

adb shell dumpsys jobscheduler > jobs_$(date +%Y%m%d).txt
diff jobs_LASTWEEK.txt jobs_$(date +%Y%m%d).txt

adb shell ss -tlnp > ports_$(date +%Y%m%d).txt
diff ports_LASTWEEK.txt ports_$(date +%Y%m%d).txt
```

- [ ] Any new packages installed since last week?
  - Yes: 🔴 Unless via your own CMS push — reject/investigate
- [ ] Any new scheduled jobs registered?
  - Yes: 🟠 Investigate — possible self-updating malware
- [ ] Any new listening ports?
  - Yes: 🟠 Investigate immediately

### OTA / Update Activity

- [ ] Did device query an OTA endpoint this week?
  - Log the full request including headers and POST body
  - What device identifiers were sent? (serial, IMEI, model): 🟡 Document
  - Was update offered? Do NOT apply without analysis.
- [ ] Did device download any file > 1MB?
  - Yes: 🟠 Capture the file, analyze before allowing

---

## Quick Reference: Payload Inspection

### Capturing a Specific Connection for Analysis

```bash
# In Wireshark: filter to device IP
ip.addr == <device_ip>

# Filter to specific destination
ip.addr == <device_ip> && ip.dst == <suspicious_ip>

# Extract HTTP/HTTPS stream: Right-click → Follow → TLS Stream (if decrypted by proxy)

# Export objects: File → Export Objects → HTTP
```

### Decoding Common Obfuscation in Traffic

```bash
# Base64 decode a suspicious string
echo "c3VzcGljaW91cw==" | base64 -d

# Check if payload is gzip compressed
echo "<hex_payload>" | xxd -r -p | file -

# XOR decode (if key is known from APK analysis)
python3 -c "
data = bytes.fromhex('<hex_payload>')
key = 0x42  # example
print(bytes([b ^ key for b in data]))
"
```

### Checking If Data Contains Device Identifiers

Search captured traffic for:

```text
<serial_number>
<MAC_address_no_colons>
<MAC_address_with_colons>
<IMEI>
<Android_ID>  # get with: adb shell settings get secure android_id
```

In pcap:

```bash
# Search PCAP for device serial
tshark -r capture.pcap -Y 'frame contains "<SERIAL>"'
```

---

## Escalation Contacts

| Severity | Escalate To | SLA |
|---|---|---|
| 🔴 CRITICAL | Security Lead + Management | Immediate |
| 🟠 HIGH | Security Lead | Within 4 hours |
| 🟡 MEDIUM | Security Analyst | Within 48 hours |
| ⚪ LOG | Weekly review meeting | Weekly |
