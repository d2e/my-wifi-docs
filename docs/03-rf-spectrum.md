## 📡 3. RF Spectrum — Reality vs. Original Design

The active network consists of four nodes following the deprecation of Huawei AP #1: JioRouter AX6000 (primary Wi-Fi 6 hub), Huawei AP #2, TP-Link Archer A6, and the Nokia Master Gateway.

| Node | 5GHz | 2.4GHz | Channel mode | How it's fixed |
|---|---|---|---|---|
| Jio AX6000 (Primary Hub) | **Ch 36 (HE160, 160MHz width, center Ch 50 @ 5250MHz)** | **Ch 1, fixed, HE20 (20MHz)** | Static | `uci`/OpenWrt wireless config (Wi-Fi 6 802.11ax, 4x4 MU-MIMO, 160MHz ultra-wideband, 30 dBm txpower) |
| Huawei AP #2 | **Ch 52 (DFS), fixed, 80MHz** | **Ch 11, fixed, HT20 (20MHz)** | Static (poller self-healed) | Vendor CLI (`set wlan basic laninst 1 wlaninst 1 Channel 11` + `set wifi expert band b bandwidth 20M pmf 1 txbf 1 scs 0 guardinterval short`) |
| TP-Link Archer A6 | **Ch 36, fixed, VHT80 (80MHz)** | **Ch 6, fixed, HT20 (20MHz)** | Static | `uci`/OpenWrt wireless config (`network.lan`-adjacent wireless config, standard OpenWrt) |
| Nokia Gateway | **Ch 44, fixed, VHT80 (80MHz)** | **Ch 11, fixed, HT20 (20MHz)** | Static | Encrypted Web API (`nokia_wifi_control.js` / `wlan_config.cgi`) |
| Huawei AP #1 | *DEPRECATED* | *DEPRECATED* | Offline | Decommissioned; replaced by Jio AX6000 |

**Collision status & Spectrum Plan:**
- **5 GHz Spectrum**: Jio=Ch 36 (HE160, 160MHz width, center Ch 50), TP-Link=Ch 36 (VHT80), AP#2=Ch 52 (DFS, 80MHz), Nokia=Ch 44 (VHT80). Non-DFS primary channel 36 ensures zero radar CAC boot delays for immediate client association.
- **2.4 GHz Spectrum**: Clean non-overlapping standard channel plan: Jio=Ch 1 (HE20), TP-Link=Ch 6 (HT20), AP#2=Ch 11 (HT20), Nokia=Ch 11 (HT20). Co-channel reuse (Ch 11) is assigned to Nokia and AP#2 because they have the greatest physical distance and RF attenuation (-81 to -89 dBm isolation through multiple concrete walls, well below the carrier sense threshold). All 2.4GHz radios are locked to **20 MHz width** to eliminate adjacent channel interference and 40MHz channel splatter.

**Durable Self-Healing (`mesh-poller.uc`)**:
AP reboot resets Huawei AP#2 radios to carrier auto-selection. `mesh-poller.uc` monitors channel states every cycle (~20s). If an `(auto)` suffix is detected on AP#2, it immediately re-applies both channel pins (5G Ch 52 / 2.4G Ch 11), locks 20MHz bandwidth, enables PMF and TxBeamforming, disables Smart Channel Selection (SCS), and logs the action to `/tmp/mesh-selfheal.log`. (Self-healing telnet loops for AP#1 were removed upon hardware deprecation to eliminate poller cycle latency).

**Security/encryption reality**: all controllable nodes now use **mixed `WPA/WPA2-PSK` with `TKIPandAESEncryption`** (`psk-mixed` in OpenWrt terms). Wi-Fi networks share a synchronized pre-shared key (PSK) across all BSSIDs.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
