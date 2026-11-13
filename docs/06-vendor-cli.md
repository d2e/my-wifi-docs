## 🛠️ 6. Vendor CLI Reference & Hard-Won Gotchas

### Huawei vendor CLI (both AP #1 and AP #2, same firmware family — "Dopra Linux")
Telnet in, login `root`/`<telnet-password>`, then `su` at the `WAP>` prompt to reach `SU_WAP>`. From there:
- `display wifi radio` — live channel/bandwidth/PMF state per band.
- `display wifi associate` — band-segmented station table (MAC, rates, mode).
- `display wifi information` — SSID/BSSID/encryption/channel summary.
- `display wifi filter` — shows configured MAC allow/deny list.
- `add wifi filter index [1|5] mac {mac}` / `del wifi filter ...` / `set wifi filter index [1|5] mode [0=off|2=blacklist]`.
- `set ssid index [1|5] name/security/password {value}` — SSID index 1 = 2.4GHz, index 5 = 5GHz.
- `set wifi expert band [a=5G|b=2.4G] pmf [0|1] bandwidth [20M|40M|80M|Auto] guardinterval/dtim/txbf/occac/scs/beaconperiod ...`
- **`get wlan basic laninst 1 wlaninst [1|5]`** — real per-band `TotalBytesSent`/`TotalBytesReceived`/`TotalPacketsSent`/`TotalPacketsReceived` counters for throughput.
- **`get wlan associated laninst 1 wlaninst [1|5] deviceinst {N}`** — real per-station RSSI/SNR/noise/uptime (`X_HW_RSSI`, `X_HW_SNR`, etc.) wired into the dynamic poller.
- **`set tr069 info Enable 0 InformEnable 0 AcsUrl http://127.0.0.1`** — Disables TR-069 carrier telemetry and periodic inform check-ins.

**Carrier Bloatware & Service Audit (Huawei HG8145V5):**
Because Huawei AP #1 and AP #2 are used as pure Layer-2 Wi-Fi APs with optical PON ports disconnected, several background carrier daemons run in RAM:
- `cwmp` (49 MB) — TR-069 auto-configuration client (disabled via `set tr069 info Enable 0`).
- `voice_h248sip` (87.4 MB) — VoIP telephony daemon for unused RJ-11 landline port.
- `omci` (87.6 MB) — Optical management interface daemon.
- `pppd nic-wan1` (1.4 MB) — Background PPPoE dialer over disconnected fiber port.
- `procmonitor` — Kernel watchdog supervisor that respawns core services if killed.
- **Performance Impact:** Zero CPU degradation. Hardware fast-forwarding (`ffwd` ASIC) handles Layer-2 Cat 6 switching directly in silicon at full wire speed.

**Dead ends already tried on AP #1 — do not re-attempt these for throughput/stats, they don't work:**
- `display port statistics` / `display portstatistics` — need a `portid` param for wired LAN ports, not wireless.
- `display sfwd port statistics` — runs fine but returns **all zeros** for every port/SSID even under live traffic; this counter appears to only track a software-forwarding path that's bypassed by hardware-accelerated forwarding on this chipset.
- `get wlan stats laninst 1 wlaninst [1|5]` — only exposes `ErrorsSent`/`ErrorsReceived`/`DiscardPacketsSent`/`DiscardPacketsReceived`, not byte/throughput data.
- AP #1's shell has **no scripting/binary-execution capability at all** — `ls`, `echo`, `cat` all fail with `ERROR::Command is not existed`. The "BusyBox v1.26.2" banner shown after typing `shell` is a decoy; it's still the same restricted `display`/`set` vendor CLI underneath. (AP #2's `shell` is a *real* busybox with working `hostapd_cli`/`ls`/`ifconfig` — the two units behave differently despite being the same hardware model.)

**Channel-pinning on Huawei APs (Working via Native Vendor CLI `set wlan basic`):**
Discovered and confirmed working on both AP #1 and AP #2 without requiring root/busybox shell:
- **5 GHz channel control**: `set wlan basic laninst 1 wlaninst 5 Channel <N>`
  (e.g. `set wlan basic laninst 1 wlaninst 5 Channel 149` or `157` or `52` or `36`)
- **2.4 GHz channel control**: `set wlan basic laninst 1 wlaninst 1 Channel <N>`
  (e.g. `set wlan basic laninst 1 wlaninst 1 Channel 1` or `6` or `11`)
- **Revert to auto channel selection**: `set wlan basic laninst 1 wlaninst [1|5] AutoChannelEnable 1`
- **Verify status**: `display wifi radio` (shows explicit channel number without the `(auto)` suffix when fixed).
- **Over-the-air confirmed**: Setting `Channel 157` on AP #1 was physically verified via real-time 802.11 beacon frames captured on TP-Link's wireless scan before returning AP #1 to Channel 149.
- Note: On AP #2, low-level direct driver ioctls (`iwpriv vap4 channel <N>`) are also available via its unlocked busybox shell, but `set wlan basic` is the universal, vendor-native command that works across both locked (AP #1) and unlocked (AP #2) units.

