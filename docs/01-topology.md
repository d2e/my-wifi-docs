## 🗺️ 1. Real Physical Topology (corrected — NOT a flat star)

The network is **not** a single flat star from one switch. Confirmed physical wiring:

```
Fiber WAN (ISP)
      │
  Nokia Gateway (192.168.0.1) — Master DHCP, WAN ONT
   ┌──┴──────────────┐
   │                 │
Huawei AP #1      Huawei AP #2 (192.168.0.98)
(192.168.0.110)        │
  direct to Nokia   TP-Link (192.168.0.12)
                     chained through AP #2's LAN port,
                     NOT directly to Nokia
```

**Practical implication**: Huawei AP #2 is a single point of failure for TP-Link's uplink. If AP #2 loses power or its Ethernet port fails, TP-Link (and everything hosted on it — the mesh dashboard, DNS, usteer) loses its path to Nokia and the internet too, even though TP-Link itself keeps running.

This also explains a real, measured, structural latency difference: pings from TP-Link (where the polling daemon runs) to each node cross a different number of Ethernet switch/bridge hops (invisible to `traceroute`/TTL, which only sees L3 router hops — there are zero of those anywhere in this network since it's all one subnet):

| Target | Path from TP-Link | Switch hops | Typical latency |
|---|---|---|---|
| Huawei AP #2 | direct cable | 0 | ~0.9-1.1 ms |
| Nokia | via AP #2's switch | 1 | ~1.0-1.1 ms |
| Huawei AP #1 | via AP #2's switch, then Nokia's switch | 2 | ~1.2-1.6 ms |

All figures are sub-2ms and not a real problem — documented here only because the pattern is consistent and previously misunderstood.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
