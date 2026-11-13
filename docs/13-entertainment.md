## 🎬 13. Entertainment System Engineering (2026-09-03)

Full home theater stack has been instrumented, optimized, and automated. All changes verified live on hardware.

### Hardware Topology

```
Xiaomi Mi QLED 75" 4K (192.168.0.10:5555)
  └── HDMI 1 ← Amazon Fire TV Stick 4K (192.168.0.7:5555)
  └── HDMI 2 eARC → Sony HT-S40R 5.1ch 600W (CEC 2.0.0.0)
```

**HDMI-CEC Logical Addresses:**
- Mi TV = Root `0.0.0.0` (CEC bus master)
- Fire TV Stick 4K = `1.0.0.0` (HDMI 1)
- Sony HT-S40R = `2.0.0.0` (HDMI 2 eARC)

### Fire TV Stick 4K — Device Facts

| Property | Value |
| :--- | :--- |
| Model | `AFTMM` (Fire TV Stick 4K, 1st Gen) |
| Fire OS | `6.0` (Android 7.1.2 kernel — **maximum for this hardware**) |
| Security Patch | November 2025 (current) |
| Last OTA | March 21, 2026 |
| ADB Address | `192.168.0.7:5555` (always online — USB-powered) |
| Widevine DRM | L1 (`drm.service.enabled=true`) |
| WebView | Chromium v118 (sideloaded `com.amazon.webview.chromium`) |
| HDMI Audio | `hdmi_audio_out_format=1` = Dolby Digital Plus E-AC3 passthrough |
| HDMI Resolution | `hdmi_resolution=3` = 4K UHD 3840×2160@60fps locked |

### Fire TV ADB Unlocks Applied (2026-09-03)

All applied via `adb -s 192.168.0.7:5555 shell settings put ...`:

| Setting | Key | Value | Effect |
| :--- | :--- | :--- | :--- |
| UI Animation Speed | `global window_animation_scale` | `0.5` | 2× faster menu transitions |
| UI Transition Speed | `global transition_animation_scale` | `0.5` | 2× faster screen transitions |
| Animator Duration | `global animator_duration_scale` | `0.5` | 2× faster overall animations |
| Ad Tracking | `secure limit_ad_tracking` | `1` | Amazon ad ID limited (privacy) |
| Dolby Volume Leveler | `secure volume_leveler_enabled` | `1` | Auto DRC — levels loud/quiet |
| Bass Enhancer | `secure bass_boost_enabled` | `1` | Boosts subwoofer low-freq signal |
| Dialogue Enhancer | `secure dialogue_enhancer_enabled` | `1` | Lifts center-channel voice |
| HDMI Audio Format | `global hdmi_audio_out_format` | `1` | Force E-AC3 Dolby Digital Plus |
| HDMI Resolution | `global hdmi_resolution` | `3` | Lock 4K UHD@60fps |
| Background Process Limit | `global background_process_limit` | `4` | Cap RAM-hungry background apps |
| A/V Sync Offset | `system av_sync_offset` | `30` | Compensate 30ms ARC latency |
| Alexa Always-On Mic | `secure alexa_hands_free_enabled` | `0` | Mic only active on remote press |

### Dolby Audio: Passthrough vs Decode

`hdmi_audio_out_format=1` means **full bitstream passthrough** — the Fire TV CPU does NOT decode audio. The raw E-AC3 compressed stream is forwarded over HDMI to the Mi TV, then over eARC to the Sony S-Master amp, which decodes it into 6 discrete channels. This is confirmed by the Audio Flinger policy dump:
```
Force use for encoded surround output = 14  (always passthrough, never PCM decode)
Available HDMI format: AUDIO_FORMAT_E_AC3 @ 48000 Hz, 5.1 channel mask 0x3f0000
```

**Sony HT-S40R Dolby decode capability:**
- ✅ Dolby Digital 5.1 (AC3) — native
- ✅ Dolby Digital Plus (E-AC3) — via ARC
- ❌ Dolby TrueHD — requires Atmos chip
- ❌ Dolby Atmos object-based — hardware not present
- ❌ **DTS / DTS-HD — not supported at all** (user-confirmed 2026-10-18). This is what makes
  the encoded-surround policy below matter: a DTS bitstream force-passed to this soundbar is
  silence, not a downmix.

### CORRECTION + encoded-surround policy: the link is plain ARC, MANUAL mode is a trap (2026-10-18)

**The `Force use for encoded surround output = 14` line quoted above is stale.** Live read of
`dumpsys media.audio_policy` gives **`0`** (unforced / AUTO), not `14`. The distinction is not
cosmetic: `14` = `FORCE_ENCODED_SURROUND_ALWAYS` would force-pass *every* bitstream including
DTS, which this Dolby-only soundbar cannot decode. `0` (AUTO) passes through only what the
sink's EDID advertises and decodes everything else on the TV — the correct behaviour here, and
already in effect. The eARC reference in the paragraph above is also wrong, see below.

**The TV<->Sony link is plain ARC, not eARC** — `dumpsys hdmi_control` reports
`mArcEstablished: true` / `mEarcConnected: false`. Plain ARC is an S/PDIF-grade pipe: it
carries a compressed 5.1 bitstream *or* 2.0 LPCM, never multichannel PCM. Combined with the
fact that the TV has **decoders only and no encoder of any kind**
(`/vendor/etc/media_codecs.xml` lists `OMX.MS.AC3.Decoder`, `OMX.MS.EAC3.Decoder`,
`OMX.MS.DTS.Decoder`; there is no Dolby MS12, no `libdolby*`, no AC-3 encoder anywhere in
`/vendor/lib` or `/vendor/lib64`), this means **anything the TV decodes itself reaches the Sony
as stereo.** There is no decode-then-re-encode path that would preserve 5.1. Consequence worth
internalising: **on this link, TV-side DSP and discrete 5.1 are mutually exclusive** — the EQ,
DTS Virtual:X and Dolby DAP post-processors only act on the PCM path, which is exactly the path
that costs the surround channels.

**`encoded_surround_output = 3` (MANUAL) is silently clamped to ALWAYS on this firmware — do
not use it.** Verified by writing each value and reading the effective policy back:

| `Settings.Global.encoded_surround_output` | Effective `Force use for encoded surround output` | |
|---|---|---|
| `0` AUTO | `0` (unforced) | OK — honoured; **current, correct setting** |
| `1` NEVER | `13` `FORCE_ENCODED_SURROUND_NEVER` | OK — honoured |
| `2` ALWAYS | `14` `FORCE_ENCODED_SURROUND_ALWAYS` | OK — honoured |
| `3` MANUAL | **`14`, not `15`** | **clamped — does the opposite of what it says** |

Requesting MANUAL therefore force-passes every format, DTS included. The corroborating symptom
is logcat spamming `APM_AudioPolicyManager: setSurroundFormatEnabled() not in manual mode for
surround sound format selection` + `AudioSystem::setSurroundFormatEnabled error -38` — the
per-format allowlist in `encoded_surround_output_enabled_formats` is rejected wholesale, so that
key is inert on this build and must not be written.

