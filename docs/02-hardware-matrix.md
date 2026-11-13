## 📇 2. Hardware & Management Matrix (corrected IPs)

| Node Role | Hardware Model | Real Management IP | Credentials | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **Master Gateway** | Nokia G-2425G-A (GPON ONT, MTK7528) | `192.168.0.1` | Web Admin + SSH/Telnet: `admin` / `<admin-password>` | **No longer opaque, but not rooted either.** Web admin is reachable directly. Telnet/SSH are closed by default - enable them yourself under **Security → Access Control** (set the **LAN** column for Telnet/SSH to `Allow`; leave WAN `Deny`). SSH drops into a restricted Zebra-VTY-style "User CLI" (`enable` → privileged `#` → `show` → read-only submenu with `wlan`, `wifi client list`, `sysinfo`, `lan ip hosts`, `ridump`, etc. - genuinely useful live data, no root needed). Root shell is walled off: the documented community shell-escape (`shell` → `Password2:` → `'; /bin/sh; #`) is patched on this firmware (`3FE49362IJLJ07` / `1.2402.307`) - `Password2` is now a dynamic/challenge password with no published algorithm. The full encrypted `config.cfg` backup export is also undecryptable with any publicly-known key (its payload starts with `0x40`, not the `0xFF`+salt format every public tool expects) - don't burn more time on that specific angle without new public research. |
| **Primary Automation Hub & AP #1 Replacement** | JioRouter AX6000 JIDU6101 (MT7986 Quad-Core ARM64) | `192.168.0.50` | `root` (no pass) | OpenWrt SNAPSHOT (Linux Kernel 6.18.55, build `r37948-fc8ff4d2ed`). Wi-Fi 6 (802.11ax) 4x4 MU-MIMO 160MHz (`HE160`). Replaced Huawei AP #1 directly under Nokia Gateway. Hosts USB 2.0 Media Stack (`/mnt/sda1` / `/mnt/usb` with native `transmission-daemon` :9091 and `minidlnad` :8200). Primary Automation Hub (hosts dashboards, pollers, AC console, TV control plane) and Primary Whole-Home Ad-Blocking DNS (`adblock-fast` + `dnsmasq`, 28k+ domains). Bridged L2 into Nokia master DHCP; 802.11k/v roaming synced with TP-Link. |
| **Secondary AP #2** | Huawei HG8145V5 | `192.168.0.98` | `root` / `<telnet-password>` | Confirmed genuinely Huawei via telnet banner. Daisy-chains TP-Link. USB storage previously on this node has been migrated to the JioRouter AX6000. Only allows **one concurrent telnet session** — see §6 gotchas. |
| **Secondary AP #3** | TP-Link Archer A6 v3 | `192.168.0.12` | `root` / `<ssh-password>`, web `root`/`<web-password>` | OpenWrt 23.05.5. OpenSSH 9.8p1 on port 22 with Post-Quantum KEX (`sntrup761x25519-sha512@openssh.com`), Dropbear fallback on port 2222. Secondary ad-blocking DNS fallback (`adblock-fast` + `dnsmasq`). Redirects web traffic to Jio dashboard (`http://192.168.0.50/mesh.html`). |
| **Huawei AP #1 (Deprecated)** | Huawei HG8145V5 | `192.168.0.110` | `root` / `<telnet-password>` | **DEPRECATED / DECOMMISSIONED.** Replaced by JioRouter AX6000 (`192.168.0.50`). Poller telnet hooks removed. |

**MAC vendor identification is now automated** — see §4's vendor cache. Known devices as of this session:

