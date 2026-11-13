## 🎛️ 5. Central Mesh Web App — Completely Rebuilt Architecture

### What it used to be
The original `mesh-api` CGI script was **100% fake**: a hardcoded IP→AP lookup table with no live queries at all, plus a hardcoded dummy fallback client if the (nonexistent) real logic produced nothing. `mesh.html` had a single flat dashboard view.

### What it is now
A three-part pipeline, hosted natively on the high-memory **JioRouter AX6000** (`192.168.0.50`, 512MB RAM, quad-core Cortex-A53), with TP-Link (`192.168.0.12`) operating as a secondary AP and auto-redirecting traffic to Jio:

1. **`/usr/bin/mesh-poller.uc`** (background daemon, managed by **`/etc/init.d/mesh-poller`** — a procd service, enabled at boot via `S99mesh-poller` in `/etc/rc.d/` on Jio). Runs an infinite loop, roughly every ~20s:
   - Pings Nokia (`192.168.0.1`), TP-Link (`192.168.0.12`), AP #1 (`192.168.0.110`), AP #2 (`192.168.0.98`) for real latency/jitter/loss.
   - Authoritative DHCP / host resolution directly from Nokia Gateway via SSH.
   - Queries Jio's own Wi-Fi 6 radios locally via `iwinfo` (real per-client signal + channel).
   - Queries TP-Link's radios via single-session SSH (`dbclient -y -i /root/.ssh/id_dropbear root@192.168.0.12`).
   - Telnets into Huawei AP #2 (`hostapd_cli all_sta` after `shell`, plus `display wifi radio` for channel) and Huawei AP #1 (vendor CLI `display wifi associate` + `display wifi radio` + `get wlan associated` for real RSSI).
   - **SSHes into the Nokia gateway** (`admin`/`<admin-password>`, then `enable` → `show` → `wifi client list` and `lan ip hosts`) for its own client table and live dynamic DHCP hostnames.
   - Resolves unknown MAC vendors via `/etc/mesh-vendor-cache`.
   - Computes **real throughput** (Mbps) via cumulative byte-counter deltas across all radios.
   - Maintains a rolling history buffer for the Performance tab's charts.
   - Writes everything atomically to **`/tmp/mesh-status.json`**.

2. **`/www/cgi-bin/mesh-api`**: serves `/tmp/mesh-status.json` instantaneously, handles whole-home YouTube blocking sync to TP-Link dnsmasq, and manages autonomous roaming kicks via `roam-daemon.uc`.

3. **`/www/mesh.html`**:
   - Access at `http://192.168.0.50/mesh.html` (TP-Link `http://192.168.0.12/` auto-redirects).
   - **📊 Live Dashboard**: Real-time client table with dynamic DHCP hostnames, device category icons, MAC-keyed vendors, and verified link quality.
   - **🗺️ Mesh Topology — Live**: single consolidated Cat6 wired backhaul tree diagram with real-time latency badges, per-node operating channels, a one-line status strip (gateway/backhaul/channel plan/client health), and scrollable client chips grouped by AP with band tags (5G/2.4G) and signal metrics.
   - **📈 Performance**: Real-time `<canvas>` line charts for sub-2ms latency, jitter, packet loss, and live radio throughput.

**Backup of the original fake `mesh-api`** is preserved at `/www/cgi-bin/mesh-api.bak.20240923` on the router.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
