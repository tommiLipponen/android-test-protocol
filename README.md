# Android Digital Signage — Security Test Protocol

A structured security evaluation protocol for Chinese-manufactured, rooted Android digital signage devices. The goal is to detect "calling home" behaviour, unauthorized data exfiltration, and other supply-chain security risks before or during deployment in a corporate environment.

---

## Context

- Devices are **Ethernet-only** (no WiFi), running **rooted Android** on Rockchip/Allwinner/MediaTek SoCs
- Testing is conducted across two sites:
  - **Corporate office showroom** — full device fleet, corporate firewall provides independent log source
  - **Home lab** — Linux gateway + 4G router, used for deep per-device traffic analysis
- All monitoring is **passive** (no traffic blocking during the test phase)

---

## Documents

| File | Purpose |
|------|---------|
| [TEST-PROTOCOL.md](TEST-PROTOCOL.md) | Main 6-phase test protocol — threat model, test procedures, risk matrix, appendices |
| [LAB-SETUP-GUIDE.md](LAB-SETUP-GUIDE.md) | Step-by-step Linux gateway setup (nftables, mitmproxy, Zeek, tcpdump, dnsmasq) |
| [TRAFFIC-ANALYSIS-CHECKLIST.md](TRAFFIC-ANALYSIS-CHECKLIST.md) | Daily/weekly analyst checklist with severity codes |
| [DEVICE-TEST-LOG-TEMPLATE.md](DEVICE-TEST-LOG-TEMPLATE.md) | Per-device tracking log — copy one per device under test |

---

## Test Phases Summary

| Phase | Duration | Focus |
|-------|----------|-------|
| 1 — Baseline & Inventory | Week 1–2 | Physical inspection, firmware fingerprint, ADB enumeration |
| 2 — Network Behaviour | Week 2–4 | Traffic capture, DNS analysis, TLS certificate inspection |
| 3 — Static Analysis | Week 3–5 | APK decompilation, pre-installed app review, permission audit |
| 4 — Dynamic Analysis | Week 4–6 | Live traffic correlation, mitmproxy interception, Zeek alerts |
| 5 — Long-term Monitoring | Month 2–3 | Scheduled/overnight captures, job scheduler inspection, OTA watch |
| 6 — Reporting | Month 3 | Risk matrix, findings, recommendations |

---

## Lab Architecture

```text
  [DUT] ──Ethernet──▶ enp2s0 [LINUX GATEWAY] enp3s0 ──Ethernet──▶ [4G ROUTER] ──▶ Internet
```

All device traffic passes through the Linux gateway. Tools used:

- **Zeek** — live protocol analysis and connection summaries
- **tcpdump** — rotating full PCAP capture
- **mitmproxy** — transparent HTTPS interception (where TLS can be intercepted)
- **dnsmasq** — DHCP server + DNS logging
- **nftables** — NAT + per-connection logging (monitor-only, policy accept)
- **Wireshark** — on-demand deep packet inspection

---

## Key Threat Areas

- Beaconing to Chinese CDN/cloud infrastructure (Alibaba, Tencent, ByteDance)
- Hardcoded DNS bypass (device ignoring DHCP-assigned resolver)
- Pre-installed spyware or adware SDKs (Adups, Umeng, Mintegral)
- Rogue CA certificates in system trust store
- Encrypted C2 channels using legitimate cloud services as cover
- OTA update mechanism pulling unsigned or unverified firmware
- USB tethering as a secondary egress path bypassing gateway monitoring

---

## Requirements

### Linux Gateway Host

- Ubuntu 24.04 LTS or Debian 12
- Two NICs: `enp2s0` (DUT-facing, 10.99.1.1/24) and `enp3s0` (WAN)
- Packages: `nftables`, `dnsmasq`, `tcpdump`, `mitmproxy`, `zeek`, `wireshark`

### Analyst Workstation

- Wireshark, `tshark`, `zeek-cut`
- Python 3 (for annotation scripts and log parsing)
- `adb`, `apktool`, `jadx`, `MobSF` (optional, for static analysis phases)

---

## Configuration Files

| File | Purpose |
|------|---------|
| [.markdownlint.json](.markdownlint.json) | Disables MD013 (line length) and MD060 (table style) for technical docs |
| [cspell.json](cspell.json) | Whitelists technical vocabulary for VS Code spell checker |

---

## License

This project is for internal security research and evaluation purposes.
