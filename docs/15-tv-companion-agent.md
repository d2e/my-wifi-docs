## 📱 15. Native Android TV Companion Agent & Real-Time OTT Telemetry (2026-09-04)

### 1. The Core OTT Metadata Problem
When video streaming apps (Disney+ Hotstar, Netflix, Prime Video) play on Android TV, their developers rarely populate full Android `MediaSession` metadata:
- Hotstar's ExoPlayer/Media3 player sets only the primary `title` field to a combined string (e.g. `"S15 E2 The Stone I Surely Want To Pick Up"` or `"S3 E25 Fairytale"`), leaving `subtitle`, `description`, `artist`, and `seriesTitle` as `null`.
- Standard ADB CLI (`adb shell dumpsys media_session`) merely prints `MediaDescription.toString()`, discarding player duration, series names, and custom extras.
- On the dashboard, generic title cleaners previously stripped `S3 E25`, leaving `"Fairytale"`, which blindly hit the Apple iTunes Song Search API and matched Alexander Rybak's 2009 Eurovision pop song `"Fairytale"` (Genre: POP, Duration: 183s / 03:03), corrupting the scrubber into `04:35 / 03:03`.

### 2. Frontend Metadata Hardening (`mesh.html`)
- **Strict Video vs. Music Isolation**: Video streaming apps (`hotstar`, `netflix`, `prime`, `jiocinema`, `zee5`, `sonyliv`, `apple tv`, etc.) and any titles matching episode syntax (`S\d+\s*E\d+`) are permanently isolated and blocked from querying the iTunes Song Search API.
- **Dedicated TV Episode Resolver**:
  - Automatically decomposes Season, Episode, and Episode Title.
  - Queries iTunes specifically with `entity=tvEpisode`, resolving canonical series names (e.g. `Bluey`), 600×600 artwork, and real episode runtimes.
  - Features a curated fallback catalog mapped to TVMaze global show data (e.g. resolving `"The Stone I Surely Want To Pick Up"` to `Crayon Shin-chan`, 1992, Comedy/Anime).
- **Scrubber Defense**: Guarded with `if (dur > 0 && dur >= pos)` across both Mi TV and Fire TV cards so impossible durations or progress fractions can never render.

### 3. Native Android TV Companion Agent (`tv-telemetry-agent`)
To eliminate all guessing and static catalog maintenance, a native, zero-bloat (20 KB) background agent (`com.home.tvagent`) was built and installed directly onto the Xiaomi Mi QLED TV (`192.168.0.10`):

- **`TvAccessibilityService` (On-Screen UI Inspector)**:
  - Listens to Android layout and window events (`AccessibilityEvent`).
  - Inspects the live `AccessibilityNodeInfo` tree when playback starts or controls appear.
  - Captures whatever is rendered on the screen: Series Title (`Crayon Shin-chan`), Episode Title, and Synopses.
- **`TvMediaSessionService` (MediaController Engine)**:
  - Binds to active media sessions via `NotificationListenerService`.
  - Reads millisecond-precision playback position and **exact player runtime duration** (`METADATA_KEY_DURATION`, e.g. `1483s / 24m 43s`).
  - Extracts direct provider CDN poster/thumbnail URLs (`METADATA_KEY_ART_URI`).
- **`LocalHttpServer` (Micro-Daemon on Port 8080)**:
  - Zero-dependency pure Java `ServerSocket` daemon (< 50KB memory footprint).
  - Serves `http://192.168.0.10:8080/nowplaying` in **sub-2ms** with CORS support.
- **`BootReceiver`**: Automatically restarts the daemon upon TV power-on (`BOOT_COMPLETED` / `QUICKBOOT_POWERON`).

### 4. ADB Permissions & Lifecycle Management
Granted via ADB (persists across reboots, zero remote interaction needed):
```bash
# Append to enabled accessibility services alongside Projectivy Launcher:
adb -s 192.168.0.10:5555 shell settings put secure enabled_accessibility_services "com.spocky.projengmenu/.services.ProjectivyAccessibilityService:com.home.tvagent/.TvAccessibilityService"
adb -s 192.168.0.10:5555 shell settings put secure accessibility_enabled 1

# Grant Notification / MediaSession listener access:
adb -s 192.168.0.10:5555 shell cmd notification allow_listener com.home.tvagent/.TvMediaSessionService

# Start daemon:
adb -s 192.168.0.10:5555 shell am start-foreground-service -n com.home.tvagent/.HttpServerService
```

