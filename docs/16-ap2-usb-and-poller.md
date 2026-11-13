## 🌊 16. USB Storage, Transmission Daemon, Media Stack & Poller Architecture Evolution

### ⚡ Architecture Evolution: Migration of USB Storage & Media Stack to JioRouter AX6000
> **Permanent Migration Note**: The physical 28GB ext4 USB flash drive has been migrated from Huawei AP #2 to the **JioRouter AX6000 (`192.168.0.50`)**.
> - **Block Device & Mount**: `/dev/sda1` is mounted at `/mnt/sda1` (with backward-compatible symlinks `/mnt/usb -> /mnt/sda1` and `/mnt/usb/data -> /mnt/sda1`).
> - **Native Daemons**: Both `transmission-daemon` (web UI `http://192.168.0.50:9091/`) and `minidlnad` (DLNA streaming `http://192.168.0.50:8200/`) run natively under OpenWrt procd (`/etc/init.d/transmission`, `/etc/init.d/minidlna`) on the Filogic 830 quad-core ARM64 CPU with 512MB RAM.
> - **Reliability Gains**: Moving USB storage away from Huawei AP #2 eliminated the AP's fragile 128MB RAM MIPS constraints, single-telnet lockouts, and multi-port `/mnt/usb/usb-_1/` path bugs.
> - **AP #1 Deprecation**: Concurrently, **Huawei AP #1 (`192.168.0.110`) was deprecated and decommissioned**, with all telnet hooks purged from `mesh-poller.uc` to eliminate 14-32s polling delays.

---

### Historical AP2 Implementation & Root Cause Archive
Installed at `/mnt/usb/usb-_1/transmission/` (also bind-mounted at `/mnt/ftproot/usb-_1/transmission/`
for FTP access): `bin/transmission-daemon` + its own `ld-linux.so.3`/`libgetrandom_preload.so`,
`start.sh`/`stop.sh`, `config/`, `web/` (the stock Transmission web UI), `transmission.log`,
`transmission.pid`. Web UI: `http://192.168.0.98:9091/transmission/web/`. AP2 has no `crond`
and no init.d hook for this at all — it was always a manually-started, unsupervised process.

