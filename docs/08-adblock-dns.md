## 🛡️ 8. Whole-Home Ad-Blocking & DNS Architecture

As of 2026-10-31, network-wide DNS filtering is active in a high-availability primary/secondary configuration hosted on the OpenWrt nodes:

### Primary DNS: JioRouter AX6000 (`192.168.0.50`)
- **Engine:** **`adblock-fast` (v1.2.4)** + **`dnsmasq`**
- **Domain Blocklist Active:** **28,205 advertising, telemetry, and malware domains** blocked at DNS layer (`NXDOMAIN`).
- **Curated Sources:** AdAway Hosts, SomeoneWhoCares Hosts, AdGuard CNAME Ads, Yoyo.org, NoCoin List, plus custom telemetry exclusions.
- **Upstream Forwarders:** `1.1.1.1`, `1.0.0.1`, `8.8.8.8`, `8.8.4.4` (with `option noresolv '1'` to prevent DHCP DNS loops).
- **Resource Footprint:** Negligible on Filogic 830 (332 MB RAM free out of 512 MB).

### Secondary DNS: TP-Link Archer A6 (`192.168.0.12`)
- **Engine:** **`adblock-fast` (v1.1.2)** + **`dnsmasq`**
- **Domain Blocklist Active:** **27,472 advertising, telemetry, and malware domains** blocked at DNS layer.
- **Redundancy Role:** High-availability secondary fallback if Jio is rebooting or updating.

### Nokia Master DHCP Integration
- **Nokia Master Gateway (`192.168.0.1`)**:
  - `firstDns`: **`192.168.0.50`** (Primary)
  - `secondDns`: **`192.168.0.12`** (Secondary)
  - Configured via: `node nokia_wifi_control.js --set-dns 192.168.0.50 192.168.0.12`
  - Advertised automatically across all DHCP clients on the mesh network.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
