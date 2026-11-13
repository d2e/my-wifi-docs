## 🗺️ 1. Real Physical Topology (corrected — NOT a flat star)

The network is **not** a single flat star from one switch. Confirmed physical wiring:

```
Fiber WAN (ISP)
      │
  Nokia Gateway (192.168.0.1) — Master DHCP, WAN ONT
   ┌──┴────────────────────────────────┐
   │                                   │
JioRouter AX6000 (192.168.0.50)      Huawei AP #2 (192.168.0.98)
  - Direct uplink to Nokia             │
  - Replaces Huawei AP #1 (deprecated) TP-Link Archer A6 (192.168.0.12)
  - Primary OpenWrt Automation Hub      chained through AP #2's LAN port,
  - [USB Storage Attached]             NOT directly to Nokia
    (/dev/sda1 -> /mnt/sda1)
    (Transmission + MiniDLNA)
```

> **Hardware Change**: **Huawei AP #1 (`192.168.0.110`) has been DEPRECATED and decommissioned.** It has been completely replaced by the **JioRouter AX6000 (`192.168.0.50`)**, which connects directly to the Nokia Master Gateway via Gigabit Ethernet. Furthermore, the **USB flash drive** hosting the media stack (`transmission-daemon` BitTorrent and `minidlnad` DLNA media server) is now plugged directly into the JioRouter AX6000 at `/mnt/sda1` (`/mnt/usb`).

**Practical implication**: Huawei AP #2 remains a single point of failure for TP-Link's uplink. If AP #2 loses power or its Ethernet port fails, TP-Link loses its path to Nokia and the rest of the network. The JioRouter AX6000 is directly uplinked to Nokia and independent of AP #2.

Switch hop and latency reference (measured from JioRouter `192.168.0.50` where the central poller runs):

| Target | Path from JioRouter | Switch hops | Typical latency |
|---|---|---|---|
| Nokia Gateway | direct cable via Nokia LAN switch | 1 | ~0.8-1.0 ms |
| Huawei AP #2 | via Nokia switch | 2 | ~1.0-1.2 ms |
| TP-Link Archer A6 | via Nokia switch, then AP #2 switch | 3 | ~1.1-1.3 ms |
| Huawei AP #1 | *DEPRECATED / DECOMMISSIONED* | - | N/A |

All active nodes maintain sub-1.5ms L2 bridge latencies across the home.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
