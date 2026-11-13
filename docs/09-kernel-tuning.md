## ⚡ 9. Linux Kernel 5.15 Carrier-Grade Tuning Profile (TP-Link)

Persistently configured on TP-Link Archer A6 via `/etc/sysctl.d/99-network-optimization.conf` and `/etc/rc.local`:
- **`net.ipv4.tcp_slow_start_after_idle = 0`**: Eliminates TCP window resets on idle connections (instant video/web resumption).
- **`net.core.rmem_max = 4194304` & `wmem_max = 4194304`**: 4 MB socket buffer scaling for Gigabit Bandwidth-Delay Product (BDP).
- **`net.core.netdev_max_backlog = 10000`**: 10x packet backlog depth to absorb line-rate bursts without softirq drops.
- **`net.ipv4.tcp_fastopen = 3`**: Client and server TCP Fast Open to shave 1 full round-trip handshake time off supported connections.
- **`vm.swappiness = 0` & `vm.vfs_cache_pressure = 50`**: Prevents virtual memory paging stalls and preserves inode/dentry caches in RAM.
- **OOM Prevention (`vm.watermark_boost_factor = 0` & `vm.min_free_kbytes = 8192`)**: Disables aggressive kernel watermark boosting (which previously inflated watermarks by 150% under network bursts, triggering premature OOM kills even with 22MB free). Reclaims 8 MB of permanent usable RAM by scaling `min_free_kbytes` from 16384 to 8192 for the 128 MB RAM architecture.
- **Multi-Core Receive Packet Steering (RPS Mask `0xf`)**: Distributes softirq packet handling across all 4 virtual CPU cores for `phy1-ap0` (5G), `phy0-ap0` (2.4G), and `br-lan`.
- **MIPS 1004Kc SMP IRQ Affinity (2026-09-03)**: Fixed hardware IRQ starvation on CPU 0 where all 9.5M+ PCIe Wi-Fi interrupts had landed. IRQ 30 (`mt7603e` 2.4G) is steered to CPU 1 (mask `0x2`), IRQ 31 (`mt7615e` 5G) is steered to CPU 2 & 3 (mask `0xc`), leaving CPU 0 unburdened for userspace processes (`hostapd`, `dnsmasq`, `mesh-poller`). Persisted in `/etc/rc.local`.
- **BSS Rate Pruning (`cell_density='2'`)**: Elevated mandatory minimum basic rate to 12 Mbps (`basic_rates=120 240`) on both 2.4 GHz and 5 GHz, eliminating legacy 1, 2, 5.5, 6, 9, 11 Mbps rates. Prevents sticky clients by sharply dropping cell edge SNR.
- **802.11k Cross-Vendor Neighbor Table Seeding**: Pre-seeds hostapd via in-memory ubus with live BSSIDs and channel descriptors for Nokia Gateway (Ch 44 / Ch 11), Huawei AP #1 (Ch 149 / Ch 1), and Huawei AP #2 (Ch 52 / Ch 11), reducing client roaming scan latencies from ~3000ms to <35ms. Maintained persistently in `mesh-poller.sh`.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