**Interactive Expect State Machine Engine ([`huawei-expect`](file:///Users/dhiraj.thakur/project/my_wifi/huawei-expect)):**
Deploys a zero-dependency POSIX FIFO interactive Expect state machine replacing blind `sleep 1` pipes on OpenWrt:
- **Bidirectional Pipes**: Uses non-blocking read-write FIFOs (`exec 3<>$FIFO_IN`, `exec 4<>$FIFO_OUT`) and `nc <IP> 23`.
- **Newline-Independent Prompt Matching**: Solves the telnet VTY prompt gotcha (prompts like `Login: `, `Password: `, `WAP>`, `SU_WAP>` emit no trailing `\n`) using BusyBox `read -t <sec> -r -n 1 ch` byte accumulation with regex/case evaluation.
- **Carriage Return Normalization**: Dopra raw VTYs output `0x0d 0x0d` (`\r\r`) rather than standard `\n`; `huawei-expect` pipes output through `tr '\r' '\n'` to preserve tabular formatting and clean line filtering.
- **Atomic Mutex & Socket Recycling**: Uses `flock -x -n` on `/tmp/huawei-telnet-<IP>.lock` with a 1-second post-disconnect cooldown in `cleanup()`. Allows concurrent daemons (`mesh-poller` and `mobile-scheduler`) to queue cleanly with zero session rejections, **completely eliminating the need to stop/restart `mesh-poller`**.
- **Performance**: Sub-second execution (~0.75s per transaction vs. 12-15s blind time-delayed chains).

### Nokia gateway CLI & Encrypted Web API (G-2425G-A)
SSH or Telnet in with `admin`/`<admin-password>` (Telnet/SSH must first be enabled under the web UI's **Security → Access Control**, LAN column only — closed by default). Landing prompt is `user>` (unprivileged). From there:
- `enable` — no password needed, escalates straight to `user#` (privileged).
- `list` at any prompt level — shows the commands valid at that level.
- `show` — enters `user(show)#`: `wifi client list` (band-segmented client table with hostnames/status), `wlan` (neighboring scan), `sysinfo`, `lan ip hosts` (DHCP leases), `lan stats`, `wan all statistics`.

**Nokia Master Gateway Control Tool ([`nokia_wifi_control.js`](file:///Users/dhiraj.thakur/project/my_wifi/nokia_wifi_control.js)):**
Programmatically controls Nokia Master Gateway via SJCL AES-CBC + RSA-1024 encrypted POST requests.
- **Check Status**: `node nokia_wifi_control.js` (reads live 2.4 GHz and 5 GHz channel, SSID, BSSID).
- **Set 5 GHz Channel**: `node nokia_wifi_control.js --5g <channel>` (e.g. `node nokia_wifi_control.js --5g 44`).
- **Set 2.4 GHz Channel**: `node nokia_wifi_control.js --24g <channel>` (e.g. `node nokia_wifi_control.js --24g 11`).
- **Set Master DHCP DNS**: `node nokia_wifi_control.js --set-dns 192.168.0.12 1.1.1.1` (configures Primary DNS to TP-Link Ad-Blocker, Secondary to Cloudflare fallback).
- **Toggle YouTube Switch**: `node nokia_wifi_control.js --youtube <on|off|status>` (synchronizes Nokia Gateway Parental Control with TP-Link whole-home DNS blackhole).
- **Parental Control**: `node nokia_wifi_control.js --parental-control <on|off>`.

### Critical parsing gotchas (all already handled in `mesh-poller.sh`, but worth knowing if editing it)
- **Every echoed command line is prefixed with the current prompt** (e.g. `SU_WAP>get wlan basic laninst 1 wlaninst 1`), not a clean standalone line. Don't anchor parsing regexes to line-start (`^`) when matching an echoed command.
- **A trailing BEL character (`0x07`, `^G`) follows every echoed command**, before the `\r\n`. This breaks `$`-anchored regex matches. Strip it explicitly: `tr -d '\r\007'`.
- **AP #1 and AP #2 allow only ONE concurrent telnet session** (hardcoded carrier limit: `The number of sessions exceeds the specificaitons`). The poller is optimized to connect, read stats in **~1.5s**, and immediately send `quit\r\n` with socket cleanup (`killall nc dbclient 2>/dev/null`), keeping the CLI available > 90% of the time.
- **This busybox `nc` supports NO flags at all** — not even `-w` for timeout. `nc IPADDR PORT` only.
- **This busybox `sleep` only accepts whole seconds** — `sleep 0.3` fails silently. Always use integer seconds (`sleep 1`).
- **A CGI script that backgrounds a process without detaching stdin/stdout/stderr can hang the HTTP response for minutes** — wrap backgrounded blocks with `</dev/null >/dev/null 2>&1 &`.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
