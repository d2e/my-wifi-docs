# Whole-Home Gigabit Multi-AP Wi-Fi Mesh & Automation — Architecture & Design Reference

This repository contains the complete **system architecture, physical & logical network design, RF spectrum plans, hardware matrix, and operational documentation** for the whole-home multi-AP Wi-Fi 6 mesh and home automation ecosystem.

> **Note on Accuracy**: All documentation in this repository reflects the **actual verified state** of the network, confirmed by direct live hardware inspection.

---

## 🏛️ System Overview & Topology

The network operates on a single flat L3 subnet (`192.168.0.0/24`) with zero internal routing hops below the fiber master gateway. 

```
                            [ ISP Fiber WAN ]
                                    │
                                    │ GPON Optical Fiber
                                    ▼
       ┌─────────────────────────────────────────────────────────┐
       │               Nokia Gateway (G-2425G-A)                 │
       │                     192.168.0.1                         │
       │       Master DHCP Server (Pool: 192.168.0.2 - .254)     │
       │       Advertises DNS: Primary .50  │  Secondary .12     │
       └───┬─────────────────────────────────────────────────┬───┘
           │ Cat6 (LAN 1)                                    │ Cat6 (LAN 2)
           ▼                                                 ▼
┌───────────────────────────────────────┐         ┌──────────────────────┐
│ JioRouter AX6000 (JIDU6101)           │         │  Huawei AP #2        │
│ 192.168.0.50                          │         │  HG8145V5            │
│                                       │         │  192.168.0.98        │
│ ★ PRIMARY AUTOMATION HUB              │         │                      │
│ • Replaced Huawei AP #1 (deprecated)  │         │ • Pure L2 AP         │
│ • Wi-Fi 6 (802.11ax 4x4 MU-MIMO 160M) │         │ • Daisy-chain uplink │
│ • OpenWrt SNAPSHOT (Quad-core ARM64)  │         │ • Real Busybox shell │
│ • Primary DNS Sinkhole (adblock-fast) │         └──────────┬───────────┘
│ • USB 3.0 Media Host Attached         │                    │ Cat6 LAN
│   (/mnt/sda1 ext4 28GB)               │                    ▼
│   - transmission-daemon (:9091)       │         ┌────────────────────┐
│   - MiniDLNA 1.3.3 (:8200)            │         │ TP-Link Archer A6  │
└───────────────────────────────────────┘         │ v3 (192.168.0.12)  │
[Note: Huawei AP #1 (.110) decommissioned]        │                    │
                                                  │ • SECONDARY AP     │
                                                  │ • OpenWrt 23.05.5  │
                                                  │ • Fallback DNS     │
                                                  │ • Web redirects    │
                                                  │   to Jio .50       │
                                                  └────────────────────┘
                                              ⚠ AP #2 is single point of failure
                                                for TP-Link's network path
```

---

## 📊 Visual Design & Architecture Artifacts

| Artifact | Format | Description |
|---|---|---|
| [`docs/diagrams/HLD-whole-architecture.drawio`](docs/diagrams/HLD-whole-architecture.drawio) | Draw.io XML | **Page 1 (One-Slide)**: High-level 15-box system overview.<br>**Page 2 (Detail)**: Component links, daemons, state files, and invariants. |
| [`docs/diagrams/HLD-whole-architecture-HLD detail - context & flows.drawio.png`](docs/diagrams/HLD-whole-architecture-HLD%20detail%20-%20context%20&%20flows.drawio.png) | Rendered PNG | Full rendered visual architecture diagram. |
| [`docs/diagrams/LLD-whole-architecture.drawio`](docs/diagrams/LLD-whole-architecture.drawio) | Draw.io XML | 5-page low-level design deck: L1/L2 topology, polling cycle, CEC control plane, AC MirAIe MQTT, and deployment map. |
| [`docs/presentation/architecture.html`](docs/presentation/architecture.html) | Interactive HTML5 Deck | Animated 13-slide visual walkthrough (open in any web browser). Keys: <kbd>→</kbd> next build/slide, <kbd>N</kbd> notes, <kbd>F</kbd> fullscreen. |

---

## 🧭 Numbered Documentation Index

