## 📡 3. RF Spectrum — Reality vs. Original Design

The original doc claimed a static, non-overlapping channel plan. For most of this project that wasn't true — both Huawei APs ran `AutoChannelEnable=1` and drifted over time (observed landing on 1, 9, 10, 36, 149, 157 across sessions). As of 2026-08-28, all three 5GHz radios are now genuinely fixed and non-overlapping:

| Node | 5GHz | 2.4GHz | Channel mode | How it's fixed |
|---|---|---|---|---|
| TP-Link | **Ch 36, fixed, VHT80 (80MHz)** | **Ch 6, fixed, HT20 (20MHz)** | Static | `uci`/OpenWrt wireless config (`network.lan`-adjacent wireless config, standard OpenWrt) |
| Huawei AP #1 | **Ch 149, fixed, 80MHz** | **Ch 1, fixed, HT20 (20MHz)** | Static (poller self-healed) | Vendor CLI (`set wlan basic laninst 1 wlaninst 1 Channel 1` + `set wifi expert band b bandwidth 20M pmf 1 txbf 1 scs 0 guardinterval short`) |
| Huawei AP #2 | **Ch 52 (DFS), fixed, 80MHz** | **Ch 11, fixed, HT20 (20MHz)** | Static (poller self-healed) | Vendor CLI (`set wlan basic laninst 1 wlaninst 1 Channel 11` + `set wifi expert band b bandwidth 20M pmf 1 txbf 1 scs 0 guardinterval short`) |
| Jio AX6000 | **Ch 36 (HE160, 160MHz width, center Ch 50 @ 5250MHz)** | **Ch 1, fixed, HE20 (20MHz)** | Static | `uci`/OpenWrt wireless config (Wi-Fi 6 802.11ax, 4x4 MU-MIMO, 160MHz ultra-wideband, 30 dBm txpower) |
| Nokia | **Ch 44, fixed, VHT80 (80MHz)** | **Ch 11, fixed, HT20 (20MHz)** | Static | Encrypted Web API (`nokia_wifi_control.js` / `wlan_config.cgi`) |

**Collision status (verified 2026-08-30, independently re-checked):**
- **5 GHz Spectrum**: Jio=Ch 36 (HE160, 160MHz width, center Ch 50), TP-Link=Ch 36 (VHT80), AP#2=Ch 52 (DFS, 80MHz), AP#1=Ch 149 (80MHz), Nokia=Ch 44 (VHT80). Non-DFS primary channel 36 ensures zero radar CAC boot delays.
- **2.4 GHz Spectrum**: Clean non-overlapping standard channel plan: AP#1=Ch 1, TP-Link=Ch 6, AP#2=Ch 11, Nokia=Ch 11. Co-channel reuse (Ch 11) is assigned to Nokia and AP#2 because they have the greatest physical distance and RF attenuation (-81 to -89 dBm isolation through multiple concrete walls, well below the carrier sense threshold). All 2.4GHz radios are locked to **HT20 (20 MHz width)** to eliminate adjacent channel interference and 40MHz channel splatter.

**Durable Self-Healing (`mesh-poller.sh`)**:
AP reboot resets Huawei radios to carrier auto-selection. `mesh-poller.sh` monitors channel states every cycle (~20s). If an `(auto)` suffix is detected on either 5GHz or 2.4GHz for AP#1 or AP#2, it immediately re-applies both channel pins (AP#1: 5G Ch 149 / 2.4G Ch 1; AP#2: 5G Ch 52 / 2.4G Ch 11), locks 20MHz bandwidth, enables PMF and TxBeamforming, disables Smart Channel Selection (SCS), kills TR-069 carrier bloatware, and logs the action to `/tmp/mesh-selfheal.log`.

**Security/encryption reality**: all three controllable nodes (TP-Link, AP #1, AP #2) now use **mixed `WPA/WPA2-PSK` with `TKIPandAESEncryption`** (`psk-mixed` in OpenWrt terms), not pure WPA2-CCMP as originally documented. This was changed on TP-Link specifically to match the Huawei APs' native mode, after inconsistent RSN cipher advertisement across BSSIDs sharing one SSID name caused some Android clients (confirmed: a Lava Agni 2) to fail negotiating with TP-Link specifically. Wi-Fi networks share a synchronized pre-shared key (PSK) across all BSSIDs.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
