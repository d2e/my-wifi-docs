## 🔍 14. Dashboard Backend Audit & Security Hardening (2026-09-04)

A full review of the "Smart TV & Streaming" dashboard backend (`mitv-api.uc`, `firetv-api.uc`, `entertainment-poller.uc`, `mesh.html`) surfaced and fixed six real bugs, one of them critical. All fixes were verified live against the actual hardware, not just code-reviewed.

### Recurring root cause: `adb shell` argument-length limit
**This router's `adb` client fails outright with `error: service name too long` on a combined multi-field `adb shell 'echo A:$(...); echo B:$(...); ...'` command once too many fields are chained in one call — and fails *silently* from the caller's perspective: every field parsed out of the (empty) response falls back to its hardcoded default, with no error surfaced anywhere.** This bit twice in the same session:
- `mitv-api.uc`'s `display_state` handler (15 fields in one call) — fixed by batching into groups of 5 separate `adb shell` calls, concatenating the raw output before regex-parsing.
- `entertainment-poller.uc`'s TV telemetry query (originally ~9 lines, broke again the moment a 10th field — `sound_auto_volume_control` — was added) — fixed by splitting into two sequential `adb shell` calls (system/RAM/DF/TEMP/PWR/WIN in one, AUDIO/HDMI/MEDIA in the other).
**Any future addition of a new `settings get`/`dumpsys` line to either of these combined-query blocks risks re-triggering this exact failure mode.** Test by running the full combined command by hand over SSH before trusting a new field's output.

### Bugs found and fixed
1. **`display_state` (Mi TV Calibration panel) silently defaulted almost every field** — root cause above. Also missing a `sound_auto_volume_control` (DRC) field entirely, separate from the length issue.
2. **Color Temperature / Local Dimming `<select>` dropdowns wrote `0` (Normal/Off) no matter what was picked** — the generic numeric write path did `int(param)` on words like `"warm"`/`"high"`; ucode's `int()` on a non-numeric string is `NaN`, and `sprintf("%d", NaN)` renders as `"0"`. Fixed with explicit enum lookup tables (`color_temp`: normal=0/warm=1/cool=2, `local_dimming`: off=0/low=1/medium=2/high=3) checked before falling through to `int(param)`.
3. **Audio Output toggle (Sony ARC vs TV Internal Speakers) never reflected the real selection** — the UI keyed off `arc_active` (the physical HDMI-ARC link-established flag, `mArcEstablished`), which stays `true` even after switching `sound_output_switch` away from ARC. Added a real `audio_output_type` field to `entertainment-poller.uc` (derived from `sound_output_switch`), and made `mesh.html` key the highlight off that instead.
4. **Command injection via `launch`/`force_stop`/`open_url`/`text` (both `mitv-api.uc` and `firetv-api.uc`) — CRITICAL.** `adb shell <cmd>` is a double-hop: the local ucode `system()` call's shell parses the command string once, then `adb` forwards the resulting argv to the *device's own* shell, which parses it again. A single layer of local quoting around the attacker-controlled `param` (raw query-string value) only protects the first hop — e.g. `param="pkg'; reboot; echo '"` would execute `reboot` as a separate command on the TV/Fire TV. Verified live with a real payload (`'; touch /sdcard/PWNED_TEST; echo '`) both before (file created) and after (blocked) the fix.
   - **Fix**: added `escSQ()` (escapes `'` → `'\''`) applied *twice* — once to safely single-quote `param` for the remote device shell, once more to safely single-quote the resulting whole remote-command string for the local shell. Single quotes are the only character with special meaning inside single-quoted POSIX shell strings, so this closes both hops without needing to separately handle `$`, backticks, or `;`.
   - Added defense-in-depth package-name whitelisting (`^[A-Za-z0-9_.]+$`) for `launch`/`force_stop`, which have no legitimate reason to contain shell metacharacters at all.