| MAC | Device | Notes |
|---|---|---|
| `96:1f:f5:8a:71:f2` | **Lava Agni 2 Phone (privacy MAC)** | Previously misidentified and blacklisted; purged from TP-Link and Huawei AP #2 filters on 2026-09-03. |
| `4c:17:44:87:5d:39` | Amazon Fire TV Stick | Vendor-confirmed (Amazon Technologies Inc.). Connects to AP #1's 5GHz. DHCP hostname `amazon-29049d4d4` confirms. |
| `40:24:b2:29:15:36` | **Actually the Mi TV, not Chromecast** — see correction below | Was previously mislabeled "Smartphone/Mobile", then "Chromecast" — DHCP hostname is literally `MITV`. |
| `7c:f3:4d:dc:24:8f` | Apple MacBook / Laptop | DHCP hostname `MacBook-Pro`. This machine also has a second, rotating-privacy-MAC lease (`34:29:8f:75:a0:ac`, same hostname) from Apple's per-network private address feature. |
| `a6:9d:de:40:3a:d7` | **Motorola Moto G45 5G phone** (IP: `192.168.0.13`) | DHCP hostname `moto-g45-5G`. **COMPLETELY BLOCKED network-wide** (2026-09-13) across Nokia 2.4G & 5G, TP-Link 2.4G & 5G, AP #1, and AP #2; 0% internet access. |
| `1a:e0:6b:fc:5d:a2` | Motorola Moto G45 5G (randomized/private MAC) | DHCP hostname `moto-g45-5G` (IP: `192.168.0.34`). **COMPLETELY BLOCKED network-wide** across all 4 nodes. |
| `08:3a:2f:87:51:b1` | Guangzhou Juan Intelligent Tech device at `192.168.0.3` | The device that was mistaken for AP #1 in an earlier session. Identity beyond manufacturer still unknown; DHCP hostname is just `Unknown_08:3a:2f:87:51:b1` (device-supplied, not resolved). Static IP lease, connects to Nokia's own 2.4GHz (`U101-slow`) - see below. |
| `1a:35:32:98:4e:62`, `9a:d9:7b:5b:07:ff` | Unknown / `realme-narzo-30-5G` | `9a:d9:7b:5b:07:ff` resolves via DHCP hostname to a Realme Narzo 30 5G phone; it also appears under a second rotating MAC (`72:3c:ad:45:7e:60`, same hostname) - privacy-MAC rotation, not two separate phones. `1a:35:32:98:4e:62` remains genuinely unidentified. |
| `ce:a5:c0:cc:b3:32` | Lava Agni 2 phone | Randomized MAC; identified by the user, not by lookup. Was hitting an RF/weak-signal issue on TP-Link's 5GHz — see §8. |
| `ac:5f:3e:29:13:d8` | Samsung Galaxy S7 phone | DHCP hostname `Galaxy-S7`. Mobile client that roams across the home; actively managed by the Autonomous Roaming Watchdog (not stationary). |
| `f8:54:f6:2d:82:29` | Desktop PC, Gigabyte B450M DS3H motherboard WiFi (new) | DHCP hostname `dt-B450M-DS3H-WIFI`. |
| `f4:96:34:7d:5f:96` | HP Envy x360 15m laptop (new) | DHCP hostname `dt-HP-ENVY-x360-Convertible-15m`. |
| `14:0a:c5:79:21:27` | Unknown (new) | No resolvable hostname or vendor yet. |

**Correction (2026-08-28): the Mi TV MAC-blocking policy in §7 was very likely targeting the wrong device.** Live DHCP host data pulled from the Nokia gateway (see §2/§5 - Nokia SSH access) shows `96:1f:f5:8a:71:f2` (the MAC documented as "Mi TV" and deliberately blocked) has DHCP hostname `LAVA-LXX504` - the same hostname shown by the Lava Agni 2 phone's other known MACs (`ce:a5:c0:cc:b3:32`, `74:cb:f3:79:9a:b0`), strongly suggesting `96:1f:f5:8a:71:f2` is just another privacy-MAC rotation of that same phone, not the TV. Meanwhile `40:24:b2:29:15:36` (labeled "Chromecast") has DHCP hostname `MITV`. **Net effect: the §7 blocking policy has likely been forcing your phone onto AP #1, not the TV, and the real Mi TV (`40:24:b2:29:15:36`) is unblocked and roaming freely.** Not yet re-verified by physically toggling the filter and watching which device drops - do that before trusting this fully, but treat the §7 policy as suspect until then.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