**Lesson (same shape as §14's recurring silent-failure class)**: `settings put` succeeding and
`settings get` reading the value back is NOT evidence the framework honoured it. A per-format
surround allowlist was written, read back as exactly `5,6,18`, and looked completely correct —
only `dumpsys media.audio_policy` revealed the policy never left ALWAYS. For any
`Settings.Global` key that feeds the audio/display policy, verify against the *policy's* own
dump, not the settings store.

> 📘 **Practical operations guide: [`AUDIO.md`](../AUDIO.md)** — how to drive the audio stack from
> the dashboard, how to read the live format HUD, measured format support per output, and
> troubleshooting. The sections below are the engineering evidence behind it.

### Verified DSP capability map — which Settings.Global audio keys are REAL (2026-10-18)

§13's DSP catalog below was assembled from decompiled `MiTVSettings.apk` strings, which lists
every key the *app* knows about — not every key this *firmware* honours. Each candidate was
toggled live and checked for `TVSettingConfig: setConfigValue is configId ...` in logcat.

**Use the Android-side marker, not the kernel one.** `logcat -c` does NOT clear the kernel
ring buffer, so stale `[SGLUE_AUD]` lines from an earlier toggle reappear in the next read and
read as a false positive — this produced a confident but wrong "DBX is real" result mid-session
before the method was corrected.

| Settings.Global key | MediaTek config ID | Status |
|---|---|---|
| `sound_dts_virtual_x` | `g_audio__dts_virtual_x` | ✅ REAL |
| `dts_virtual_surround` | `g_audio__dts_surround` | ✅ REAL |
| `dts_dialog_clarity` | `g_audio__dts_limiter` | ✅ REAL — **name and register disagree** |
| `dts_bass_enhancement` | `g_audio__dts_tbhdx` | ✅ REAL (TruBass HDX) |
| `sound_dts_drc` | `g_fusion_sound__dts_drc` | ✅ REAL |
| `sound_dts_tbhdx`, `sound_dts_limiter` | — | ❌ INERT — §13 documented these names, but the live aliases are the two `dts_*` keys above |
| all `sound_dbx_*` | — | ❌ INERT |
| `sound_sonic_emotion_premium` | — | ❌ INERT |
| all `sound_dts_studio_*` | — | ❌ INERT |
| `sound_downmixmode` | — | ❌ INERT |

Inert keys were deleted from the settings store after probing; don't write them.

### `sound_output_switch` is a single-select enum — simultaneous TV + Sony output is not possible

Decompiled from `SoundDeviceManager.java`:
`0=BLUETOOTH 1=ALL_SCO 2=USB_SOUND_CARD 3=HEADSET 4=HEADPHONE 5=HDMI_ARC 6=SPDIF 7=SPEAKER 8=AUX_LINE`

One integer, one sink; `getSoundDevice()` resolves the device bitmask through a priority
if/else chain that returns exactly one index. There is no combined value, `sound_speakers`
(a separate key the settings app defines) reads `null` and is unused, HDMI System Audio mode is
actively enforced (`Force use for HDMI system audio: 12` = `FORCE_HDMI_SYSTEM_AUDIO_ENFORCED`,
which mutes the internal speakers by design), and AudioFlinger has a single output thread
(`type 0 (MIXER)`) with no duplicating output. Instantiating one would mean editing
`/vendor/etc/audio_policy_configuration.xml` on a read-only vendor partition, i.e. root.
Even if forced, the ARC path needs ~70 ms of compensation the speaker path does not, so
unsynchronised dual output would comb-filter.

**Bug fixed (2026-10-18)**: `mitv-api.uc`'s `audio_output` handler mapped `spdif`/`optical` to
`1` — `ALL_SCO` (Bluetooth SCO), the wrong sink entirely. Correct value is `6`. `arc`→`5` and
`speaker`→`7` were already right. No end-state probe catches this class of bug, since nothing
reads back "is this the sink the user asked for"; covered now by a source-level assertion in
`regression-test.uc` §3e.

### `mitv-api?cmd=audio_profile` — one coherent profile per output path (2026-10-18)

The two sinks want genuinely different configurations, and settings that matter for one are
inert or misleading for the other, so they are applied as whole profiles rather than left to be
combined by hand:

| | `param=sony` | `param=tv` |
|---|---|---|
| Sink | `sound_output_switch 5` (HDMI-ARC) | `sound_output_switch 7` (internal speakers) |
| Surround | `encoded_surround_output 0` (AUTO) → Dolby bitstream out, DTS decoded locally | n/a — stereo sink |
| DTS Virtual:X suite | **off** (bypassed on a passthrough bitstream) | **on** — Virtual:X + surround + dialog clarity + TruBass + DTS DRC |
| Dolby | `sound_style 1` Theater, `dolby_sound 2`, DAP on | `sound_style 0`, DAP on, dialogue enhancer 2 |
| EQ | all five bands reset to 0 | small-driver voicing: `0 / +2 / +1 / +3 / +2` |
| Lip-sync | `sound_spdif_delay 70` (the §13 calibration — found at 0) | `0` (no ARC in path) |

`param=status` reports `active_profile`, which is **`custom` whenever the sink and the DSP
state disagree** (e.g. routed to ARC with Virtual:X still on from the speaker profile) rather
than guessing from the sink alone. UI: the **Best Audio Profile** card at the top of the Mi TV
tab's Audio Output subcard in `mesh.html`, with a live badge and per-profile explanation.

Both profiles write all five EQ bands explicitly, because they are otherwise sticky across
profile switches — a Dialogue-Clarity curve (`2/-2/2/4/3`) was found still applied while the
active profile was Theater, the same leftover-state bug class as the Night/Voice DRC mixup in
§14 bug 6.

Covered by `regression-test.uc` §3e (profile coherence, Virtual:X-vs-sink consistency, non-zero
ARC lip-sync delay, and the optical enum). Both new assertions were verified to fail when
deliberately broken.

### `mitv-api?cmd=audio_format` — what is actually being decoded, and by which box (2026-10-18)

Neither the file nor the app can answer this; only the audio HAL knows. Source of truth is
`dumpsys media.audio_flinger`'s output threads, parsed by `parseActiveAudioFormat()` in
**`poller-lib.uc`** (put there so it is unit-testable — see `test-poller-lib.uc`).

**The trap this parser exists to avoid**: closed/stale threads are printed with a LEADING
DASH (`- Output thread …`) while live ones start at column 0, and the dump is full of idle
placeholder OFFLOAD/DIRECT threads still advertising `AUDIO_FORMAT_AC3` at 8000 Hz on a single
front-left channel. A parser that ignores the dash cheerfully reports "Dolby Digital
passthrough" while the TV sits on a screensaver playing silence. Both fixtures (real idle
capture, synthetic live-E-AC3) are in the unit tests, and the test was confirmed to fail when
the dash guard is removed.

A compressed HAL format on a live thread means the TV is **not** decoding — the bitstream is
going out and the Sony decodes it. PCM means the TV decoded it, and on plain ARC whatever
reaches the soundbar is stereo.

**Sink-awareness is essential and was initially missing.** "Compressed format ⇒ passthrough"
is WRONG on this TV: the Speaker device port declares its own `AUDIO_FORMAT_AC3` / `E_AC3` /
`E_AC3_JOC` profiles, so an identical compressed format means *the Sony is decoding* on the ARC
sink and *the TV's own HAL is decoding* on the speaker sink. The first cut ignored the sink and
announced "Sony HT-S40R decodes" while playing to the internal speakers. The derivation now
lives in `poller-lib.uc` as `describeAudioPath(fmt, sinkCode)` with all four
sink × format combinations pinned in `test-poller-lib.uc`; the tests were confirmed to fail
when the naive rule is restored. `mesh.html` keys its rendering off `passthrough`, not
`compressed`, for the same reason.

**Never render a channel count for a passthrough stream.** Confirmed during real Apple TV+
Dolby playback: the live E-AC3 OFFLOAD thread reports channel mask `0x3`
(front-left, front-right) — the IEC61937 *carrier* framing, not the 5.1 audio inside it. The
first HUD printed it, producing `DOLBY DIGITAL PLUS (Bitstream -> Sony decodes | front-left,
front-right)` next to a genuine 5.1 bitstream, which reads as stereo and is exactly backwards
from what the sony profile delivers. The payload now carries `channels_are_carrier` so
consumers can suppress it; the real capture is pinned as a fixture in `test-poller-lib.uc`.