5. **On-screen keyboard text mangled any punctuation beyond a 6-item whitelist** — the hand-rolled `urlDecode()` only decoded `%20 %3A %2F %3F %3D %26` explicitly; `encodeURIComponent()` on the frontend also escapes `,` `#` `@` `%` and more, which arrived at the backend still literally as `%2C`/`%40`/etc. and got typed onto the TV as garbage. Fixed by decoding any `%XX` sequence generically via `replace(str, /%([0-9A-Fa-f]{2})/g, function(m) { return chr(hex(substr(m,1,2))); })` instead of enumerating a subset.
6. **"Night" and "Voice" sound profiles were indistinguishable in telemetry, and Voice could silently inherit Night's leftover state** — both write `sound_style=3`; only the DRC flag (`sound_auto_volume_control`) actually distinguishes them, and only Night ever touched that flag. Fixed by (a) having `entertainment-poller.uc` label code-3 as `"Night Mode"` when DRC is on, `"Voice Boost"` otherwise, and (b) making Voice/Cinema/Music/Standard explicitly reset `sound_auto_volume_control=0` on write, so each profile is a clean, deterministic state regardless of what was pressed before.

### Also fixed (non-bug)
- Mi TV native app-name mapping in `entertainment-poller.uc` was missing a `smarttube` case (Fire TV's list already had it) — SmartTube played as the raw package name `org.smarttube.stable` instead of a friendly label, which *also* silently broke the dashboard's Tier-3 generic-poster fallback (it string-matches `activeAppName.toLowerCase().includes('youtube')`, which never matched the raw package name). One fix resolved both symptoms.
- Mi TV tab's 📸 Screen Preview panel enlarged (`min-height` 160→420px, image `max-height` 280→640px) in `mesh.html`.

### Known limitation — Solved by Native Companion Agent (2026-09-04)
Previously, real per-video thumbnails and series titles were not retrievable over stock ADB because `dumpsys media_session` only prints `MediaDescription.toString()` and drops custom bundles. This limitation is now **completely resolved** by the native `tv-telemetry-agent` companion APK running on the TV (§15).

### Episode number silently dropped from the Now Playing display (2026-09-04)

When the show-enrichment lookup moved server-side into `tv-telemetry-agent` (see above), the
`agent_verified` short-circuit added to `mesh.html`'s `resolveShowMetadata()` only returned
`title`/`album`/`artist`/`poster`/`tier` — it never carried `extraMeta.subtitle` (the
"Season N • Episode M" text) through. The rendering code builds the displayed line purely from
the RESOLVED `meta.album`/`meta.artist`/`meta.show`, never from the raw `now_playing.subtitle`
field directly, so the episode number silently disappeared for all Mi TV content the moment
the agent started providing a verified poster (i.e. as soon as the previous fix started
working). Fixed by combining show + season/episode into `album`, matching the pattern every
other tier already uses (e.g. `${jw.show} • Season ${sNum} • Episode ${eNum}`). Verified against
real live data: now renders "🎬 Thukra Ke Mera Pyaar • Season 2 • Episode 4" correctly.

**Known test-coverage gap**: this is frontend JS with no test framework in this project (the
live smoke suite is ucode-based and can't execute browser JS) — verified this fix by manually
running the exact rendering logic against the real live data via Node, not via an automated
regression check. A real JS test harness for `mesh.html`'s rendering logic would close this gap
if this class of bug recurs.

### Dashboard's ARC/sound-mode status was permanently hardcoded, hiding the real flapping bug (2026-09-04)

While live-diagnosing the Sony HT-S40R ARC-flapping issue (see "CEC/ARC investigation" below),
check the dashboard for any visible indication turned up nothing — because there wasn't any
*to* find. `sony_hometheater.arc_handshake` in `entertainment-poller.uc`'s JSON output was a
literal hardcoded string `"ACTIVE"`, never actually reading the real `tvArc` value that sits
right next to it in the same payload (`tv.arc_active`). Same bug, same object:
`sony_hometheater.sound_mode` was hardcoded to `"Voice Boost / Cinema 5.1"` regardless of the
real `tv.sound_profile`. Both are leftover remnants of this dashboard's original 100%-fake-data
incarnation (documented at the top of this file) that never got wired to real values during
the rebuild, unlike everything else in the same JSON object. Fixed: `arc_handshake` now reads
`tvArc ? "ACTIVE" : "INACTIVE"`, `sound_mode` now reads `tvStyle` directly. Verified live: it
now correctly shows "INACTIVE" during an actual real-time ARC drop. Covered in
`regression-test.uc` (cross-checks both fields against `tv.arc_active`/`tv.sound_profile`).

### CORRECTION: the CEC/ARC "Sony firmware bug" conclusion below was wrong — a full TV reboot fixed it completely (2026-09-04)

The investigation below concluded, with what seemed like exhaustive evidence, that the ARC-
flapping was a permanent Sony HT-S40R firmware defect. **That conclusion was wrong.** The user
noted this had worked fine until earlier that same day — not "always broken," which doesn't fit
a baked-in hardware/firmware defect. A full reboot of the Mi TV (not a settings toggle, not the
targeted `hdmi_control_enabled` subsystem restart tried earlier — an actual full device reboot)
fixed it completely: verified zero ARC flap/re-negotiation messages across two consecutive
monitoring windows (~220+ seconds combined) post-reboot, versus 13+ drop/reconnect cycles in a
comparable window beforehand.

**Most likely explanation**: this same debugging session did extensive live CEC/ARC settings
toggling on the TV (`hdmi_control_enabled` off/on multiple times, several `resync_arc` calls,
disabling/re-enabling Fire TV's CEC) while chasing the root cause — and that experimentation
itself most likely left the TV's in-memory HDMI-CEC/audio subsystem state degraded, which is
exactly the kind of thing a settings toggle can't fully clear but a genuine reboot does.

**Lesson**: when CEC/ARC starts misbehaving, try a full device reboot FIRST, before any
protocol-level log diagnosis — a plausible-looking, evidence-backed root-cause story ("every
message in the trace comes from Sony, therefore it's Sony's firmware") can still be wrong if it
doesn't fit the actual timeline the user reports ("this worked yesterday"). Timeline info like
that should have prompted trying the simplest fix immediately, not after extensive elimination
testing of every other component.

### CEC/ARC investigation: Sony HT-S40R has a persistent ARC-flapping firmware bug (2026-09-04) — SUPERSEDED, kept for the diagnostic technique/evidence, not the conclusion

User-reported symptom: audio audibly switching between TV speakers and the Sony soundbar, and
"something wrong with CEC" / "Fire TV fighting for CEC". Full live investigation, every
hypothesis tested empirically rather than assumed:
- **Fire TV ruled out, twice.** Its own CEC traffic (logical address 0x04) is normal one-time
  discovery handshake only — zero ARC/audio-related messages, confirmed via direct CEC log
  filtering. Fully removing it from the CEC bus (disabling its `hdmi_control_enabled`) did NOT
  stabilize ARC — it broke ARC establishment entirely (`mArcEstablished` stuck `false`, no
  negotiation attempts at all even after an explicit resync). Reverted immediately.
- **A real, separate bug WAS found and fixed**: a stuck CEC action-queue entry referencing Fire
  TV's logical address (`HdmiCecLocalDevice: check if pending cleaered[4], FAIL due to actions
  non-empty!!!`, spamming logcat dozens of times/minute) on the TV's own MediaTek CEC stack.
  Cleared by toggling `hdmi_control_enabled` off/on on the TV (full CEC subsystem restart). This
  fixed the log spam permanently but had **zero effect** on the ARC flapping itself — proving
  they were two independent issues, not cause-and-effect.
- **The Sony soundbar itself is the party actually flapping ARC.** Every message in the raw
  CEC trace is tagged `[R]` (received BY the TV FROM Sony) — the TV never initiates any of it;
  it's purely reactive to Sony's own soundbar repeatedly terminating and re-initiating ARC on
  its own, every ~4-12 seconds.
- **Cable ruled out** (user confirmed it's the original Sony-supplied cable, not a mismatched
  third-party one).
- **Soundbar power cycle (full unplug, not standby) did NOT fix it** — the same flapping pattern
  (Terminate/Initiate ARC every ~4-12s, all `[R]` from Sony) resumed immediately after a fresh
  boot, proving this is baked into the soundbar's current firmware behavior, not a transient
  stuck state.
- **Confirmed no script in this project causes it** — audited every call site of `resync_arc`/
  `audio_output`/`surround`; all are exclusively triggered by explicit dashboard button clicks,
  never by the automatic poller loop (which only reads CEC/ARC state, never writes it).
- **Latest public HT-S40R firmware appears to be ~mid-2022** (Sony India: 1.387/2.077; other
  Sony regions: 1.388/2.078) — Sony's support pages return HTTP 403 to programmatic fetches
  (same Akamai-style bot-blocking as Hotstar), so exact release dates couldn't be independently
  verified beyond "multiple sources agree on mid-2022, several years stale."

**Conclusion**: this is a genuine firmware defect inside the Sony HT-S40R's own ARC/CEC
implementation, unrelated to Fire TV, the cable, the TV's CEC stack, or anything in this
project's automation — every lever reachable from the router/ADB side has been tested and
ruled out. Remaining options are outside what's scriptable here: check for/install a newer
regional firmware, contact Sony support, or fall back to Optical/S-PDIF output (sidesteps the
CEC/ARC handshake entirely, at the cost of losing CEC-based volume/power sync).

### Regression test itself was opening a browser on the TV every run (2026-09-04)

User-reported bug: a browser kept opening on the Mi TV. Root cause was `regression-test.uc`'s
own command-injection defense check — it sends `cmd=open_url&param=http://x.test/foo'; ...`
to verify the shell-injection portion doesn't execute, but the underlying Android
`android.intent.action.VIEW` intent still legitimately resolves `http://x.test/foo` and
opens the default browser (`org.chromium.webview_shell`) for it. Confirmed live: three
stacked `WebViewBrowserActivity` instances had accumulated from repeated test runs this
session. Fixed by switching the payload to a made-up, unregistered URL scheme
(`regressiontestnoop://`) — `am start -a VIEW` then fails to resolve any app at all (verified:
"unable to resolve Intent"), so nothing visibly opens, while still testing the exact same
thing (do the embedded shell metacharacters execute as a separate command). Also added an
`am force-stop org.chromium.webview_shell` cleanup as defense in depth, in case a future
payload change reintroduces a real scheme.

### Fire TV Home showed as raw activity string (2026-09-04)

Dashboard bug: Fire TV's active-app display showed
`com.amazon.tv.launcher.ui.HomeActivity_vNext` instead of "Fire TV Home" whenever Fire TV was
sitting at its own launcher/home screen. Root cause: `mCurrentFocus` in `dumpsys window`
captures the ACTIVITY name, not the package, and `resolveFtvAppName()` (poller-lib.uc) only
checked for `prime|kuro` to detect the Fire TV launcher — the DSP auto-switching logic in
`entertainment-poller.uc` already correctly matched `launcher|amazon\.tv\.launcher` for this
same case, so the two had drifted out of sync. Fixed by adding the same pattern to
`resolveFtvAppName()`, plus filling in two other apps from the real installed roster (ZEE5,
MX Player) that were missing from this function entirely. Covered in `test-poller-lib.uc`
with the exact live fixture string.

### Critical: JUnit passing does NOT prove Android/ART regex compatibility (2026-09-04)

After the accent-normalization fix above shipped, live on-device testing (not just the JUnit
suite, which had already gone green) immediately hit a real crash in logcat:
```
ShowEnrichment: enrich(Pokemon): java.lang.IllegalArgumentException: Unsupported flags: 256
```
`Pattern.compile("\\p{Mn}+", Pattern.UNICODE_CHARACTER_CLASS)` is valid on the plain JVM
(where JUnit tests run — they passed) but Android's ART regex engine rejects the
`UNICODE_CHARACTER_CLASS` flag outright. **This is exactly the gap the `/test` skill's rule 3
warns about** ("a passing unit test suite alone doesn't prove the APK still runs") — it just
took a real occurrence to make the point concrete. Fixed by replacing the Unicode
property-class + flag with an explicit `[̀-ͯ]` (Combining Diacritical Marks block)
character range, which needs no special flag and covers what NFD decomposition actually
produces. Verified live end-to-end after the fix: real Hotstar Pokémon episode →
`show: "Pokemon"` → resolved `poster_url` → confirmed real 200 webp image → confirmed present
in `entertainment-status.json`'s `now_playing.poster_url` all the way through to the poller.

**Lesson for any future `java.util.regex` usage in this app**: stick to basic character
classes/ranges; avoid `\p{...}` Unicode property escapes and any `Pattern.*` flag beyond the
basics (`CASE_INSENSITIVE`, `MULTILINE`) without verifying on a real device first — the JVM
test suite cannot catch this class of incompatibility, ART's regex engine (not the same
implementation as the JDK's) is meaningfully less complete.

### Two more real bugs found by testing the discovery live (2026-09-04)

Checking the enrichment feature against live TV state (not just the JUnit fixtures) surfaced
two more real bugs in `TitleMatcher`/`isPlausibleTitleMatch`, both now fixed and covered:
1. **Accented characters silently deleted, not normalized.** `[^a-z0-9]` treats `é` as
   punctuation and strips it — verified live: JustWatch's real top result for "Pokemon" is
   "Pokémon"; `normalize("Pokémon")` produced `"pok mon"` vs `normalize("Pokemon")` →
   `"pokemon"`, rejecting an objectively correct match. Fixed with Unicode NFD decomposition
   + combining-mark stripping *before* the ASCII filter, in both `TitleMatcher.java` and the
   `mesh.html` mirror (which had its own separate, older, unfixed copy of the same logic
   inside `queryJustWatch()` — now points at the shared function instead of duplicating it).
2. **Length floor ran before the exact-match check.** `"pokemon"` is 7 characters, under the
   8-char "too short/generic to trust a fuzzy match" floor — which incorrectly rejected it
   even as an EXACT match. An exact match (post-normalization) must always be trusted
   regardless of length ("Pokemon"/"Loki"/"Bluey" are all legitimately short real titles);
   the length floor should only ever gate the *fuzzy* (edit-distance) path. Fixed by
   reordering the exact-match check first in both copies.

### Show-enrichment moved server-side into tv-telemetry-agent (added 2026-09-04)

The JustWatch/TVMaze lookup (see below) moved from `mesh.html` into the Android app itself
for Mi TV content — `TitleMatcher.java` (the shared verification gate, pure logic, JUnit-
tested), `ShowEnrichmentFetcher.java` (JustWatch first, TVMaze fallback, both gated through
`TitleMatcher`), wired into `TelemetryCollector.maybeEnrichShow()` which fires once per
*actual content change* (tracked via `TelemetryState.getContentKey()`), not once per 3s poll
tick like the old browser-side version did. Runs on a background thread — `refresh()` is
called from `HttpServerService`'s polling `Handler` and must never block on network I/O.
`TelemetryState.updateEnrichment()` re-checks the content key before writing, so a slow
response for content that's already changed again can't overwrite the current state.

`entertainment-poller.uc` passes the agent's verified `poster_url` straight through into
`now_playing.poster_url`. `mesh.html`'s `resolveShowMetadata()` now short-circuits to use it
directly when present, skipping every external-lookup tier entirely for Mi TV content. Fire
TV (no companion agent) and the pre-agent ADB-dumpsys fallback path still run those tiers in
the browser — same `isPlausibleTitleMatch`/`normalizeTitle`/`levenshteinDistance` algorithm,
now applied to all 4 previously-unguarded TVMaze/JustWatch call sites, not just the one found
first.

**Known separate limitation surfaced while verifying this live** (not fixed): the
accessibility scraper's adjacent-node heuristic (`ShowMetadataParser`) can swap which text
node it treats as the show name vs. the episode title — observed live: a Zig & Sharko episode
came through with `show: "Cold Snap"` (actually the episode title) instead of `"Zig & Sharko"`.
When this happens, enrichment correctly finds no confident match for the wrong query and
leaves `poster_url` empty — that's the safety design working as intended (missing beats
wrong), but it does mean some content won't get enriched due to this upstream heuristic
issue, not a matching-algorithm failure.

### JustWatch poster-resolution tier (added 2026-09-04)

`mesh.html`'s `resolveShowMetadata()` now queries JustWatch's public GraphQL API
(`apis.justwatch.com/graphql`, `country: "IN"`) directly from the browser — CORS is open
(verified: reflects any `Origin`). This specifically covers Indian OTT catalog content
(JioHotstar, Netflix, Prime Video, etc.) that TVMaze/iTunes (Western-catalog-centric) have
no record of, which is why the hardcoded `KNOWN_EP_MAP` episode→show lookup existed in the
first place — JustWatch replaces the need to keep growing that map by hand.
- **TIER A (episodes)**: tried first using whatever real show name the tv-telemetry-agent's
  accessibility scraper already detected (`extraMeta.artist`/`album`, populated via
  `entertainment-poller.uc`'s hybrid merge — see §14), ahead of `KNOWN_EP_MAP` and the iTunes
  tvEpisode fallback.
- **TIER B (movies/standalone)**: tried first using the raw title, ahead of TVMaze.
- Poster URLs are templates (`/poster/<id>/{profile}/<slug>.{format}`) — must be prefixed
  with `https://images.justwatch.com` and have `{profile}`/`{format}` substituted (`s332`/`webp`
  verified working: real 200, real webp image).
- Regression coverage: `regression-test.uc` §7 smoke-tests the JustWatch endpoint itself
  (schema/reachability only — there's no server-side path to test the frontend JS directly).

### ⚠️ Rule: tests are mandatory for poller/app changes

**Any change to `entertainment-poller.uc`, `poller-lib.uc`, `mitv-api.uc`, `firetv-api.uc`,
or anything under `tv-telemetry-agent/` must be followed by running the full regression
suite below (`/test` skill) before considering the change done — not just a syntax check
or a single manual curl.** This is a repeat-offender codebase: the same silent-failure shape
(adb shell length limit, a missing field, a stale mapping) has recurred multiple times across
sessions, always caught only by actually running things, never by reading the diff.

**If the change adds or modifies capability that isn't covered by an existing test, add a
test case for it in the same change** — don't ship new poller/app logic with zero coverage
and rely on a future session to notice it's untested. See `.claude/skills/test/SKILL.md` for
which of the three layers (pure-logic unit test, live smoke test, JUnit) fits which kind of
change, and the deploy-before-test caveat (testing the router's stale prior copy of a file
gives a false pass).

### Regression test suite (added 2026-09-04)

Three independent layers, each catching a different class of bug — all were verified to
genuinely fail when the corresponding bug is deliberately reintroduced (not rubber-stamped):

1. **`test-poller-lib.uc`** — pure-logic unit tests for `poller-lib.uc` (the shared library
   extracted from `entertainment-poller.uc`: `parseMediaSession`, `resolveTvAppName`,
   `resolveFtvAppName`, `resolveSoundProfileLabel`, `resolveOutputType`). No device/network
   needed — 33 assertions covering the exact bugs found this session (SmartTube titles with
   embedded commas, 3-field music-app descriptions, the missing `smarttube` app-name mapping,
   Night-vs-Voice DRC distinction, stale/fully-null sessions). Run on the router (ucode isn't
   installed locally on macOS):
   ```bash
   ssh root@192.168.0.12 'cd /usr/bin && ucode test-poller-lib.uc'
   ```
2. **`regression-test.uc`** — live smoke test against the actually-running system: process
   health, `entertainment-status.json` schema/range validation, cross-checks the poller's
   *reported* values against ground truth read directly off the device (the exact bug class
   behind the `audio_output_type`/`arc_active` mixup), `mitv-api` endpoint checks, a real
   command-injection payload sent to `open_url` (must NOT execute on the TV), a punctuation
   round-trip through `urlDecode()`, and an explicit `am start-foreground-service` sent to
   `tv-telemetry-agent` (must survive — this is the literal ANR this session fixed). Gates
   every TV-dependent check on a live ping first, so a genuinely offline TV produces `SKIP`,
   not a false failure.
   §3b/§3c (added 2026-09-09) are **source-level** assertions rather than live probes: §3b
   checks the `sports`/`cinema` scene handlers actually write `tv_picture_advance_video_mjc_effect`
   (invoking the scenes for real would switch HDMI input and wake the TV), and §3c checks the
   poller's keep-alive string contains no `&;`. Both catch "the code claims it but never does
   it" bugs that no end-state probe can see. Note §3b was first written with the regex
   `/param == "sports"[\s\S]*?print\(sprintf/`, which **ucode's regex engine returns `null`
   for** — a third instance of this codebase's recurring ucode-regex gotcha; rewritten with
   plain `index()`/`substr()`. Run on demand after any deploy:
   ```bash
   ssh root@192.168.0.12 ucode /usr/bin/regression-test.uc
   ```
3. **JVM unit tests for `tv-telemetry-agent`** (`ShowMetadataParserTest`, `TelemetryStateTest`,
   `TelemetryCollectorTest`) — `ShowMetadataParser` was extracted from `TvAccessibilityService`
   specifically to make the episode/show-title detection heuristic (sanity bounds, UI-chrome
   word filtering, synopsis proximity) testable without Robolectric or an emulator. Runs on
   the plain JVM:
   ```bash
   cd tv-telemetry-agent && JAVA_HOME=/opt/homebrew/opt/openjdk@17 ./gradlew test
   ```

**Deployed copies live at** `/usr/bin/poller-lib.uc`, `/usr/bin/test-poller-lib.uc`,
`/usr/bin/regression-test.uc` on the router — keep these in sync with the repo copies on
every deploy of `entertainment-poller.uc`.

### Mi TV Firmware & Android 11 Upgrade (Verified Live 2026-09-14)
Confirmed via live ADB inspection that the Mi TV (`ro.product.device=croods` / `ro.product.model=MiTV-MOOQ0`) has been successfully upgraded to **Android 11** on the stable build **8835**:
- **Display ID**: `RTM4.220307.164.8835 release-keys`
- **Android Version**: `11` (API level / SDK `30`, `ro.build.version.release=11`)
- **Incremental Build**: `8835` (fixes the historical HDMI-disconnect regression present in build 8313)
- **Security Patch**: `2025-06-01`
- **Kernel**: Linux `4.19.195+-ab8835 #1 SMP PREEMPT Thu Jul 17 20:17:30 CST 2025 armv7l`
- **Boot Region**: `ro.boot.region=in` (retains Indian locale and regulatory channels)
- **DRM & Widevine**: Intact and operational (Widevine HAL 1.3, Netflix DRM HAL 1.3, ClearKey HAL 1.3 verified active)
- **Settings & MediaTek HAL**: Global MediaTek display controls (`tv_picture_*`, `picture_*`, `sound_output_switch=7`, `mjc`, local dimming) remain fully functional and integrated with `/www/cgi-bin/mitv-api.uc`.
- **Companion Telemetry**: `tv-telemetry-agent` (`com.home.tvagent`) re-installed and granted secure accessibility & notification listener permissions (`settings put secure enabled_accessibility_services com.home.tvagent/.TvAccessibilityService`, `cmd notification allow_listener com.home.tvagent/.TvMediaSessionService`). Local HTTP daemon on port 8080 is verified serving real-time telemetry to `entertainment-poller.uc`.

### Hardware identity facts (confirmed live 2026-09-08/09)
- **SoC**: MediaTek MT9611 (quad Cortex-A55 @1.5GHz, dual-core Mali-G52, 2GB LPDDR3) — this is genuinely the Mi TV Q1 75 / Mi QLED TV 75 chipset; MStar-lineage boot stack visible in `getprop` (`init.svc.mstar-light-hal-2-0`, `ro.boot.boot_devices=mstar_mci.0`, `ro.board.platform=m7632`).
- **Panel**: 75" CSOT-vendor LED (`ro.boot.mi.panel_vendor=CSOT`, `ro.boot.mi.panel_size=75`), 192-zone FALD, 120Hz — but this Q1/QLED-75 model is well-documented (VideoCardz, XDA) as **unable to accept a 4K@120Hz input signal**; the HDMI 2.1 ports only support 4K60 input, upscaled to 120Hz via MJC/MEMC frame interpolation. There is no EDID/scaler bug to "fix" via CRU — the panel genuinely cannot take 120Hz input, full stop.
- **Wi-Fi/Bluetooth combo chip**: MediaTek MT7668, attached internally via USB bridge (`mt7668_usb`, visible in Wi-Fi HAL logs). Wi-Fi: 802.11a/b/g/n/ac (Wi-Fi 5), dual-band, 1×1 spatial stream, VHT80 — confirmed live via an 866Mbps/5745MHz association to Nokia's 5GHz radio (VHT-MCS9 ceiling for a 1-stream 80MHz client, matches spec exactly). No Wi-Fi 6/6E. Bluetooth: 5.0 dual-mode (BR/EDR + LE), `MaxConnectedAudioDevices=2`, `A2dpOffloadEnabled=false` (software A2DP decode on host).
- **The TV correctly associates to the nearest AP on its own.** Its BSSID (`48:f8:db:6d:98:74`, Huawei OUI) sits on channel 149 — Huawei AP #1's fixed 5GHz channel — even though the shared mesh SSID (`U101`, broadcast identically by all four radios for 802.11k roaming) makes it look at a glance like it might be on Nokia. Channel/BSSID, not SSID, is the only way to tell which physical AP a client is actually on in this mesh.

---

[← Back to CLAUDE.md index](../CLAUDE.md)