| § | Section Document | Description |
|---|---|---|
| 1 | [`docs/01-topology.md`](docs/01-topology.md) | Physical Cat6 wiring, switch hops, structural latency profile, AP #2 daisy-chain gotcha |
| 2 | [`docs/02-hardware-matrix.md`](docs/02-hardware-matrix.md) | Full hardware specifications, node management IPs, access methods, MAC identification table |
| 3 | [`docs/03-rf-spectrum.md`](docs/03-rf-spectrum.md) | 5GHz and 2.4GHz non-overlapping channel allocations, auto-channel healing, encryption modes |
| 4 | [`docs/04-roaming.md`](docs/04-roaming.md) | 802.11k/v/r fast transition & neighbor reports, autonomous roaming watchdog |
| 5 | [`docs/05-mesh-web-app.md`](docs/05-mesh-web-app.md) | Architecture of the single pane-of-glass dashboard (`mesh-poller` → `mesh-api` → `mesh.html`) |
| 6 | [`docs/06-vendor-cli.md`](docs/06-vendor-cli.md) | Huawei SU_WAP and Nokia Zebra-VTY vendor CLI commands, telnet concurrency limits |
| 7 | [`docs/07-device-policy.md`](docs/07-device-policy.md) | Network-wide MAC access filtering, DNS blackholing, Mi TV YouTube/SmartTube native app lock |
| 8 | [`docs/08-adblock-dns.md`](docs/08-adblock-dns.md) | High-availability dual-node `adblock-fast` + `dnsmasq` whole-home sinkhole (28k+ domains) |
| 9 | [`docs/09-kernel-tuning.md`](docs/09-kernel-tuning.md) | Linux kernel network profile: BDP buffer sizing, TCP Fast Open, SMP IRQ affinity, 4-core RPS steering |
| 10 | [`docs/10-cli-tools.md`](docs/10-cli-tools.md) | Copy-pasteable administrative commands: node access, RF discovery optimizer, poller control |
| 11 | [`docs/11-file-locations.md`](docs/11-file-locations.md) | Filesystem directory map for both OpenWrt hubs (JioRouter AX6000 & TP-Link Archer A6) |
| 12 | [`docs/12-panasonic-ac.md`](docs/12-panasonic-ac.md) | Panasonic MirAIe MQTT-over-TLS control plane, energy ledger, and thermodynamic comfort prediction models |
| 13 | [`docs/13-entertainment.md`](docs/13-entertainment.md) | Home theater stack: Xiaomi Mi QLED 75, Fire TV 4K, Sony HT-S40R 5.1 ARC/CEC control, PQ preset calibration |
| 14 | [`docs/14-dashboard-audit.md`](docs/14-dashboard-audit.md) | Backend reliability audit: silent-failure bug classes, injection prevention, regression testing tiers |
| 15 | [`docs/15-tv-companion-agent.md`](docs/15-tv-companion-agent.md) | Native TV companion micro-agent (`com.home.tvagent`), OTT metadata extraction, Shizuku lifecycle |
| 16 | [`docs/16-ap2-usb-and-poller.md`](docs/16-ap2-usb-and-poller.md) | Huawei AP #2 USB media hub (Transmission torrent RPC, MiniDLNA UPnP server, storage safety) |

### Companion References
- [`AUDIO.md`](AUDIO.md) — Home-theater audio operations, ARC/eARC constraints, Bitstream vs. PCM routing, and format decoding.
- [`mitv_hidden_services_and_hardware_matrix.md`](mitv_hidden_services_and_hardware_matrix.md) — Xiaomi Mi TV MT9611 register map, hidden Android services, and vendor properties.

---

## 📇 Hardware Quick Reference

| Node | Model | IP | Role |
|---|---|---|---|
| **Nokia G-2425G-A** | MTK7528 GPON ONT | `192.168.0.1` | Fiber WAN Gateway, Master DHCP Server, 5G Ch 44 / 2.4G Ch 11 |
| **JioRouter AX6000 (JIDU6101)** | MT7986 ARM64 4x4 Wi-Fi 6 | `192.168.0.50` | Primary Automation Hub, AP #1 replacement, Native USB 3.0 Media Host (:9091 Transmission, :8200 MiniDLNA), Primary DNS (`adblock-fast`), 5G Ch 36 HE160 |
| **Huawei HG8145V5 (AP #2)** | Dual-band AC1200 GPON | `192.168.0.98` | Secondary L2 Access Point, daisy-chains TP-Link, 5G Ch 52 DFS / 2.4G Ch 11 |
| **TP-Link Archer A6 v3** | MediaTek MT7621AT | `192.168.0.12` | Secondary Access Point, Secondary DNS (`adblock-fast`), 5G Ch 36 / 2.4G Ch 6 |
| **Huawei HG8145V5 (AP #1)** | Dual-band AC1200 GPON | `192.168.0.110` | **[DEPRECATED / DECOMMISSIONED]** Replaced by JioRouter AX6000 |
| **Xiaomi Mi QLED 75** | MT9611 Android 11 | `192.168.0.10` | CEC Bus Master (0.0.0.0), Native Rust agent (:9988), Telemetry agent (:8080) |
| **Amazon Fire TV 4K** | AFTMM Fire OS 6 | `192.168.0.7` | HDMI 1 Input, locked 4K@60, Dolby Digital Plus bitstream |
| **Sony HT-S40R 5.1** | 600W S-Master Soundbar | CEC `2.0.0.0` | HDMI 2 ARC audio sink |
| **Panasonic Smart AC** | MirAIe Wi-Fi Hub | `192.168.0.4` | Controlled via MirAIe Cloud MQTT-over-TLS (:8883) |