### Bug #1: stale hardcoded USB mount path in `start.sh` (the actual outage)
`start.sh` sets `BASE=/mnt/usb/transmission`, but the live mount is nested at
`/mnt/usb/usb-_1/transmission` (Huawei's multi-port USB naming scheme). `/mnt/usb/transmission`
still exists as a leftover, near-empty directory (only `config/`+`lib/`, no `bin/`) — so
`$BASE/lib/ld-linux.so.3` didn't exist and `start.sh` silently failed with exit 127. The daemon's
own log (`transmission.log`) stopped mid-session on 2026-09-08 23:47 with no shutdown message,
consistent with a kill/crash rather than a clean stop; no OOM-killer evidence survived in
`dmesg`'s ring buffer by the time this was investigated. **Fix**: edited the deployed
`start.sh` in place (`BASE=/mnt/usb/usb-_1/transmission`), backed up the original as
`start.sh.bak`, relaunched. Confirmed live via `curl -I http://192.168.0.98:9091/...` → `Server: Transmission`.

### Bug #2: `mesh-poller`'s AP2 "shell" sub-flow has *never* actually worked
While adding a watchdog for the above, discovered `ap2_rf_watchdog` (Capability D, RF site
survey) and `mesh-roam-kicker.log` (Capability C, sticky-client kicker) had **never once fired
successfully** despite being deployed for a while — both route through the same `h2ShellExtra`
mechanism in `mesh-poller.uc`/`.sh` (drop from the Huawei vendor CLI into a real busybox `shell`
mid-session to run `iwpriv`). Root cause: the command immediately preceding `printf 'shell\r\n'`
(`get wlan basic laninst 1 wlaninst 5`) had **no `sleep` before it**, unlike every other command
pair in the same session. AP2's vendor CLI needs a real wall-clock gap between lines, not just
a `\r\n` separator — sent too fast, the two lines merged into one garbled command
(`...wlaninst 5shell` → `ERROR::Command is not existed`), silently dropping everything after.
**Fixed** by adding `sleep 1` before entering `shell` (and again before `exit`, for the same
reason). Verified live: `ap2_rf_watchdog` populated for the first time ever on the very next
cycle (`aps_24g`/`aps_5g` real BSSID counts, not the hardcoded `11`/`6` baseline).

### Bug #3: AP2's one-concurrent-telnet-session limit is easy to self-inflict
`mesh-poller`'s AP2 session is torn down via `kill "$H2_PID"` after a fixed sleep budget, with
`quit` only sent if every prior command already finished in time — there's no guarantee `quit`
is ever reached. When it isn't, AP2 is left holding a **stuck session under TP-Link's own IP**
(`telnet-192.168.0.12`), and the *next* cycle's login gets trapped forever answering
"Enter the ID of the session to be removed" instead of reaching `su`/`shell` at all — a
self-perpetuating lockout with no automatic recovery. Hit repeatedly while diagnosing this
session. **Mitigated** (not eliminated) by unconditionally sending `1\r\n` right after the
password on every AP1/AP2 connection: answers the removal prompt when present, and is a
harmless no-op (`ERROR::Command is not existed`) at the normal `WAP>` prompt otherwise. If a
session ever gets stuck again, clear it manually: telnet in, answer the `Enter the ID...`
prompt with `1`, then `quit`.

### Capability F: AP2 Transmission-Daemon Watchdog (new, `mesh-poller.uc`)
Every poll cycle, inside the same AP2 shell session used for kicks/RF-scan, unconditionally
invokes `cd /mnt/usb/usb-_1/transmission && ./start.sh` — safe because `start.sh` already
self-guards (`if netstat ... :9091 already listening; then exit 0`), so this is a no-op when
healthy and a real relaunch when down. Status is read back from `start.sh`'s **own** log
output (`"already running on port 9091"` vs `"Starting transmission-daemon"`), not from an
inline `echo ALIVE`/`echo DOWN` marker pair — an earlier version tried that and always reported
"down" because the telnet session echoes back typed input verbatim, so *both* branches'
literal marker text showed up in `h2Raw` regardless of which branch actually executed. Reading
back genuine command output side-steps this. Reported in `/tmp/mesh-status.json` as
`transmission_watchdog: { status, last_check, last_restart, restart_count }`; restarts are
also appended to `/tmp/mesh-selfheal.log`.

### `mesh-poller.uc` is now the deployed daemon; `mesh-poller.sh` is retired
`/etc/init.d/mesh-poller` previously ran `/bin/sh /usr/bin/mesh-poller.sh` — silently violating
this file's own "always `.uc` on TP-Link" execution rule, undiscovered until today. `.uc` was
also missing the AP1/AP2 channel self-heal write-back that `.sh` had (it only *read* and
reported channel drift, never re-pinned it). Both gaps are now closed: self-heal write-back was
ported into `mesh-poller.uc` verbatim from `.sh`, plus the new Capability F and the Bug #2/#3
fixes above. `/etc/init.d/mesh-poller` now runs `/usr/bin/ucode /usr/bin/mesh-poller.uc`
directly. `mesh-poller.sh` is kept (both in the repo and on-router, exec bit removed on-router)
as a deprecated reference only — headers marker it accordingly; do not deploy or edit it further.
Backups from the cutover: `/usr/bin/mesh-poller.uc.bak.20260909`,
`/usr/bin/mesh-poller.sh.deprecated.20260909`, `/etc/init.d/mesh-poller.bak.20260909`.

### Bug #4: `/mnt` is a 4KB tmpfs — `start.sh`'s download/incomplete dirs were never on the USB drive at all
Root cause of "download pauses every 2-3 seconds" and a recurrence of the earlier "No space
left on device" error, found *after* Bug #1 was already fixed. `start.sh` does
`mkdir -p /mnt/usb/downloads /mnt/usb/incomplete` — but per `mount`, `/mnt` itself is
`none on /mnt type tmpfs (rw,nodev,relatime,size=4k,mode=755)`, a **4 kilobyte** tmpfs. The
real 28GB USB drive is only mounted at the nested `/mnt/usb/usb-_1` (and its `/mnt/ftproot/usb-_1`
FTP mirror) — never at plain `/mnt/usb`. So Transmission's `download-dir`/`incomplete-dir`
(set in `config/settings.json`, independent of the `BASE` path fixed in Bug #1) were pointed at
a directory with ~0 real bytes of capacity the entire time; confirmed live with
`dd if=/dev/zero of=/mnt/usb/downloads/bigtest bs=1024 count=100` → `0 bytes copied` /
"No space left on device" instantly, while `df -h` showed the real drive at only 31% used with
18.4GB free. **This is why deleting old videos never freed any space**: whatever the user
deleted lived on the real 28GB drive, but Transmission's active write target was a disconnected
4KB tmpfs that no amount of deleting elsewhere could ever affect.

**Fix, part 1 (necessary but not sufficient)**: `sed`-edited `config/settings.json`'s
`download-dir`/`incomplete-dir` to `/mnt/usb/usb-_1/downloads`/`/mnt/usb/usb-_1/incomplete`
(backed up as `settings.json.bak`), created those directories for real, fixed `start.sh`'s
`mkdir -p` line to match (future-proofing against the file regenerating). This alone produced
~15-25s of real download (confirmed via `torrent-get` RPC: `rateDownload` up to 150-170 KB/s,
growing `percentDone`) before the *exact same* "No space left on device" error recurred.

**Fix, part 2 (the actual full fix): the existing torrent's resume-state kept using the old
tmpfs path regardless of any live config change.** `torrent-set-location` (RPC) and even a full
daemon restart updated the *reported* `downloadDir`/`session-get` values, but
`find /mnt/usb -iname "*punisher*"` proved the real `.part` file was still being written to
`/mnt/usb/incomplete/...` (the dead 4KB tmpfs) the whole time — this torrent's per-torrent
resume file (`config/resume/<hash>.resume`) had baked in the broken `incomplete-dir` from
when it was first added, and neither a config edit nor `torrent-set-location` migrates that.
`torrent-remove` with `delete-local-data:true` also silently failed to clean up the tmpfs copy
(same disconnect). **The only fix that actually worked**: pulled the torrent's `magnetLink` via
RPC, manually `rm -rf`'d the stray file under `/mnt/usb/incomplete/` and its
`config/resume/<hash>.resume` + `config/torrents/<hash>.torrent`, then re-added it fresh via
`torrent-add` with an explicit `download-dir`. A fresh torrent has no stale resume state, so it
picks up the corrected paths cleanly. Also set `preallocation: 0` (was `1`/fast) along the way,
since a real 2.15GB sparse-file test (`dd ... seek=2307810093`) succeeded instantly at the
filesystem level — proving preallocation itself wasn't the blocker, just ruled out as a cause.
**Verified conclusively**: sustained, accelerating real download for 60+ seconds straight
(rateDownload 100 KB/s → 2.5 MB/s, 30 peers, `percentDone` climbing continuously, `error: 0`)
— far past the ~12-25s window every prior attempt failed at.

**Lesson**: for this class of "moved the mount, fixed the config, still broken" bug, always
check whether the *in-flight* torrent/session/resume state independently cached the old path
before concluding the config fix didn't work — a live reload or even a full process restart
does not guarantee already-loaded per-item state gets migrated to a corrected default.

**Telnet automation gotcha hit while debugging this**: an `expect` script using `expect -re {#}`
to detect the shell prompt will falsely match on any `#` character inside an *echoed* command
it just sent — including `sed`'s own `#` delimiters (`sed -i "s#old#new#"`) — causing the
script to advance to its next step mid-echo and corrupt the in-flight command. Symptom: the
sed command appears to hang/truncate with no confirmation, and the edit silently never applies.
Fix: either avoid `#` as the sed delimiter, or match the real prompt precisely
(`expect -re {\) # $}` for this device's `WAP(Dopra Linux) # ` prompt) instead of a bare `#`.

### Bug #5: AP2's MiniDLNA install had the exact same stale-tmpfs-path bug as Transmission (2026-09-10)
A previously-undocumented MiniDLNA (UPnP media server) install was found at
`/mnt/usb/usb-_1/minidlna/` on AP2, installed alongside Transmission, also with no `crond`/
init.d supervision — same unsupervised, manually-started shape as the torrent daemon. It was
down: no `minidlnad` process, nothing listening on port 8200, only a stale `minidlna.pid` left
over from a prior crashed run.

Root cause was **the same class of bug as Bug #1/#4**, not a new one: `start.sh` hardcoded
`BASE=/mnt/usb/minidlna`, and `minidlna.conf`'s `media_dir`/`db_dir`/`log_dir` all pointed at
`/mnt/usb/minidlna/...` and `/mnt/usb/downloads` — but `/mnt/usb/minidlna` didn't even exist,
and (per Bug #4) plain `/mnt/usb` is the dead 4KB tmpfs, not the real 28GB ext4 drive mounted
at the nested `/mnt/usb/usb-_1`. Had `start.sh` been run as-is, it would have silently created
`db`/`log` dirs on the 4KB tmpfs and pointed `media_dir` at an empty directory — the identical
failure mode Bug #4 already diagnosed for Transmission, just never previously noticed here
because nothing had tried to start this service in a while.

**Fix**: `sed`-edited both `start.sh`'s `BASE` and `minidlna.conf`'s `media_dir`/`db_dir`/
`log_dir` to the real `/mnt/usb/usb-_1/minidlna`/`/mnt/usb/usb-_1/downloads` paths, cleared the
stale pid file, reran `start.sh`. Verified live: MiniDLNA 1.3.3 started, scanned
`/mnt/usb/usb-_1/downloads` (3 real files), and is listening on `0.0.0.0:8200`.

**Process note**: per user instruction, `mesh-poller` was stopped on TP-Link before this
manual telnet session and restarted after — see Bug #3 above (AP1/AP2's one-concurrent-
telnet-session limit means a manual session run alongside the poller's own per-cycle telnet
connections collides and can lock/stick the CLI).

**Lesson**: any future USB-drive-hosted daemon on AP2 (or a fresh install of one) should have
its `BASE`/`media_dir`/`download-dir`/equivalent path checked against the real
`/mnt/usb/usb-_1` mount *before* first start, not after a failure — this is now the second
service to ship with a path pointed at the dead 4KB `/mnt/usb` tmpfs instead of the real
nested mount.

### Permanent hardening: stable `/mnt/usb/data` symlink so the `usb-_N` index can never go stale again (2026-09-10)
Bug #1/#4/#5 are all the same root cause recurring across two services: something hardcoded
the raw `/mnt/usb/usb-_1` mount path, and that index (`usb-_1`) is Huawei's own USB-hotplug
numbering, not guaranteed to stay at `_1` across a future replug or remount. Rather than wait
for a third recurrence, added one layer of indirection: `/mnt/usb/data` is now a symlink on
AP2 pointing at whichever `usb-_N` directory is actually mounted. Both Transmission's and
MiniDLNA's `start.sh` (`BASE=`) now point at `/mnt/usb/data/...`, as does MiniDLNA's
`minidlna.conf` (`media_dir`/`db_dir`/`log_dir`) and `mesh-poller.uc`'s `TRANSMISSION_DIR`
constant. None of these reference the raw `usb-_1` path anymore.

`mesh-poller.uc` self-heals this symlink once at every poller startup (`healUsbDataSymlink()`,
called right before `uloop.init()`) — a fresh `ln -sfn` against whichever `usb-_*` glob match
exists, logged to `mesh-selfheal.log`. This runs once per poller start (service
start/restart, router reboot), not every poll cycle: the mount only changes on
reboot/replug, not every ~20s, so re-running it every cycle would be pure overhead — and
doing exactly that during development directly caused the regression below.

**Regression introduced and caught during this same fix**: an earlier version of this change
folded the symlink-refresh command into the per-cycle AP2 shell session (the same session
Capability F's transmission watchdog uses, on a fixed ~18s kill budget). That pushed the
session past its budget, causing `transCheckCmd`'s `cat` to run before `start.sh`'s real
output was ready — `h2Raw` ended up matching `/Starting transmission-daemon/` every single
cycle even though Transmission was genuinely up the whole time, and the watchdog dutifully
"restarted" an already-running daemon every ~20-60s (`mesh-selfheal.log` restart_count
climbing #1→#4 in minutes). Caught immediately by checking `mesh-selfheal.log` after deploy
instead of assuming success. Fixed by moving the symlink heal to poller-startup-only as
described above.

**Second bug from the same edit, caught the same way**: the fix-for-the-fix removed the
per-cycle `usbSymlinkCmd` variable's *definition* but missed one lingering reference to it in
the `h2ShellExtra` `sprintf(...)` call — an undefined-variable reference in ucode evaluates to
null rather than erroring, so `%s` silently rendered the literal string `"null"` as an extra
line typed into AP2's live busybox shell every cycle, corrupting the rest of that cycle's
command sequence (explains why the first "fix" deploy still showed the restart loop — the
stray `null` was still there). Caught by re-`grep`ping the file for the variable name after
editing, not just by re-running `ucode -c` (a stray-but-syntactically-valid reference doesn't
fail syntax check). **Lesson: after deleting a `let`, grep the whole file for the name before
redeploying** — a syntax check alone does not catch a dangling reference to a since-removed
local variable.

Verified live after both fixes: `mesh-selfheal.log` shows zero new restart entries and
`transmission_watchdog.restart_count` held at `0` across multiple poll cycles, while AP2's
telnet CLI remained reachable (not stuck) throughout.

### Bug: Nokia client counts always read zero, hiding most of the mesh's real clients (2026-09-11)

Dashboard only ever showed the 4 devices directly associated to TP-Link — every Nokia-attached
client (routinely 8-11 devices on `U101-slow`/`U101`) was invisible, even though a direct
`show wifi client list` on the gateway proved they were genuinely connected. Root cause: the
regex in `mesh-poller.uc`'s Nokia-parsing loop (`cliM = match(l, /([0-9A-Fa-f:]{17})\s+([^\s]+)\s+(2\.4G|5G)/)`)
expected a `2.4G`/`5G` band token on the *same line* as each client row. The real
`wifi client list` output prints the band once per section header
(`OperatingFrequencyBand  = 2.4GHz` / `= 5GHz`) followed by a `DeviceName  MACAddress  Status`
table with no band token on individual rows at all — so the regex never matched a single real
client line, and `nokia24_count`/`nokia5_count` silently stayed `0` every cycle.

**Fix**: track the current band via the `OperatingFrequencyBand` header line instead of
expecting it per-row, and match client rows by anchoring on `<MAC>\s+(Connected|Not Connected)`
(only counting `Connected`). The anchor-on-status approach also sidesteps a real formatting
landmine: unresolved devices get an auto-generated name of `Unknown_<their own MAC>`, which
sits directly against the real MACAddress column with no separating space (e.g.
`Unknown_08:3a:2f:87:51:b108:3a:2f:87:51:b1       Connected`) — anchoring the match on the
MAC immediately preceding `Connected`/`Not Connected` reliably lands on the real MACAddress
column regardless of what precedes it. Verified live: `nokia_wifi.clients_24g`/`clients_5g`
went from `0`/`0` to `8`/`3`, and the dashboard's total client count went from 4 to 13 (4 on
TP-Link + 9 newly-visible Nokia clients, after dedup against clients already seen on other
radios). Deployed copy backed up as `/usr/bin/mesh-poller.uc.bak.20260911` on the router.

### Bug: Observer never actually watched Nokia's radio despite claiming "ALL RADIOS OPTIMAL" (2026-09-11)

`mesh-observer.uc`'s `evaluate_cycle` accepted Nokia's composite congestion score (`n_cov`, its
6th CLI argument) but never referenced it anywhere in the `isCongested` check — only TP-Link's
airtime and both Huawei APs' congestion were evaluated. So Nokia's radio could be arbitrarily
congested and the "🟢 CLEAN (ALL RADIOS OPTIMAL)" badge would still show, and it could never
trigger the auto-optimizer on Nokia's behalf. Fixed by adding an `n_cov > 40.0` branch mirroring
the existing Huawei checks. Not visible under current load (Nokia's synthetic `overall` score is
36.2% with 11 clients, genuinely under the threshold), but the gap was real and is now closed.

### Bug: "Probe Spectrum & Optimize" button's self-heal sync step silently patched a dead file (2026-09-11)

The dashboard's optimizer button (`mesh-api?action=optimize` -> `mesh-optimizer.uc --apply`)
genuinely works for the actual per-AP channel application — verified live twice, all 4 nodes
reconfigured successfully both times. But its last step, re-pinning `mesh-poller.uc`'s self-heal
watchdog targets so a newly discovered channel plan survives a future AP reboot, was `sed`-ing
`/usr/bin/mesh-poller.sh` — the file retired during the `.sh`->`.uc` cutover (§16) — using a
shell-variable pattern (`EXPECTED_H1_5G_CH=...`) that wouldn't even match `mesh-poller.uc`'s real
`let EXPECTED_H1_5G_CH = "...";` syntax if it had targeted the right file. The step's own log
line ("Updating watchdog daemon targets in `/usr/bin/mesh-poller.sh`") was the tell. Silent
because the discovered plan currently always matches the already-hardcoded values, so nothing
visibly broke. Fixed by targeting `mesh-poller.uc` with the correct `let`-statement pattern,
plus a backup (`mesh-poller.uc.bak.autoheal`) and `ucode -c` syntax-check gate before restarting
the service, restoring the backup if the patched file fails to compile. Verified live: the log
now reads "...targets in `/usr/bin/mesh-poller.uc`", the live file's `EXPECTED_*` constants were
genuinely rewritten, and the poller restarted cleanly under a fresh PID.

### Bug: Dashboard's Device Hostname & IP column always showed the "192.168.0.x" placeholder (2026-09-11)

Same bug class as the two above, third instance found in the same file. The `dhcpM` regex that
parses Nokia's `lan ip hosts` output (`([0-9A-Fa-f:]{17})\s+([0-9.]+)\s+([^\s]+)`) assumed
column order MAC -> IP -> hostname. The real table is
`Status | ConnType | Allocation | IPv4Address | HardwareAddress | HostName` -- IP comes
*before* the MAC, the reverse of what the regex expected -- so it never matched a single real
row, `dhcpHosts` stayed permanently empty, and every client fell back to the hardcoded
`"192.168.0.x"` placeholder for both IP and (when no vendor cache hit existed either) name.
Confirmed via a live capture: `Active  Ethernet  DHCP  192.168.0.4  98:3d:ae:55:e2:6c espressif`.
Fixed by matching IP first, then MAC, then hostname, and swapping which capture group feeds
`ip` vs. the `dhcpHosts` key. Verified live: real IPs (`192.168.0.2`, `.11`, `.29`, `.13`, `.3`,
`.14`, `.5`, `.17`, ...) now populate for every client Nokia's DHCP table actually has an entry
for; a handful of clients Nokia genuinely has no lease record for still correctly show the
placeholder (expected, not a bug). Deployed copy backed up as
`/usr/bin/mesh-poller.uc.bak.20260911b` on the router.

**Note for next session**: `a4:f9:33:51:9b:cc`'s resolved name renders as
`"Intel Corporateac:19:8e:ce:e3:39|Intel Corporate"` -- looks like a separate, pre-existing
vendor-cache concatenation bug, not yet investigated.

### Capability G: AP2 USB Storage File Management & Deletion Safety Net (2026-09-12)

When torrents were historically removed from Transmission via third-party or native interfaces without checking "delete local files" (or prior to the Bug #4 path fix when Transmission pointed to `/mnt/usb/downloads`), physical media files were left orphaned on the USB drive (`/mnt/usb/usb-_1/downloads/`). Because the dashboard previously only rendered active items returned by Transmission's `torrent-get` RPC, orphaned files were completely invisible and undeletable from the UI, while MiniDLNA continued indexing and streaming them to home TVs.

**Implemented End-to-End USB Storage & File Management**:
1. **Live Directory Auditing (`mesh-poller.uc`)**:
   During `mesh-poller`'s AP2 shell session, `ls -la /mnt/usb/usb-_1/downloads` is executed between strict line markers. Terminal echo and ANSI color escape codes (`\x1b[...m`) are stripped, and entries are parsed into structured file records (`name`, `is_dir`, `size`, `date`) exported in `/tmp/mesh-status.json` under `usb_files`.
2. **Collision-Free Asynchronous Deletion Queue (`mesh-poller.uc` + `mesh-api.uc`)**:
   Direct telnet access from CGI would collide with `mesh-poller`'s strict single-session limit on AP2. Deletions are queued via `/tmp/mesh-pending-usb-deletes.txt` with strict basename sanitization (preventing traversal `..` or command injection). On the next poll cycle, `mesh-poller` executes `rm -rf "/mnt/usb/usb-_1/downloads/<sanitized_name>"` inside its own shell session and logs to `/tmp/mesh-selfheal.log`.
3. **Double Safety Net on Torrent Removal (`mesh-api.uc`)**:
   When `torrent_action&cmd=remove` is triggered from the active downloads table, `mesh-api.uc` not only sends `delete-local-data: true` to Transmission RPC, but also queues the physical torrent directory in `/tmp/mesh-pending-usb-deletes.txt` so incomplete or directory-level leftovers are guaranteed to be purged from the USB drive.
4. **Dedicated UI Panel (`mesh.html`)**:
   Under the Torrent Hub tab, a new **"📁 Downloaded Media & Files on USB Drive"** table renders all physical files/directories currently residing on the USB drive, showing real-time disk sizes, file types (🎬 Media / 📁 Directory / 📄 File), whether they match an active download, and a 1-click **"🗑 Delete File"** confirmation button.
5. **Auto MiniDLNA Refresh**:
   MiniDLNA's kernel inotify watcher detects removals immediately and updates its media library (`http://192.168.0.98:8200/`), instantly freeing disk space and removing dead items from smart TV DLNA menus.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
