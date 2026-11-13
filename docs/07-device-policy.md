## 🔒 7. Device Policy & Filter History
- **Purged Stale Phone Block (2026-09-03)**: A historical policy originally intended to pin the Mi TV had blacklisted MAC `96:1f:f5:8a:71:f2` on Huawei AP #2 and TP-Link. Live DHCP investigation confirmed this MAC belonged to the user's Lava Agni 2 phone (`LAVA-LXX504`), while the real Mi TV is `40:24:b2:29:15:36` (connecting stably to Huawei AP #1 on 5 GHz).
- **Motorola Feature Decommissioned (2026-09-26)**:
  - Per user confirmation that the Motorola device is no longer in the household, the entire Motorola Access Control system was decommissioned:
    - MAC blacklist filters cleared across all nodes (Nokia 2.4G & 5G, TP-Link, Huawei AP #1, Huawei AP #2).
    - UI card `#motorola-control-card` removed from `mesh.html`.
    - API endpoints removed from `mesh-api.uc` and deauth enforcement rules cleared from `roam-daemon.uc`.
    - `/etc/motorola-rules.json` deleted from router.
- **YouTube & SmartTube App Lock (Mi TV & Network-Wide, Deployed 2026-09-26)**:
  - **Network-Level Switch**: Synchronizes Nokia Gateway parental control filtering with TP-Link dnsmasq blackholing (`0.0.0.0` for `youtube.com`, `googlevideo.com`, `youtu.be`, `ytimg.com`, `youtubei.googleapis.com`). Managed via `mesh.html` Tab 1 (`#youtube-control-card`) and `/cgi-bin/mesh-api?action=youtube_toggle`.
  - **Mi TV App-Level Freeze / Lock (`pm disable-user` + `am force-stop`)**:
    - Freezes or unfreezes Android packages `org.smarttube.stable`, `com.google.android.youtube.tv`, and `com.google.android.youtube.tvmusic` on the Mi TV (`192.168.0.10`).
    - **Pure Rust Native Execution (Updated 2026-11-13)**: Operates 100% via the on-device native Rust agent (`mitv-rust-agent` on port `:9988`) with zero ADB dependency: immediately calls `/stop?pkg=...` to force-stop active playback across all YouTube/SmartTube packages, injects `/key?code=home` (KEYCODE_HOME) to return to the launcher, and invokes `/action?cmd=youtube_toggle` to freeze or enable packages. State is persisted in `/etc/mitv-youtube-app.state`.
    - **Web UI Controls**:
      - Tab 5 (📺 **Mi TV Remote**): Interactive card `#mitv-yt-app-card` with live status badge (`🟢 APPS ACTIVE` / `❄️ FROZEN / DISABLED`) and one-click freeze/restore button; SmartTube launcher card dims when frozen.
      - Tab 1 (📊 **Live Dashboard**): Quick-action button `#mitv-yt-app-btn-tab1` inside the Whole-Home YouTube card.
    - **API Endpoints**:
      - Status: `/cgi-bin/mitv-api?cmd=youtube_app_status` or `/cgi-bin/mesh-api?action=mitv_youtube_app_status`
      - Toggle: `/cgi-bin/mitv-api?cmd=youtube_app_toggle&param=<freeze|unfreeze>` or `/cgi-bin/mesh-api?action=mitv_youtube_app_toggle`

---

[← Back to CLAUDE.md index](../CLAUDE.md)
