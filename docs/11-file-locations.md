## 📁 11. File Locations

## 📁 11. File Locations

### On JioRouter AX6000 (`192.168.0.50` — Primary Automation Hub)
- `/www/mesh.html` — Live whole-home dashboard frontend (Dashboard, Topology, Performance, Panasonic AC Hub, TV / Entertainment controls).
- `/www/ac.html` — Standalone full-featured embedded Panasonic AC web console.
- `/www/index.html` — Default portal redirecting to `mesh.html`.
- `/www/cgi-bin/mesh-api` — REST endpoint serving mesh telemetry, autonomous roaming triggers, and YouTube blocking sync.
- `/www/cgi-bin/ac-api` — Native REST endpoint for AC telemetry and MQTT MirAIe control.
- `/www/cgi-bin/mitv-api` — Low-latency API for Xiaomi Mi TV control, picture modes, and HDMI-ARC audio profiles.
- `/www/cgi-bin/firetv-api` — Fast API for Amazon Fire TV Stick control and sleep/wake.
- `/usr/bin/mesh-poller.uc` — Event-loop mesh polling daemon & self-healing watchdog.
- `/usr/bin/ac-poller.uc` — 24/7 background Panasonic AC poller & AI thermodynamic model evaluator.
- `/usr/bin/entertainment-poller.uc` — Entertainment stack poller & app-aware audio profile switcher.
- `/usr/bin/roam-daemon.uc` — Whole-home autonomous roaming watchdog & sticky-client kicker.
- `/usr/bin/poller-lib.uc` — Shared library for entertainment parsing, audio profile mapping, and media states.
- `/usr/bin/seed-rrm.sh` — 802.11k neighbor seeding script (seeded at boot via `/etc/rc.local`).
- `/etc/init.d/mesh-poller` — Procd service definition for mesh-poller.
- `/etc/init.d/ac-poller` — Procd service definition for ac-poller.
- `/etc/init.d/entertainment-poller` — Procd service definition for entertainment-poller.
- `/etc/init.d/shoonya-qos` — Bridge QoS NFTables priority marking (DSCP EF) for trading APIs.
- `/etc/ac-poller.conf` — MirAIe cloud credentials for AC control.
- `/etc/mesh-vendor-cache` — Persistent MAC→vendor lookup cache.
- `/etc/mesh-youtube.state` — Whole-home YouTube blocking toggle state.
- `/mnt/sda1` (symlink `/mnt/usb`) — 28GB ext4 USB 2.0 flash drive mount.
- `/mnt/sda1/downloads` — Download destination for torrents and DLNA indexed media.
- `/mnt/sda1/transmission` — Transmission daemon state, settings, and RPC assets.
- `/etc/init.d/transmission` — Procd service definition for Transmission BitTorrent daemon (`http://192.168.0.50:9091/`).
- `/etc/init.d/minidlna` — Procd service definition for MiniDLNA media streaming server (`http://192.168.0.50:8200/`).
- `/tmp/mesh-pending-usb-deletes.txt` — Atomic queue for file and folder deletions triggered via the web UI.
- `/tmp/mesh-status.json` — Current live mesh snapshot (tmpfs, regenerated every ~20s).
- `/tmp/ac-status.json` — Current live AC telemetry and AI thermodynamic models (regenerated every 10s).
- `/tmp/entertainment-status.json` — Current live TV / Fire TV / Audio status.

### On TP-Link Archer A6 (`192.168.0.12` — Secondary AP & DNS Server)
- `/www/index.html` — HTTP redirect forwarding browsers to `http://192.168.0.50/mesh.html`.
- `/etc/init.d/adblock-fast` & `/etc/config/adblock-fast` — Network-wide ad-blocking engine.
- `/etc/config/dhcp` — `dnsmasq` DNS forwarding and adblock sinkhole configuration.
- `/etc/init.d/shoonya-qos` — Bridge QoS NFTables priority marking.
- `/etc/sysctl.d/99-network-optimization.conf` & `/etc/rc.local` — Linux 5.15 kernel tuning profile and 4-core RPS steering.
- Note: Background pollers (`mesh-poller`, `ac-poller`, `entertainment-poller`) are stopped and disabled on TP-Link to free up its limited 64MB RAM.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
