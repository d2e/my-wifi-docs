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

# Set Whole-Home DHCP DNS
node nokia_wifi_control.js --set-dns 192.168.0.12 1.1.1.1
```

### Direct Node Access
```bash
# TP-Link (OpenWrt) — SSH, root shell, no session limits
ssh root@192.168.0.12

# Huawei AP #1 & AP #2 — telnet
telnet 192.168.0.110    # AP #1 - login root/<telnet-password>, then `su` for SU_WAP> vendor CLI
telnet 192.168.0.98     # AP #2 - same login; `shell` after `su` gives a REAL busybox shell here

# Mesh status (fast, reads cached file, no live telnet)
curl -sk "https://192.168.0.12/cgi-bin/mesh-api?action=status"

# Poller service control
ssh root@192.168.0.12 '/etc/init.d/mesh-poller stop|start|restart'
ssh root@192.168.0.12 'cat /tmp/mesh-status.json'      # current snapshot
ssh root@192.168.0.12 'cat /tmp/mesh-history.tsv'      # raw rolling history buffer
```

---

[← Back to CLAUDE.md index](../CLAUDE.md)