`resolveSourceFormat(mime)` recovers the DECODER-side identity from `dumpsys media.metrics`,
which is the only place a Dolby origin survives when the platform decodes to PCM before the
mixer sees it — this is what lets the speaker path still report "Dolby Digital Plus → out: PCM".
`audio/raw` means the app decoded in its own process, so the source codec is genuinely
unobservable; it is reported as `app-decoded PCM` rather than implying native PCM content.

- `cmd=audio_format` → `{format, format_label, compressed, passthrough, decoded_by,
  source_format_label, path_summary, sample_rate, channels, thread_type, source_mime,
  source_codec, sink}`
- `cmd=audio_format&param=hud` → same, plus a **2-second on-screen HUD** on the TV via
  `am broadcast -a com.home.tvagent.SHOW_MODE`. The 2s fade is the companion agent's own
  `VolumeOverlayManager.DISMISS_DELAY_MS = 2000`; nothing new was needed for the timing.
- UI: a live format line plus a **🔍 Show on TV** button in the Best Audio Profile card.
  The dashboard poll refreshes the line on a 12s client-side throttle, because this probe
  runs `dumpsys` on the TV and is much heavier than the profile status read.

Covered by `regression-test.uc` §3f. The HUD branch is asserted at source level rather than
fired, so running the suite does not flash an overlay on the TV — same reasoning as §3b, and
the same lesson as the regression test that used to open a browser on the TV every run.

**Also fixed in `poller-lib.uc`: `resolveOutputType()` named the wrong sinks.** It mapped
code `0` to "TV Internal Speakers" (0 is Bluetooth; the speakers are `7`) and code `1` to
"Optical / S-PDIF" (1 is Bluetooth SCO; S-PDIF is `6`). This feeds the dashboard's
`audio_output_type`, so those codes displayed a sink the TV was not using. The two
`test-poller-lib.uc` cases covering them had encoded the same wrong mapping and were corrected
alongside.

### 🚨 BUG CLASS: "preset bleed" — PQ writes land in whatever picture mode is on screen (fixed 2026-10-18)

**Read this before touching any `picture_*` write path.** It has already produced two
user-visible faults and will recur in any new handler that writes PQ.

**The rule**: every `picture_*` register on this TV is stored **per picture mode**. A write
lands in whichever mode is selected at that instant. `picture_mode` and `picture_mode_dolby`
track each other, and a `picture_mode` write has been observed **not to take on the first
attempt**, so pinning a mode must be *verified*, never assumed.

**What it looked like in the wild** — both reported as Dolby Vision faults, neither actually
about Dolby Vision:
- *"DV Bright shows black & white"* → `picture_saturation=0` in that mode alone (DV Dark was 60).
- *"DV Dark is brighter than DV Bright"* → DV Bright held the **cinema** preset verbatim
  (`b50 c50 sharp1 bl45 cs5`) and DV Dark held the **sports** preset (`b55 c65 sat60 sharp5 bl85`).

Three defects combined to cause it, all now fixed:

| # | Where | Defect | Fix |
|---|---|---|---|
| 1 | `mitv-api.uc` `scene` handlers | Wrote `picture_backlight`/`saturation`/`hue`/`color_temperature` and **never set `picture_mode` at all** — guaranteed to land in the active mode | Pin the scene's mode first (`cinema`→9, `sports`→2, `night`→9, `music`→7); PQ writes are gated on `scenePqOk` and skipped if the pin fails, with `pq_applied` in the response |
| 2 | `mitv-api.uc` `display_preset` | Wrote `picture_mode` in the **same adb command** as the PQ values, giving the panel no chance to switch, and never verified | `ensurePictureMode()` pins and re-reads, retrying once; refuses the preset outright if the mode will not move |
| 3 | `mesh.html` `applyMasterPreset()` | Called the TV's Rust daemon **directly from the browser** (`:9988/preset`), bypassing the router guard entirely — the UI half of the bug | Routes through `mitv-api?cmd=display_preset`; `mitv-api` still uses the Rust path internally *after* pinning, so almost no speed is lost |

**Helpers** (`mitv-api.uc`): `currentPictureMode(ip)`, `ensurePictureMode(ip, target)` →
verified pin, `isDolbyVisionMode(mode)` (5, 6, 13).

**When adding any new PQ-writing handler**, call `ensurePictureMode()` first and bail if it
returns false. Do not write `picture_mode` inline with the values.

**UI**: the Display & Sound panel now shows `#mitv-active-mode` — which mode the sliders are
editing — and turns amber with an explicit warning on a Dolby Vision mode, because that is
where an accidental write is both easiest to make and hardest to undo from the stock menus.

**Verified live**: with the panel sitting in DV Dark, applying the `cinema` preset switched to
mode 9 and wrote there; DV Dark came back intact at `bl 45, sharp 1, sat 60`. Before the fix
those values landed in DV Dark itself.

