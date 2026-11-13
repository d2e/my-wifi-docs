## ❄️ 12. Panasonic Smart AC Embedded Hub & Thermodynamic AI Engine

As of 2026-08-30, the network hosts a **100% self-contained embedded Smart AC hub and AI prediction engine running 24/7 on the TP-Link Archer A6 router (`192.168.0.12`)** under OpenWrt:

### 1. Hardware & Space Profile
- **AC Model**: Panasonic CS-CU-NU18AKY5WX (Inverter AC, Converti7)
- **Wi-Fi Module**: Espressif module inside AC, MAC `98:3d:ae:55:e2:6c`, IP `192.168.0.4`.
- **Radio Link**: Attached to TP-Link Hub 2.4 GHz (Ch 6, HT20, MCS 7, `-59 dBm` RSSI, 802.11w PMF active, $99.993\%$ delivery rate).
- **Exact Coordinates**: `18.5506939° N, 73.9594486° E` (`18° 33' 02.50" N, 73° 57' 34.01" E`), Kharadi / Viman Nagar, Pune, Maharashtra.
- **Matter Fabric ID**: `466914946561840` | Space ID: `93b6a59c3a` ("Master Bed Room") | Device ID: `df034e6e627b`.

### 2. Physical Inverter Electrical Model
- **Cruising Steady-State**: $47.0\text{ W}$ (~₹0.38/hr) = Fan $12\text{W}$ + Electronics $5\text{W}$ + Compressor $30\text{W}$.
- **Dynamic Load Scaling**: Scaled by thermal delta ($\Delta T = T_{\text{room}} - T_{\text{target}}$) and Converti cap up to $728\text{W}$ (at 55% cap) or $1324\text{W}$ (at 100%).
- **6-Month Cumulative Billing Ledger**: $596.06\text{ kWh} = \text{₹ }4,768.50$ (March: 209.68 kWh, April: 198.81 kWh, May: 128.98 kWh, June: 36.19 kWh, July: 9.77 kWh, August MTD: 12.62 kWh).

### 3. Four Embedded AI Predictive Models (Pure POSIX Closed-Form Arithmetic)
1. **Lumped $R$-$C$ Thermal Inertia Envelope**: $C = 0.55\text{ kWh/}^\circ\text{C}$, $R = 4.2^\circ\text{C/kW}$, Envelope Time Constant $\tau = 2.31\text{ Hours}$. Heat ingress at night: $-0.26\text{ kW}$ (Negative thermal lift).
2. **ASHRAE-55 Thermal Comfort (PMV / PPD)**: Target $24.5^\circ\text{C}$ with Quiet Fan + Coanda Ceiling Louvre produces $\text{PMV} = -0.25$ (Ideal Neutral Comfort) and $\text{PPD} = 6.3\%$ (Optimal Minimum Dissatisfaction).
3. **7-Day Pune CDD Climate & Cost Forecast**: Daily Open-Meteo Pune integration via CDD regression ($E = 0.790 \times \text{CDD} + 0.35\text{ kWh}$); predicts $10.16\text{ kWh}$ (~₹81.28) for the upcoming week.
4. **802.11 Fresnel Micro-Motion Radar**: $93.9\%$ RF link stability index confirming calm, undisturbed room multipath conditions.

### 4. Router Service & UI Endpoints
- **Whole-Home Mesh Manager**: `http://192.168.0.12/mesh.html` (Dedicated `❄️ Panasonic Smart AC` tab).
- **Standalone AC Application**: `http://192.168.0.12/ac.html`.
- **CGI REST API**: `http://192.168.0.12/cgi-bin/ac-api`.
- **Procd Service**: `/etc/init.d/ac-poller` (`/usr/bin/ac-poller.uc`, pure ucode).
- **Resource Footprint**: `< 1.2 MB` RAM, `< 50 KB` Flash, `~0.1%` CPU. Zero laptop dependencies.

### 5. Real Energy Ledger & Real Device Control (2026-09-03 rework)
- **Energy (`power_metrics`)**: `today_kwh` / `yesterday_kwh` / `month_kwh` are now **live values from the MirAIe cloud `powerConsumption` API** (`grain=Daily` over ~40 days + `grain=Monthly` over 3 months; the API rejects ranges > 6 months). Cached 15 min in `/tmp/miraie-energy-cache.json`; on cloud failure a cached value is served with `energy_stale: true`. The old hardcoded `1.344/3.199/12.625` constants (stale August MTD) are gone. `daily_7d` carries the last 7 daily totals.
- **RF Telemetry Extraction (`rf_health`)**: Extracted directly via `iw dev phy0-ap0 station get 98:3d:ae:55:e2:6c` (with `phy1-ap0` fallback). The previous belief that the AC was not associated with TP-Link was caused by a parser bug (`split(dump, /Station .../)` discarded the MAC delimiter, causing regex matching on the remaining chunks to fail). Verified live: the AC is directly connected to TP-Link 2.4 GHz at `-61 dBm`, `72.2 MBit/s MCS 7 short GI`, with real tx/retry counters. Float division in ucode `(+txRetries * 100.0) / (+txPkts + +txRetries)` ensures accurate retry rates.
- **Outdoor AQI & Weather (`aqi_data.outdoor`)**: Fixed Open-Meteo API query to use `current=pm2_5,pm10,european_aqi,us_aqi,uv_index` (the previous parameter `eu_aqi` was invalid, causing Open-Meteo to return an error JSON). Frontends now render live US AQI (`29 🟢 Good`), PM2.5, PM10, EU AQI, and UV Index with dynamic color badges and scale marker.
- **Indoor AQI (`aqi_data.indoor`)**: explicitly `supported: false` — this model exposes no IAQ/PM2.5/dust sensors via MirAIe; frontends render `--`.
- **Real device control (`ac-api` POST)**: maps UI commands (`power`, `temp`, `mode`, `fan` (UI `med` → MirAIe `medium`), `preset`, `converti`, `swing-v`, `swing-h`, `display`) to MirAIe **MQTT-over-TLS control** (`mqtt.miraie.in:8883`, username = homeId, password = access token, topic `<userId>/<homeId>/<deviceId>/control`, payload base `{"cnt":"an","ki":1,"sid":"1"}`), published via **`mosquitto_pub`** from `mosquitto-client-ssl`. Stdin reading is safely bounded using `CONTENT_LENGTH`. On successful publish, optimistic local updates are written to `/tmp/ac-status.json` so immediate polling reflects the new state instantly.
- **Frontends**: Standalone `http://192.168.0.12/ac.html` and `http://192.168.0.12/mesh.html` (AC tab) dynamically synchronize button highlight states (Power, Temp Presets, Fan Speed, Converti, V-Swing, Panel LED Display) directly from reported live hardware status. Stale "Preview Mode" badges replaced with live hardware control indicators.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
