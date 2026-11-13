## ⚡ 4. Fast Roaming Architecture — Reality vs. Original Design

- **802.11k (RRM)**: Genuinely enabled and fully reciprocal across OpenWrt nodes (Jio AX6000 and TP-Link Archer A6) and seeded with accurate vendor AP channels. Native hostapd neighbor reports are statically seeded at boot via `/usr/bin/seed-rrm.uc` and monitored/refreshed dynamically by `ensureRrmNeighbors()` in `mesh-poller.uc`.
- **802.11k Channel & Sub-element Accuracy (Fixed & Verified 2026-11-12)**:
  - Previously, neighbor tables contained stale/invalid channel mappings (e.g. reporting TP-Link on Ch 36 instead of Ch 44, AP #1 on Ch 149 instead of Ch 36, and 2.4 GHz neighbors as SSID `U101-slow` instead of `U101`), which caused 802.11k directed probe scans to fail silently.
  - Symmetrical 802.11k Neighbor Profiles currently enforced:
    - **Jio AX6000 5G**: BSSID `8c:83:94:42:a6:d9`, Ch 149 HE80 (OpClass 128, Center 155, PHY 0x0e).
    - **Jio AX6000 2.4G**: BSSID `8c:83:94:42:a6:d8`, Ch 1 HE20 (OpClass 81, PHY 0x0e).
    - **TP-Link Archer A6 5G**: BSSID `10:27:f5:b0:b2:0a`, Ch 44 VHT80 (OpClass 128, Center 42, PHY 0x09).
    - **TP-Link Archer A6 2.4G**: BSSID `10:27:f5:b0:b2:09`, Ch 6 HT20 (OpClass 81, PHY 0x07).
    - **Huawei AP #1 5G**: BSSID `48:f8:db:6d:98:74`, Ch 36 VHT80 (OpClass 128, Center 42, PHY 0x09).
    - **Huawei AP #1 2.4G**: BSSID `48:f8:db:6d:98:70`, Ch 11 HT20 (OpClass 81, PHY 0x07).
    - **Nokia ONT 5G**: BSSID `e4:da:df:83:8e:ad`, Ch 44 VHT80 (OpClass 128, Center 42, PHY 0x09).
    - **Nokia ONT 2.4G**: BSSID `e4:da:df:83:8e:a9`, Ch 11 HT20 (OpClass 81, PHY 0x07, SSID `U101-slow`).
  - Added a 30-second ubus retry loop in TP-Link's `/usr/bin/seed-rrm.uc` so neighbor tables never fail to load if hostapd starts after `rc.local`.
- **802.11r (FT - Fast BSS Transition)**: Enabled on both OpenWrt APs (`default_radio0` and `default_radio1` on Jio AX6000 and TP-Link Archer A6) with shared mobility domain `e101` and `ft_psk_generate_local 1`, enabling zero-reauth L2 handoffs between Jio and TP-Link without full 4-way WPA handshake.
- **802.11w (PMF)**: Genuinely enabled (`Enable`, confirmed via `display wifi radio`) on Huawei AP #1. TP-Link and Jio AX6000 UCI configs set `ieee80211w=0` for mixed compatibility.
- **802.11v (BSS Transition) is NOT functionally supported on TP-Link.** Its hostapd build rejects the `bss_transition` config directive outright (`Line 47: unknown configuration item 'bss_transition'`) — this bug previously took down TP-Link's entire Wi-Fi stack. Kept removed from `/etc/config/wireless`.
- **usteer is completely disabled.** It was previously overwriting hostapd's 802.11k neighbor reports with only local interfaces. It remains disabled in favor of native hostapd RRM tables.
- **Autonomous Roaming Watchdog (`roam-daemon.uc` & `mesh-poller.uc`)**:
  - **Problem Solved**: Eliminates Apple/Android "sticky client syndrome" where devices cling to distant APs (e.g. phone trapped at `-89 dBm` on Jio while standing right next to TP-Link).
  - **Runtime & Execution**: Native ucode script at `/usr/bin/roam-daemon.uc`, called every polling cycle (~20s) by `mesh-poller.uc`.
  - **Thresholds & Safety Rules (Updated 2026-11-12)**:
    - 5 GHz threshold = **`-76 dBm`** (tuned for Apple/Android native scan onset).
    - 2.4 GHz threshold = **`-80 dBm`** (edge of usable SNR).
    - Cooldown = **`180s`** (3 minutes per client to prevent thrashing).
    - Consecutive degraded cycles required = **`2`** (~40s of sustained low RSSI before action).
    - Deauth Reason Code: Reason 3 (`DEAUTH_LEAVING`) instead of Reason 5 (`AP_BUSY`), preventing client-side AP blacklisting.
    - Stationary Whitelist: Smart TVs (Mi TV 75", Fire TV Stick), Panasonic Smart AC, PC, HP Envy x360 laptop, and IoT devices are excluded from roam kicking.
  - **Verified Result**: Lava Agni 2 phone was successfully roamed from a degraded link on Jio (-89 dBm, 6 Mbps) to TP-Link 5 GHz (-57 dBm, 780 Mbps, SNR 35).
- **API & CLI Control**:
  - CLI: `/usr/bin/roam-daemon.uc --status`, `--evaluate`, `--kick <mac> [ap] [band]`, `--json`.
  - Web API: `/cgi-bin/mesh-api?action=roam_status` and `/cgi-bin/mesh-api?action=roam_kick&mac=<mac>&ap=<ap>&band=<band>`.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
