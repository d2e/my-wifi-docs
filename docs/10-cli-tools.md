## 💻 10. CLI Tools & Script Reference

### Spectrum Discovery & Automated Channel Optimizer
```bash
# Scan whole-home RF environment and compute optimal non-overlapping channel plan
python3 wifi_channel_optimizer.py

# Discover and automatically apply + persist optimal channels across all 4 APs
python3 wifi_channel_optimizer.py --apply
```

### Nokia Gateway CLI Tool
```bash
# Check status (radios, channels, BSSIDs)
node nokia_wifi_control.js

# Change Wi-Fi channel
node nokia_wifi_control.js --5g 44 --24g 11

# Set Whole-Home DHCP DNS (Primary JioRouter, Secondary TP-Link)
node nokia_wifi_control.js --set-dns 192.168.0.50 192.168.0.12
```

### Direct Node Access
```bash
# JioRouter AX6000 (OpenWrt) — Primary Automation Hub, SSH, root shell
ssh root@192.168.0.50

# TP-Link (OpenWrt) — Secondary AP, SSH, root shell, no session limits
ssh root@192.168.0.12

# Huawei AP #2 — telnet (AP #1 is decommissioned and replaced by JioRouter)
telnet 192.168.0.98     # AP #2 - login root/<telnet-password>, then `su` for SU_WAP> vendor CLI; `shell` gives REAL busybox shell

# Mesh status (fast, reads cached file on JioRouter primary hub)
curl -sk "http://192.168.0.50/cgi-bin/mesh-api?action=status"

# Poller service control (runs on JioRouter .50)
ssh root@192.168.0.50 '/etc/init.d/mesh-poller stop|start|restart'
ssh root@192.168.0.50 'cat /tmp/mesh-status.json'      # current snapshot
ssh root@192.168.0.50 'cat /tmp/mesh-history.tsv'      # raw rolling history buffer
```

---

[← Back to CLAUDE.md index](../CLAUDE.md)