Covered by `regression-test.uc` §3i (source-level — invoking presets for real would rewrite
the user's calibration, which is the damage under test). Confirmed to fail when the pin is
removed.

### Dolby Vision Bright showed black & white — PQ values are stored PER PICTURE MODE (2026-10-18)

User-reported: Dolby Vision **Bright** displayed in black and white while Dolby Vision
**Dark** was fine. Cause was not Dolby Vision at all:

```
picture_mode_dolby=6 (DV Dark)   -> picture_saturation=60   correct
picture_mode_dolby=5 (DV Bright) -> picture_saturation=0    black & white
```

**Every `picture_*` register is stored per picture mode.** Reading `picture_saturation` tells
you the value for the *currently selected* mode only, and writing it only affects that mode.
So a bad value can sit undetected in one mode indefinitely while every other mode looks
perfect — and a `display_state` dump taken in a healthy mode shows nothing wrong. Verified by
switching 5→6→5 and watching the value follow the mode.

Fixed by selecting DV Bright and writing `picture_saturation 60` to match DV Dark.

**Still anomalous, left for the user to decide**: DV Bright has `picture_backlight=45` while
DV Dark has `85` — "Bright" is dimmer than "Dark". 45 is the value the `cinema` preset and
scene both write, so that mode most likely had a cinema preset applied while it was active.
Backlight is a room-preference setting rather than a correctness one, so it was not changed.

**Follow-up: "Dark" was also BRIGHTER than "Bright", and for the same reason.** The names
describe the **room**, not the picture — *Dolby Vision Bright* is the bright-room preset and
*Dolby Vision Dark* the dark-room one, so Bright should carry the higher backlight. Both modes
turned out to be carrying our own presets verbatim:

| | live values | matches |
|---|---|---|
| DV Bright (5) | `b50 c50 sharp1 bl45 cs5` | the **cinema** preset (`b:50 c:50 sharp:1 bl:45 cs:5`) |
| DV Dark (6) | `b55 c65 sat60 sharp5 bl85` | the **sports** preset (`b:55 c:65 sat:60 sharp:5 bl:85`) |

So a `cinema` preset had been applied while DV Bright was selected and a `sports` preset while
DV Dark was — each writing its PQ values into whichever Dolby Vision mode happened to be
active. Hence Dark (sports, backlight 85) outshining Bright (cinema, backlight 45), and the
zeroed saturation arriving by the same route.

⚠️ **`display_preset` and `scene` write PQ into the CURRENTLY SELECTED mode.** They do set
`picture_mode` first, but the values still end up wherever the panel is actually sitting — and
on this TV `picture_mode` and `picture_mode_dolby` track each other, so applying any preset
while a Dolby Vision mode is on screen silently overwrites that DV mode's stored calibration.
(Note DV does *not* lock the mode: writing `picture_mode 9` during DV playback sticks, so this
is preset bleed, not a DV restriction.) If you apply presets while watching DV content, expect
to re-check the DV modes afterwards.

Corrected on the device, leaving DV Dark as the user had been watching it:
`DV Bright → bl 95, c 55, sharp 1, sat 60`. DV Dark still carries the sports fingerprint
(`sharp 5`, `c 65`) which is over-sharpened for film — not changed, since it is what was in
active use.

**Latent bug fixed at the same time**: the generic numeric path in `mitv-api.uc` did
`v = int(param)`, and ucode's `int()` returns NaN for a non-numeric or empty string while
`sprintf("%d", NaN)` renders `"0"` — so a malformed request silently wrote **zero** to a
picture register. The bounds clamp underneath cannot catch it, because 0 is a legal value for
every one of these fields. On `saturation` that is exactly a black-and-white picture. This is
the same NaN→`"0"` shape as §14 bug 2, which had only ever been fixed for the two enum fields
(`color_temp`, `local_dimming`). The path now rejects non-numeric input instead:
```
cmd=saturation&param=abc -> {"success":false,"message":"Non-numeric value 'abc' ... would have silently set 0"}
cmd=saturation&param=60  -> {"success":true,"value":60}
```
Covered by `regression-test.uc` §3h, which also asserts live `picture_saturation > 0`.

### ⚠️ OPERATIONAL: never `adb connect` to the Mi TV from a second host (2026-10-18)

Running `adb connect 192.168.0.10:5555` from the workstation, while the router already held
the authorised session, put the TV into `unauthorized` for **the router too**:

```
192.168.0.10:5555   unauthorized
```

The second host offers a different RSA key, the TV raises a fresh "Allow USB debugging?"
prompt, and until that prompt is answered the existing authorisation stops working. This takes
down the whole entertainment stack at once — `entertainment-poller` loses its only read path,
concludes the TV is idle, and the app-aware DSP switcher helpfully "resets" the sound profile
to Standard (visible in `/tmp/dsp-auto.log` as `Mi TV idle/standby → Profile reset to Standard`).

**Always drive the TV through the router's adb** (`ssh root@192.168.0.12 'adb -s 192.168.0.10:5555 …'`),
which is what every script in this project already does. If a file needs pulling to the
workstation, pull it to the router first and copy it from there.

**Recovery, in order of escalation:**
1. `ssh root@192.168.0.12 'adb kill-server; sleep 2; adb start-server; adb connect 192.168.0.10:5555'`
   — worked the first time. `adb disconnect`+reconnect alone is NOT enough; it reuses the
   stuck session. Only a server restart forces a fresh handshake, and the router's key is
   normally still in the TV's `adb_keys`.
2. **Kill the workstation's adb server too** (`adb kill-server` locally). The adb server is a
   background daemon that outlives the command which started it and keeps re-offering its key,
   so a single stray `adb connect`/`adb install` from a laptop can re-break the router
   repeatedly, long after the command finished. This is the step that is easy to miss.
3. If it is still `unauthorized`, the TV is waiting on an on-screen **"Allow USB debugging?"**
   prompt that only a human can accept — there is no programmatic route. Injecting ENTER via
   the Rust agent does not work; the dialog does not take injected focus.

**What stays up while adb is down**: the Rust agent (`:9988`) and Java agent (`:8080`) are
plain HTTP and keep serving, as do the mesh/AC/network stacks. Only the adb-dependent TV
telemetry degrades — and `entertainment-poller` will read the TV as idle and reset
`sound_style` to 0, so re-apply an audio profile after recovery. Injecting ENTER via the Rust agent does nothing; the dialog does not take injected
focus (and there may be no dialog at all). Re-apply an audio profile afterwards, since the
DSP idle-reset will have moved `sound_style` to 0.

### Validation of the three-path soundbar-control analysis (2026-10-18)

Each claim tested on the hardware rather than against AOSP source.

| Claim | Verdict | Evidence |
|---|---|---|
| TV is the HDMI-CEC bus master | ✅ TRUE | TV `logical_address 0x00` / `physical_address 0x0000`; Sony `0x05` / `0x2000` |
| `IHdmiControlService` exposes `sendVendorCommand` / `sendKeyEvent` / `setSystemAudioMute` | ✅ exist, ❌ unreachable | `android.permission.HDMI_CEC` is `prot=signature\|privileged\|vendorPrivileged`; `com.android.shell` does not hold it |
| Inspect the runtime via `app_process` | ❌ impossible here | `app_process / java.lang.String` is **Killed, exit 137** for the shell domain — fails before any class is named |
| **Path A** — volume / mute / power / audio mode over ADB | ✅ **TRUE, all four confirmed on the bus** | see below |
| **Path B** — IR for deep DSP | ✅ **TRUE on both halves** | (a) those params are not CEC-addressable: `MtkTvCecBase`'s entire API is standby/usercontrol/vendorcmd/timer/record, with no DRC, subwoofer or surround-trim method anywhere; (b) external hardware really is required — the TV **has no IR transmitter**: `android.hardware.consumerir` is absent from `pm list features`, `dumpsys consumer_ir` is empty, and there is no `consumerir` HAL in `/vendor/bin/hw` or `/vendor/lib*/hw`. The `consumer_ir` binder service in `service list` is the AOSP stub that exists regardless of hardware — do not mistake it for an emitter |
| **Path C** — `sound_digital_output_arc=7` = "Pass-Through" | ❌ **REFUTED** (definitively) | decoded from the TV's own resource tables, not just the constant names — see below |

**Path C, settled from the resource tables.** `apktool d -s MiTVSettings.apk` gives the
user-facing list and its values, which align positionally:

| `sound_spdif_type_entries` | `sound_spdif_type_entry_values` |
|---|---|
| Auto | **7** |
| PCM | 2 |
| Passthrough | **4** |

So `sound_digital_output_arc=7` is **Auto**. Passthrough is **4**. This matches the decompiled
constants exactly (`mDigitalOutputAuto_1 = 7`, `mDigitalOutputBypass = 4` — "Bypass" is the
internal name for the user-facing "Passthrough").

**Do not "fix" this to 4.** Auto is the correct setting here and is already doing the right
thing — verified live, Apple TV+ Dolby content reaches the Sony as a real E-AC3 bitstream on
an OFFLOAD thread. Forcing Passthrough would also force DTS out as a bitstream, which the
Dolby-only HT-S40R cannot decode — silence instead of the stereo downmix Auto produces.
Worth noting the ARC-specific menu (`sound_spdif_type_arc_entries`: PCM=2, Passthrough=4,
Dolby Digital=1) does not even offer Auto, so the live value is one that menu would never
show — it is set correctly regardless.

Path C's "pristine, uncompressed bitstreams" also conflates two things: a bitstream is by
definition compressed, and plain ARC cannot carry uncompressed multichannel at all.

**Path A is confirmed working, with evidence.** `input keyevent` from ADB reaches the soundbar
because the TV forwards it as standard CEC:
```
VOLUME_UP   -> [S] <User Control Pressed> 05:44 -> [R] <Report Audio Status> 50:7A:3A  (57->58)
VOLUME_DOWN -> ...                               -> [R] <Report Audio Status> 50:7A:39  (back to 57)
MUTE        -> [S] <User Control Pressed> 05:44 -> [R] <Report Audio Status> 50:7A:80  (mute bit)
```
**Audio mode: ✅ VALIDATED — but only via the right lever.** `sound_output_switch` is NOT it:
that is a local TV routing setting and switching it tv→sony→tv emits zero CEC frames
(`mSystemAudioActivated` stays `true` either way). The actual CEC control is
**`hdmi_system_audio_control_enabled`**, and toggling it over ADB drives the full handshake:
```
settings put global hdmi_system_audio_control_enabled 0
   [S] <System Audio Mode Request> 05:70        [R] <Set System Audio Mode> 5F:72:00   (Sony: OFF)
settings put global hdmi_system_audio_control_enabled 1
   [S] <System Audio Mode Request> 05:70:00:00  [R] <Set System Audio Mode> 5F:72:01   (Sony: ON)
```
Note the key does not appear in `settings list global` until first written — it defaults
unset, which is why an earlier pass concluded there was no lever. Toggling it briefly drops
the soundbar out of system audio, so audio blips to the TV for a second or two.

**Power: ✅ FULLY VALIDATED (both directions).** Engaging system
audio mode is what brings the soundbar into service and the Sony acknowledged it (`72:01`), so
the power-on direction is covered by the exchange above. **Power-state monitoring is fully validated**, with the decode: the TV polls
`<Give Device Power Status> 05:8F` once a minute and the Sony answers
`<Report Power Status> 50:90:00` — opcode `0x90`, parameter `0x00` = **On** (`0x01` Standby,
`0x02`/`0x03` in transition). 21 queries, 21 replies, no drops in a single window. So the
power channel is live, bidirectional, and already readable; only the *write* direction
(standby) is unexercised. `hdmi_control_auto_device_off_enabled=1` means the TV will
broadcast `<Standby>` when it sleeps. **The write direction is confirmed too**, tested with the user's explicit go-ahead:
```
(before)  grep -ci standby  ->  0
input keyevent KEYCODE_SLEEP
          [S] time=2026-10-18 23:35:04 message=<Standby> 0F:36
input keyevent KEYCODE_WAKEUP  ->  mWakefulness=Awake
```
`0F` = broadcast to all devices, `0x36` = `<Standby>`. So an ordinary ADB keyevent really does
put a power command on the CEC bus — Path A's power claim holds in full.

**Restore afterwards**: sleeping the TV pauses playback and the DSP idle-reset moves
`sound_style`, so re-apply an audio profile and resume. Verified back to `active_profile:
sony`, `mSystemAudioActivated: true`, `mArcEstablished: true`, Sony reporting `50:90:00` (On),
playback `state=3`.

### The app-domain CEC path: REACHABLE BUT INERT (2026-10-18)

Worth recording because it looks like a working bridge right up until the last check, and
because the two on-TV agents are **not** equivalent:

```
mitv-rust-agent   u:r:shell:s0          <- same domain as ADB; same restrictions, no help
com.home.tvagent  u:r:untrusted_app:s0  <- different domain, and it CAN load the vendor API
```

`/system/etc/permissions/mitvmiddlewareimpl.xml` declares `mitvmiddlewareimpl` as an
**app-usable `<library>`**, and its dex genuinely contains
`MtkTvCecBase$CecVndrCmdWithIdInfo` plus `setVendorCmdWithId` / `setUserCtrlPressed`, which
reach the CEC HAL through `TVNativeWrapper` JNI — bypassing system_server's permission gate
entirely. Adding `<uses-library>` to `tv-telemetry-agent` and probing confirmed:
`class_loaded: true`, `has_setVendorCmdWithId: true`, `instantiated: true`.

**But it does not transmit.** `setUserCtrlPressed()` / `setUserCtrlReleased()` both return
`rc=0` (the API's success code; non-zero throws `MtkTvExceptionBase`), and **three consecutive
transmits produced zero new `[S]` frames in `dumpsys hdmi_control`**, with the soundbar never
moving off its previous volume and never sending a `<Report Audio Status>` — while an ordinary
`input keyevent` minutes earlier produced both instantly. Android's HdmiControlService owns the
CEC line on this build; the MediaTek middleware path is vestigial.

Same failure shape as the clamped MANUAL surround mode and the inert DBX keys: **a call that
reports success and changes nothing.** The lesson from those applies again — never accept an
API's own return code as proof; verify against the bus/policy/hardware.

The reproducer is kept as `CecProbe.java` plus the agent's `/cec_probe` and `/cec_tx`
endpoints, so a future firmware can be re-tested in seconds instead of re-derived.

**Net: the conclusion below stands** — a USB-CEC adapter remains the only route to Sony vendor
frames, and IR the only route to DRC / subwoofer / surround trims.

### Dead end: raw Sony BRAVIA Sync CEC vendor frames cannot be injected from this setup (2026-10-18)

**The premise is correct and verified on the bus** — this is not a case of the research being
wrong, only of the injection path being unavailable here. From `dumpsys hdmi_control`:

```
CEC: logical_address: 0x05 device_type: 5 vendor_id: 524358 display_name: HT-S40R
[R] message=<Vendor Command With Id> 5F:A0 <Redacted len=6>
[R] message=<Device Vendor Id> 5F:87:08:00:46
```

`524358` = `0x080046`, the Sony OUI, and opcode `0xA0` (`<Vendor Command With Id>`) traffic is
genuinely flowing. So `tx 05 a0 08 00 46 …` is a real frame the HT-S40R would understand.

**Four injection routes, all closed on this device:**

| Route | Result |
|---|---|
| `cec-client` / libCEC | Not an Android tool; not present, and no `/dev/cec*` Linux-CEC-framework node exists at all |
| `cmd hdmi_control …` | `No shell command implementation` on this Android 11 build — the shell command that exposes `vendorcommand` on newer AOSP isn't here |
| `/dev/mik!cec` (MediaTek/MStar raw node) | **Exists, mode `crw-rw-rw-`** — but SELinux is `Enforcing` and the node is `u:object_r:mstar_mik_cec_device:s0`, so the `shell` domain gets `Permission denied` on open. File permissions are a red herring |
| Binder `hdmi_control` → `sendVendorCommand()` | Gated behind the `HDMI_CEC` permission (`signature|privileged`). `tv-telemetry-agent` is a normal app and cannot hold it without being installed to `/system`, which needs root |

Shizuku would not help even if reinstalled: it grants **shell**-level privileges, and shell is
exactly the domain SELinux is denying here.

**What would actually work**, if vendor-frame control of the soundbar is ever wanted:
a USB-CEC adapter (Pulse-Eight or equivalent) on the TP-Link or a Pi, which gives a real
`/dev/ttyACM*` + `cec-client` and full raw frame injection including `0xA0` / `0x8A`. The same
box would cover the settings the research correctly identifies as unreachable over CEC at all
(DRC, subwoofer level, individual surround trims), via an IR blaster speaking Sony SIRC —
note `/dev/mik!ir` exists on the TV but is SELinux-blocked the same way, and a TV's IR hardware
is a receiver, not a transmitter.

**Already available without any of this**: standard CEC volume and power reach the Sony
through the TV's System Audio path (what the dashboard's volume control already uses), plus
`mitv-api?cmd=resync_arc`. Only the *proprietary* sound-field/theater-mode frames are out of
reach.

⚠️ Before experimenting further here, re-read §14's CEC/ARC episode: heavy live CEC toggling
degraded the TV's in-memory CEC state badly enough to need a full reboot, and produced a
confident but wrong "Sony firmware bug" conclusion.

### Shizuku is no longer installed on the Mi TV (2026-10-18)

`regression-test.uc` §2b was failing on `shizuku_server is running`. The reviver in
`entertainment-poller.uc` (§15.7) is **not** at fault: `pm path moe.shizuku.privileged.api`
returns nothing and `pm list packages` has no shizuku entry — the package is gone, most likely
wiped by the Android 11 / build 8835 upgrade (§14), which reinstalled `tv-telemetry-agent` but
not this. A bare "is the process up?" assertion reported that as a keep-alive regression and
sent a session chasing a bug that did not exist. §2b now probes `pm path` first and reports
SKIP (package absent, nothing to revive) versus FAIL (installed but dead — the real
regression). Reinstall the APK if the Shield remote remap is wanted back.

**`mitv-api?cmd=surround_mode`** exposes this: `param=status` reports the requested mode and the
effective policy side by side with a `mode_clamped` flag (they disagree for value 3);
`param=auto|never|always` sets it. MANUAL is deliberately not offered. Covered by
`regression-test.uc` §3d, which fails if the live setting is ever 3, if requested and effective
modes diverge, or if a per-format preset is reintroduced into the source.

### Active Streaming Apps (Fire TV — post-cleanup)

| App | Package | Version |
| :--- | :--- | :--- |
| SmartTube Stable 4K | `org.smarttube.stable` | v31.57 |
| Netflix 4K | `com.netflix.ninja` | v13.1.3 |
| Disney+ Hotstar | `in.startv.hotstar` | v26.08.17.0 |
| Apple TV+ | `com.apple.atve.amazon.appletv` | v16.2.0 |
| SonyLIV | `com.onemainstream.sonyliv.android` | v9.39.2 |
| ZEE5 | `com.zee5.amazon` | v5.82.9 |
| MX Player TV | `com.mxtech.videoplayer.television` | v1.56.0A |
| VLC for Android TV | `org.videolan.vlc` | v3.5.4 |
| Plex for Android TV | `com.plexapp.android` | v10.21.1 |
| Send Files to TV | `com.yablio.sendfilestotv` | v1.2.2 |
| Projectivy Launcher | `com.spocky.projengmenu` | v4.55 |
| Shizuku Service | `moe.shizuku.privileged.api` | v13.5.4 |
| Companion Telemetry Agent | `com.home.tvagent` | v1.0.0 (Port 8080) |

### App-Aware DSP Auto-Switching (entertainment-poller.sh)

The `entertainment-poller.sh` daemon (PID tracked, running 24/7 on TP-Link) includes an **edge-triggered app-aware DSP routing block** that fires `mitv-api?cmd=sound_profile` whenever the Fire TV or Mi TV's own foreground app changes:

| App Pattern | Source | DSP Profile | Mi TV `sound_style` |
| :--- | :--- | :--- | :--- |
| `*netflix*`, `*smarttube*`, `*apple*`, `*zee5*`, `*mxtech*` | Fire TV | Cinema 5.1 Surround | `1` |
| `*hotstar*`, `*sonyliv*` | Fire TV | Voice Boost (Sports/Live) | `3` |
| `*launcher*` (idle) | Fire TV | Standard | `0` |
| `*plex*`, `*vlc*`, `*kodi*`, `*youtube*` | Mi TV native | Cinema 5.1 Surround | `1` |
| `*hotstar*`, `*sonyliv*` | Mi TV native | Voice Boost (Sports/Live) | `3` |
| `*flauncher*`, `*projengmenu*`, `*launcher*` | Mi TV native | Standard | `0` |

Switch history logged to `/tmp/dsp-auto.log` (rolling 100-line buffer).

**Two bugs fixed 2026-09-03:**
1. **Stuck profile on standby** — the original edge-trigger only fired on a *named* app change; when Fire TV went idle its foreground package becomes `"None"`, which never matched any case, so the DSP profile stayed on whatever was last active indefinitely (reproduced live: TV showed `sound_profile: "Voice Boost"` while sitting in standby). Fixed by adding an explicit `FTV_PKG = "None"` → reset-to-Standard branch.
2. **Mi TV native apps were invisible to DSP switching** — the poller already tracked the Mi TV's own foreground app (`TV_PKG`, used for telemetry) but never wired it into the DSP switcher, so content played directly on the TV (Plex/VLC/YouTube/Hotstar/SonyLIV via HDMI2/eARC, not through the Fire TV Stick) never triggered a profile change. Added a parallel `LAST_TV_PKG` edge-trigger mirroring the Fire TV logic, plus its own idle-reset.

Both idle-reset branches gate on `$TV_ONLINE` (Mi TV ADB reachability) — matching the existing switch logic's convention — so a reset only logs/fires when it can actually reach the TV, and safely retries on the next poll cycle otherwise.

### Display & Sound Calibration Controls (2026-09-03)

Live ADB-verified audit of the Mi TV's actual controllable surface, feeding a new **Display & Sound Control** panel in `mesh.html`'s Mi TV tab plus matching `mitv-api` commands:

| `mitv-api?cmd=` | `param` | Settings.Global key |
| :--- | :--- | :--- |
| `brightness` | `0-100` | `picture_brightness` |
| `contrast` | `0-100` | `picture_contrast` |
| `saturation` | `0-100` | `picture_saturation` |
| `sharpness` | `0-10` | `picture_sharpness` |
| `backlight` | `0-100` | `picture_backlight` |
| `hue` | `-50..50` | `picture_hue` |
| `color_temp` | `normal\|warm\|cool` | `picture_color_temperature` |
| `local_dimming` | `off\|low\|medium\|high` | `picture_local_dimming` |
| `game_mode`, `noise_reduction`, `film_mode`, `uhd_upscaling`, `allm` | `0/1` | `tv_picture_video_*` |
| `drc` | `0/1` | `sound_auto_volume_control` |
| `sound_effects` | `0/1` | `sound_effects` |
| `sound_delay` | `0-250` | `sound_spdif_delay` (70ms calibrated for Dolby Vision QLED + Sony ARC lip-sync) |
| `display_state` | — | read-only JSON dump of all of the above (drives the mesh.html panel's sliders on tab-open) |

**Important finding — two separate mechanisms, don't conflate them:** this `Settings.Global` surface (used above) is completely unrelated to the MediaTek `FactoryCommandService.apk`'s native `MtkTvConfig`/UART command IDs documented in `scratch/cve_2021_1048/mitv_factory_commands.md` — confirmed by exhaustive grep across all 237 decompiled classes of that APK (zero references to `Settings.Global` or `picture_*`). The UART service only fires over a physical service-jig serial port and is not reachable via ADB or network at all.

**Sound control architecture & verified DSP modes (Updated 2026-10-18):** Reverse engineering of `/system/priv-app/MiTVSettings/MiTVSettings.apk`, `TvService.apk`, and `Settings.Global` revealed that audio control is far deeper than a simple 0-3 slider. The MediaTek audio HAL supports **8 distinct native audio DSP profiles** (`sound_style` paired with `dolby_sound`):

| Profile | `sound_style` | `dolby_sound` | Color / HUD | Acoustic Profile & DSP Target |
| :--- | :---: | :---: | :---: | :--- |
| **Standard** | `0` | `8` | ⚙️ Slate (`#94A3B8`) | Balanced neutral studio reference profile with flat factory baseline. |
| **Theater / Cinema** | `1` | `2` | 🎬 Cyan (`#38BDF8`) | Cinema 5.1 dynamic surround soundstage, widening acoustic presence over HDMI ARC. |
| **Music** | `2` | `3` | 🎵 Pink (`#F472B6`) | Acoustic fidelity with neutral frequency response across midrange and highs. |
| **Dialogue Clarity** | `3` | `5` | 🗣️ Amber (`#FBBF24`) | Center vocal lift (+4dB in 1kHz–2.5kHz range) cutting muddy boxiness for speech intelligibility. |
| **Game** | `4` | `4` | 🎮 Blue (`#60A5FA`) | Ultra-low latency spatial soundstage optimizing console directional audio cues. |
| **Sports** | `5` | `9` | ⚽ Emerald (`#34D399`) | Stadium acoustic simulation, enhancing crowd ambience while keeping commentary audible. |
| **Night Mode** | `3` | `5` (DRC=1) | 🌙 Purple (`#A78BFA`) | Vocal lift paired with Dynamic Range Compression (`sound_auto_volume_control=1`) to prevent volume spikes. |
| **Custom / User** | `6` | `10` | 🎛️ Violet (`#C084FC`) | Unlocks manual 5-band graphic equalizer control. |

#### Graphic Equalizer Engines
* **5-Band Graphic Equalizer (`Settings.Global`)**: Actively controllable via registers:
  * `sound_equalizer_detail_100hz`: Sub-bass rumble and low-end punch
  * `sound_equalizer_detail_250hz`: Lower-mid warmth and vocal chestiness
  * `sound_equalizer_detail_1000hz`: Vocal body and presence
  * `sound_equalizer_detail_2500hz`: Upper-mid clarity, consonants, and speech intelligibility
  * `sound_equalizer_detail_6000hz`: Treble brilliance, airiness, and sparkle
* **7-Band Native DSP Equalizer (`DtsEqualizerFragment`)**: Low-level MediaTek HAL filter mapped to `g_audio__aud_eq_band_1..7` spanning 100 Hz to 12 kHz.

#### Advanced Dolby Audio Processing (DAP)
* **Dolby Dialogue Enhancer (`sound_advanced_dialogue_enhancer` 0/1/2)**: Multi-stage speech isolation algorithm boosting conversational frequencies.
* **Dolby Volume Leveler (`sound_advanced_volume_leveler`)**: Broadcast loudness regulation preventing sudden jumps between apps and commercials.
* **Dolby Audio Processing Suite (`sound_advanced_dolby_ap`)**: Dolby virtual surround matrixing.

#### DTS Hardware Silicon & Virtualization Suite
The MediaTek processor embeds licensed DTS silicon decoders and post-processors:
* **Hardware Decoders (`OMX.MS.DTS.Decoder`)**: Bitstream decoding for Core DTS 5.1, DTS-HD Master Audio (up to 7.1ch / 192 kHz), and DTS Express LBR.
* **DTS Virtual:X (`sound_dts_virtual_x`)**: Psychoacoustic virtual height and 3D spatial soundfield engine (`dts_virtual_surround`).
* **DTS TruBass & Dialog Clarity (`sound_dts_tbhdx` & `dts_dialog_clarity`)**: Harmonic bass synthesis producing perceived sub-bass plus center vocal lift.
* **DTS-StudioSound 2 (`sound_dts_studio_enable`)**: Surround widening with integrated output peak limiter (`sound_dts_limiter`).
* **DTS Dynamic Range Control (`sound_dts_drc`)**: Late-night dynamic compression.

#### Hidden Vendor DSP Suites
* **DBX-TV Pro Suite**: `sound_dbx_enable`, `sound_dbx_total_sonic` (psychoacoustic bass & harmonic exciter), `sound_dbx_total_surround` (spatial widening), and `sound_dbx_total_vol` (loudness regulation).
* **Sonic Emotion 3D Audio**: `sound_sonic_emotion_premium` (algorithmic soundfield synthesis).

#### HDMI ARC Routing & Synchronization
* **HDMI eARC / ARC (`sound_output_switch=5`)**: Bitstream pass-through and 5.1 PCM to Sony HT-S40R 5.1ch soundbar (`hdmi_control_enabled=1`, `hdmi_arc_control_enabled=1`).
* **Downmix Matrix Modes (`sound_downmixmode`)**: `0` = Lo/Ro standard stereo downmix; `1` = Lt/Rt Pro Logic matrix-encoded surround downmix.
* **Calibrated Lip-Sync Delay (`sound_spdif_delay`)**: Hardware delay register (calibrated at 70–80 ms to synchronize HDMI ARC audio with 4K video post-processing).
* **Panel-Off / Audio-Only Streaming (`power_picture_off=1`)**: Shuts off the QLED panel and backlight completely while keeping the DSP and HDMI-ARC audio output alive for low-power music streaming.
* **Full Engineering Matrix**: Full breakdown and register catalog documented in [`mitv_hidden_services_and_hardware_matrix.md`](../mitv_hidden_services_and_hardware_matrix.md).

**Hidden Hardware Picture, Motion & Diagnostics Surface (Discovered 2026-10-18):**
Extracted from `MiTVSettings.apk`, `TvService.apk` (`mitv.service`), and `MediatekFactory.apk` (`mediatek.factorymenu.ui`):
1. **6-Axis CMS Color Tuner** (`tv_picture_color_tune_enable`): Full color management across 6 discrete axes (Red, Green, Blue, Cyan, Magenta, Yellow + Flesh Tone) with individual `Hue`, `Saturation`, `Brightness`, `Gain`, and `Offset` registers (0-100, center 50) in the MediaTek hardware LUT.
2. **11-Point & 20-Point White Balance** (`picture_white_balance11_enable`): 11 distinct IRE grayscale calibration steps (Gain 5%, 10%, 20%... 100% IRE) with discrete R/G/B gain trims.
3. **De-Contour / Debanding Filter** (`tv_picture_de_contour` 0-3): Hardware dithering engine that smooths out macroblocking and color stepping in dark scenes of compressed OTT feeds (Netflix, Prime Video, YouTube).
4. **AI-PQ & AI Super Resolution** (`picture_ai_pq_switch`, `tv_picture_ai_sr`): MediaTek APU neural processing engine that classifies scene types (face, sky, foliage, night) and applies targeted neural upscaling.
5. **MaxVivid Dynamic Saturation** (`tv_picture_video_maxvivid`): MediaTek proprietary color expansion boosting secondary color punch while protecting human flesh tones.
6. **Split-Screen MEMC Live Comparison Demo** (`tv_picture_video_mjc_demo_partition`): Splits the 75" display down the middle (left half = raw 24p/60p cadence, right half = real-time 120Hz MEMC interpolation).
7. **Discrete De-Judder & De-Blur Sliders** (`tv_picture_mfc_smooth`, `tv_picture_mfc_clear`): 0-10 sliders (`MFCCustomActivity`) for fine-grained motion flow tuning.
8. **HDMI True RGB Quantization Range** (`tv_picture_video_hdmi_rgb_range`): 0=Auto, 1=Full 0-255 (PC/Console), 2=Limited 16-235 (Broadcast).
9. **Panel-Off / Audio-Only Streaming** (`power_picture_off = 1`): Shuts down the 75" QLED Full-Array Local Dimming backlight and panel gate drivers completely while keeping music streaming through HDMI ARC / Sony HT-S40R soundbar.
10. **Hardware Lifetime Operating Hours** (`Settings.System.panel_display_time`): True hardware panel power-on time counter (verified at `7519` hours).
11. **MediaTek Engineering Menus** (`mediatek.factorymenu.ui`): Factory Menu (`FactorymenuActivity`: ADC adjust, overscan adjust, aging/burn-in mode, power-on behavior) and Design Menu (`MainmenuActivity`: Spread Spectrum Clocking `SSC Adjust`, Ursa MEMC co-processor flashing `FactoryUpgradeUrsaFwActivity`, and Panel Test Patterns `TestImagePatternsActivity`). Full reference catalog in `mitv_hidden_services_and_hardware_matrix.md`.

**Reference/cinema-accurate calibration applied 2026-09-03:** Brightness 50, Contrast 50, Saturation 50, Sharpness 1, Hue 0, Color Temp Warm, Local Dimming High, Film Mode ON, Noise Reduction OFF, Game Mode OFF, ALLM OFF — the standard "turn off the marketing modes" baseline. Not colorimeter-verified; Warm is the closest built-in preset to 6500K without a meter in the loop. Backlight left at prior value (room-brightness preference, not a color-accuracy variable). 4K Upscaling was subsequently turned back ON per follow-up preference for sub-4K sources.

**Reference calibration can be silently overridden by the TV's own native picture-mode presets, outside `mitv-api` entirely (found 2026-09-09).** The physical remote's Quick Settings/Picture Mode menu writes a native `picture_mode` index (e.g. `6`, a Vivid/Sport-style preset) that bulk-sets `picture_backlight=100`, `picture_sharpness=10` (max), and `picture_color_temperature=3` — a value **outside** `mitv-api.uc`'s own 0/1/2 (normal/warm/cool) enum, confirming the native menu has more temperature options than the dashboard exposes. This diverged silently from the documented reference values with no log trail, since it bypasses the API path entirely. Re-detected by comparing live `settings get global` values against this section's documented baseline. Fixed by re-applying `backlight`/`sharpness`/`color_temp` via `mitv-api`; note that writing individual fields this way promotes/demotes `picture_mode` to a different index (observed `6` → `7`, a "User/Custom" slot) since the values no longer match any factory preset exactly — expected behavior, not a bug. **Anyone checking "is the calibration still correct" should diff live `settings get global` values against this section, not just trust `mitv-api`'s `display_state` output alone if the native remote menu might have been touched.**

**Local dimming has no finer control than the single 4-level knob (off/low/medium/high → `picture_local_dimming` 0-3).** Verified 2026-09-08 via the live on-screen factory menu (`mediatek.factorymenu.ui/mediatek.tvsetting.factory.ui.factorymenu.FactoryMenuActivity`, launched and fully explored): top level is ADC Adjust / White Balance / OverScan / Other Options / Info; Other Options contains only Mute Color (Blue) / Test Pattern For Panel / UART to HDMI. No LD-speed/smoothing/dynamic-backlight setting exists anywhere in this menu — confirming the earlier finding that finer PQ/backlight-timing tuning is locked in the native `FactoryCommandService.apk` `CFG_AUD_*` config space, UART-only, unreachable from ADB. This is a genuine 192-zone FALD panel (independently corroborated by the AVS Forum and HiFiVision thread titles plus Xiaomi's spec sheet) — optimization is limited to picking off/low/medium/high per content type (High for HDR/dark cinematic content, Medium to reduce blooming around static bright elements like subtitles, Low/Off for fast-changing SDR/gaming to avoid backlight "pumping" and added input lag), not tuning parameters within a level.

### 1-Click Master Scene Macros (mitv-api + mesh.html)

Available via `mitv-api?cmd=scene&param=<name>` or the Scene Console on the Home tab of `mesh.html`:

| Scene | Trigger | What Happens |
| :--- | :--- | :--- |
| `cinema` | 🍿 Cinema Movie Night | Wake TV + Fire TV → HDMI 1 → Cinema 5.1 DSP → Backlight 45% → MJC/MEMC explicitly OFF → Volume converges to 65% |
| `sports` | 🏏 Live Sports | Wake TV + Fire TV → HDMI 1 → Voice Boost DSP → Backlight 85% → MJC/MEMC ON (`tv_picture_advance_video_mjc_effect=3`) |
| `music` | 🎵 Hi-Fi Acoustic | Sony 5.1 Music DSP → Backlight 15% ambient → Volume 50% |
| `night` | 🌙 Late-Night | Auto DRC Leveler ON → Backlight 20% → Volume converges to 25% |
| `all_off` | 🔌 All Standby | CEC sleep broadcast → TV + Fire TV + Soundbar all off |

Volume convergence in `cinema` and `night` scenes uses a multi-pass ADB keyevent loop to reach the target level from any starting point without a hard absolute set (which is unreliable on this TV firmware).

**MJC/MEMC was advertised but never written, until 2026-09-09.** The `sports` scene's response
message had always claimed "MEMC Smooth Motion" while its handler wrote only
`sound_output_switch`/`sound_style`/`picture_backlight`/`picture_intelligent_switch` — nothing
ever set `tv_picture_advance_video_mjc_effect`. The message string was the *only* thing
asserting it, the same hardcoded-string bug class as `sony_hometheater.arc_handshake`
(§14). Note the separate `display_preset` path was always correct (its `sports` preset carries
`mjc: 3`), so `cmd=display_preset&param=sports` genuinely enabled MEMC while
`cmd=scene&param=sports` — the button actually wired to the Scene Console — did not.
`cinema` now explicitly writes `mjc=0` for the same reason Voice/Cinema/Music/Standard reset
DRC (§14 bug 6): without it, running sports then cinema leaves frame interpolation
(soap-opera effect) switched on for a film. Covered in `regression-test.uc` §3b.

**Context on MJC**: MJC = Motion Judder Cancellation, MediaTek/MStar's name for MEMC frame
interpolation. It is how this 120Hz panel is ever driven above the source frame rate — the Mi
Q1 / QLED 75 has HDMI 2.1 ports and a 120Hz panel but **cannot accept a 4K120 input signal**
(a well-documented limitation of this model), so interpolation is the only path to 120Hz.
Correct default is OFF for film (soap-opera effect + added input lag) and ON for live sport.

### Mi TV Wake Behaviour

- **Soft standby (screen off, Android running):** `mitv-api?cmd=wake` (ADB KEYCODE_WAKEUP 224) works instantly.
- **Hard off (physical standby switch):** ADB connection drops, ping fails (100% loss). Only physical IR remote or HDMI-CEC from Fire TV remote wakes it.
- **Network Standby setting:** `Mi TV Settings → Device Preferences → Power → Network Standby: ON` keeps ADB alive through soft-off cycles. Not yet confirmed as permanently enabled.

### Key Files on Router (Entertainment Stack)

| File | Purpose |
| :--- | :--- |
| `/usr/bin/entertainment-poller.uc` | 24/7 telemetry + App-Aware DSP auto-switching daemon (pure ucode, Fire TV + Mi TV native apps, idle-reset on standby) |
| `/www/cgi-bin/mitv-api` | Mi TV REST endpoint (ucode runtime, repo source `mitv-api.uc`): wake, sleep, volume, sound_profile, hdmi_input, scene, test_channel, reboot, plus display calibration (`brightness`/`contrast`/`saturation`/`sharpness`/`backlight`/`hue`/`color_temp`/`local_dimming`/`game_mode`/`noise_reduction`/`film_mode`/`uhd_upscaling`/`allm`/`drc`/`sound_effects`/`display_state`), plus `surround_mode` (encoded-surround passthrough policy) `audio_profile` (coherent per-output-path audio profiles) and `audio_format` (live decoded-format readout + 2s on-screen HUD) |
| `/www/cgi-bin/firetv-api` | Fire TV REST endpoint (ucode runtime, repo source `firetv-api.uc`): all ADB controls + app launcher. **The repo's plain `firetv-api` (bash, `#!/bin/sh`) is a stale leftover from before the ucode port — it is NOT deployed and NOT the live script; always edit `firetv-api.uc`.** |
| `/www/cgi-bin/ac-api` | Panasonic Smart AC REST endpoint (ucode runtime): GET telemetry & POST **real device control via MirAIe MQTT** (`mosquitto_pub`) |
| `/www/cgi-bin/mesh-api` | Mesh Network REST endpoint (ucode runtime): live status, channel optimization, password sync |
| `/www/mesh.html` | Whole-home dashboard: Home/Dashboard/Mi TV/Fire TV/HT/AC/Performance tabs; Mi TV tab includes a Display & Sound Control panel (sliders + toggles, backed by `display_state`) |
| `/tmp/entertainment-status.json` | Live telemetry snapshot (regenerated every 3s) |
| `/tmp/dsp-auto.log` | Rolling 100-line DSP switch event log |
| `AUDIO.md` | Home-theatre audio operations guide: profiles, live format HUD, measured per-output format support, troubleshooting |
| `scratch/cve_2021_1048/mitv_factory_commands.md` | Full reverse-engineered reference for the separate, network-unreachable `FactoryCommandService.apk` UART command set — not part of the live control surface above |

---

[← Back to CLAUDE.md index](../CLAUDE.md)