### 5. Router Integration (`/usr/bin/entertainment-poller.uc`)
The router polling daemon on JioRouter (`192.168.0.50`) now queries the TV agent before falling back to ADB:
```ucode
// Fast path: query native on-device HTTP agent (sub-2ms)
let pAgent = fs.popen(sprintf("curl -s -m 1 http://%s:8080/nowplaying 2>/dev/null", TV_IP));
let agentJson = pAgent ? (pAgent.read("all") || "") : "";
if (pAgent) pAgent.close();
// Falls back to parseMediaSession(tvRaw) if agent is offline
```

### 6. Verified Live Contract
```json
{
  "app_pkg": "in.startv.hotstar",
  "app_name": "Disney+ Hotstar",
  "title": "The Stone I Surely Want To Pick Up",
  "season": 15,
  "episode": 2,
  "state": "playing",
  "position_sec": 1239,
  "duration_sec": 1483,
  "timestamp": 1788531079
}
```
Result: The dashboard renders `The Stone I Surely Want To Pick Up`, `🎬 Crayon Shin-chan • Season 15 • Episode 2`, exact progress `20:39 / 24:43` (84%), high-resolution anime artwork, and zero fake song durations.

### 7. Shizuku Server Keep-Alive & Post-Reboot Persistence (2026-09-05)
To support privileged tools and advanced remap/accessibility utilities on the Mi TV without root:
- **Dynamic Binary Resolution**:
  On unrooted Android 10, `/data/app` permissions prevent wildcard glob expansion (`ls /data/app/moe.shizuku.privileged.api*` fails with permission denied). The dynamic binary path is resolved via:
  ```bash
  SH_LIB=$(pm path moe.shizuku.privileged.api | sed -e 's/package://' -e 's|/base.apk|/lib/arm/libshizuku.so|')
  ```
- **Autonomous Post-Reboot Reviver (`entertainment-poller.uc`)**:
  Integrated into the router's 24/7 background keep-alive loop (`ts % 30 < 4` when TV is awake):
  ```bash
  pidof shizuku_server >/dev/null || { S=$(pm path moe.shizuku.privileged.api 2>/dev/null | sed -e "s/package://" -e "s|/base.apk|/lib/arm/libshizuku.so|"); [ -x "$S" ] && "$S" </dev/null >/dev/null 2>&1 & }
  ```
  Within 30 seconds of any TV reboot, `shizuku_server` automatically starts under the `shell` UID with zero manual intervention or USB/PC connection required.

  **This reviver never actually ran between its introduction (commit `2ab231f`) and
  2026-09-09.** The generated shell string joined the two halves as
  `... shield_remap.sh </dev/null >/dev/null 2>&1 &; pidof shizuku_server ...`, and `&;` is a
  POSIX shell syntax error (`/system/bin/sh: syntax error: unexpected ';'`). Because the parse
  failed, *neither* the `shield_remap.sh` relaunch nor the shizuku revive ever executed — the
  entire keep-alive silently no-oped while looking correct in the source. It was masked
  because both processes happened to already be running when the feature was added, so the
  "verified passing live" check below was really only observing manually-started processes,
  not the reviver. Caught for real after a TV reboot left both dead for ~1h44m. Fixed by
  dropping the semicolon (`2>&1 & pidof ...`). Covered in `regression-test.uc` §3c, which
  asserts the `&;` sequence is absent from the generated string.

  **Lesson**: a keep-alive that runs `adb shell` with a compound command can fail *entirely*
  at the parse stage and still look healthy — the router-side wrapper redirects stderr to
  `/dev/null`, so the syntax error was never surfaced anywhere. When adding or editing this
  block, run the inner command by hand with stderr visible before trusting it.
- **`pidof` vs `pgrep` Gotcha**:
  On this MediaTek Android 10 build, `pgrep shizuku_server` exits with code 0 even when no process matches. `pidof` was verified to reliably exit 0 when running and exit 1 when absent.
- **Hardware Remote Remap Integration (`shield_remap.sh`)**:
  Updated to use `pidof shizuku_server` to verify daemon status before attempting invocations.
- **Regression Verification**:
  Covered under `regression-test.uc` Section 2b ("shizuku_server is running on the Mi TV under shell UID (automated reboot persistence)"). Note §2b only observes the *end state* (is the process up?), which is why it passed for days against a reviver that never ran — §3c was added to assert the reviver's shell string is actually parseable. Suite is 42 assertions green as of 2026-09-09.

  **Known remaining issue (not fixed)**: the reviver can spawn *duplicate* `shield_remap.sh`
  instances. Its "already running?" probe scans `/proc/*/cmdline` for `shield_remap` and skips
  only its own `$$`, but the wrapper `sh -c` process's cmdline also contains that literal
  string, so detection is fragile; two live instances were observed after the fix. Benign
  (it's a key-event listener) but it means duplicate remote-key handling is possible.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
