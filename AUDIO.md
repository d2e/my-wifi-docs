# 🔊 Home Theatre Audio — Operations Guide

How to get the best sound out of this system, how to tell what it is actually doing, and what
it genuinely cannot do. Everything here was measured on the hardware on **2026-10-18**, not
taken from a spec sheet.

Engineering detail and the evidence behind each claim lives in `CLAUDE.md` §13–14; the
register catalogue (with per-key verification status) is in
[`mitv_hidden_services_and_hardware_matrix.md`](mitv_hidden_services_and_hardware_matrix.md).
This file is the practical one.

---

## 1. The chain, and the one constraint that governs everything

```
   source app / file
          │
   ┌──────▼──────────────────────────────┐
   │  Xiaomi Mi QLED 75 (192.168.0.10)   │   decoders: AC-3, E-AC3, DTS, DTS-HD
   │                                     │   encoders: NONE
   └───┬─────────────────────────┬───────┘
       │                         │
  internal speakers        HDMI 2 eARC port, but the link
  STEREO ONLY              negotiates as PLAIN ARC
  (channel mask 0x0003     (mEarcConnected: false)
   on every profile)             │
                           ┌─────▼──────────────────┐
                           │ Sony HT-S40R 5.1       │  decodes: Dolby only
                           │                        │  no DTS, no TrueHD, no Atmos
                           └────────────────────────┘
```

**The constraint:** plain ARC is an S/PDIF-grade pipe. It carries either a *compressed 5.1
bitstream* or *2.0 PCM* — never multichannel PCM. And the TV has **no audio encoder of any
kind**, so it cannot decode something and re-encode it to AC-3 to fit down that pipe.

Three consequences follow, and they explain every design decision below:

1. **Dolby is the only format that reaches the Sony as real 5.1.** It gets passed through
   untouched and the soundbar decodes it.
2. **DTS always ends up stereo on the soundbar.** The TV decodes it (the Sony can't), and
   stereo PCM is all the pipe will carry.
3. **TV-side DSP and discrete 5.1 are mutually exclusive.** The EQ, Virtual:X and Dolby DAP
   only act on the PCM path — which is exactly the path that costs you the surround channels.

---

## 2. Using it: the dashboard

`http://192.168.0.12/mesh.html` → **Mi TV** tab → **Audio Output** panel.

```
Best Audio Profile                        Sony 5.1 · active
┌──────────────────────┬──────────────────────┐
│ 🔊 Sony 5.1          │ 📺 TV Speakers       │
│ Dolby bitstream ·    │ DTS Virtual:X ·      │
│ discrete 5.1         │ dialogue lift        │
└──────────────────────┴──────────────────────┘
Dolby passes through untouched as discrete 5.1. DTS is decoded on the
TV and arrives as stereo — plain ARC carries no multichannel PCM…

┌────────────────────────────────────────────┬──────────────┐
│ 🔊 Dolby Digital Plus · 6ch — passed        │ 🔍 Show on TV│
│    through — Sony decodes                   │              │
└────────────────────────────────────────────┴──────────────┘
```

**One click applies a whole chain** — sink, surround policy, DSP state, EQ and lip-sync delay
together. There is deliberately no pile of individual toggles to combine by hand, because the
right value for most of them depends on which output you are on.

### The badge

| Badge | Meaning |
|---|---|
| 🟢 `Sony 5.1 · active` | Routed to ARC **and** the DSP state matches that path |
| 🟢 `TV Speakers · active` | Routed to the speakers **and** the DSP chain is engaged |
| 🟠 `custom (…)` | **Sink and DSP disagree** — a half-applied state |

That amber case is the one worth paying attention to. "Routed to ARC but Virtual:X still on
from the speaker profile" is a real state the TV can sit in, it is silently wrong, and nothing
in the stock UI would ever show it to you. Click either profile button to resolve it.

### What each profile sets

| | 🔊 **Sony 5.1** | 📺 **TV Speakers** |
|---|---|---|
| Sink | `sound_output_switch 5` (HDMI-ARC) | `sound_output_switch 7` |
| Surround policy | `encoded_surround_output 0` (AUTO) — Dolby out as bitstream, DTS decoded locally | n/a (stereo sink) |
| DTS Virtual:X suite | **off** — bypassed on a passthrough bitstream anyway | **on** — Virtual:X, surround, dialog clarity, TruBass, DTS DRC |
| Dolby | Theater (`sound_style 1`, `dolby_sound 2`), DAP on | `sound_style 0`, DAP on, dialogue enhancer 2 |
| 5-band EQ | all flat | `0 / +2 / +1 / +3 / +2` |
| Lip-sync | `sound_spdif_delay 70` ms | `0` (no ARC in the path) |

**Why the speaker EQ is shaped that way:** no 100 Hz boost, because two small sealed drivers
physically cannot move that air — asking them to only eats headroom and adds distortion. The
lift sits at 250 Hz for warmth and 2.5 kHz for the consonant band, which is where TV-speaker
dialogue actually falls apart.

**Why Virtual:X is off for Sony:** it would do nothing (the bitstream bypasses TV
post-processing), but it would make the TV *report* a processing state it isn't in. Leaving it
on is how you end up with the amber `custom` badge and a confusing readout.

---

## 3. Knowing what is actually being decoded

The format strip under the profile buttons reads the **audio HAL**, not the file and not the
app — so it tells you which box is doing the decoding right now.

**🔍 Show on TV** flashes the same information on the TV screen for **2 seconds** and refreshes
the strip at the same time. Useful when you are across the room with a remote in your hand.

### Reading it

| Strip shows | What it means |
|---|---|
| 🔊 `Dolby Digital Plus — Bitstream → Sony decodes` | Best case. Discrete 5.1 at the soundbar. |
| 📺 `Dolby Digital Plus → out: PCM 16-bit — Decoded on TV → TV speakers` | A Dolby track, decoded by the TV for its own speakers. |
| 📺 `PCM 16-bit — Decoded on TV → stereo over ARC` | The TV decoded it; the soundbar is getting stereo — usually a DTS track. |
| 📺 `PCM 16-bit … (app decoded it in-process — source codec not visible to the system)` | The app did its own decoding. See the caveat below. |
| 🎧 `Idle — no active audio output` | Nothing is playing. |

If you are watching something and see a plain PCM line while on the Sony profile, the file's
audio track is a format the soundbar can't take. See §5.

### On TV speakers, will it still tell me it's Dolby?

**Usually yes, but it depends on who does the decoding** — and the readout is explicit about
which case you're in:

- **The TV's audio HAL decodes it** → the strip shows the compressed format directly
  (`Dolby Digital Plus … Decoded on TV → speakers`). The Speaker device port declares its own
  `AUDIO_FORMAT_AC3` / `E_AC3` / `E_AC3_JOC` profiles, so a Dolby stream can be opened straight
  against the speakers and decoded in-house.
- **The platform decoder decodes it before the mixer** → the output side reads PCM, but the
  source codec is recovered separately from `dumpsys media.metrics` and shown as the headline,
  with the PCM output after the arrow.
- **The app decodes in its own process** (common for streaming apps, reported as `audio/raw`)
  → the original codec is genuinely not observable anywhere in the system. The strip says so
  in plain words rather than implying the content was PCM to begin with.

> ⚠️ **No channel count is shown for a passthrough stream, and that is deliberate.** Verified
> during real Dolby playback: the E-AC3 thread reports channel mask `0x3`
> (front-left, front-right) because that is the **IEC61937 carrier framing**, not the audio
> inside it. Rendering "2ch" beside "Dolby Digital Plus" would read as stereo when the Sony is
> in fact decoding discrete 5.1 — exactly backwards. The real channel count lives inside the
> bitstream and only the soundbar ever sees it.
>
> ⚠️ **"Compressed" does not by itself mean "passed through to the Sony."** The same
> `AUDIO_FORMAT_E_AC3` means the soundbar is decoding when the sink is ARC, and the TV is
> decoding when the sink is the speakers. Only the sink distinguishes them. The readout is
> sink-aware; an early version was not, and announced "Sony decodes" on the speaker path.

### Command line

```bash
curl -s "http://192.168.0.12/cgi-bin/mitv-api?cmd=audio_format"            # read
curl -s "http://192.168.0.12/cgi-bin/mitv-api?cmd=audio_format&param=hud"  # read + 2s HUD
curl -s "http://192.168.0.12/cgi-bin/mitv-api?cmd=audio_profile&param=sony"
curl -s "http://192.168.0.12/cgi-bin/mitv-api?cmd=audio_profile&param=tv"
curl -s "http://192.168.0.12/cgi-bin/mitv-api?cmd=audio_profile&param=status"
```

The 2-second fade is the companion agent's own `VolumeOverlayManager.DISMISS_DELAY_MS`; the
HUD reuses the same overlay as the sound-mode popups.

---

## 4. Format support, measured

### What the TV can decode

| Format | Decode | Note |
|---|---|---|
| Dolby Digital (AC-3) | ✅ 5.1 | `OMX.MS.AC3.Decoder`, channel-count max 6 |
| Dolby Digital Plus (E-AC3) | ✅ 5.1 | `OMX.MS.EAC3.Decoder`, max 6 |
| DTS / DTS-HD MA / HR / Express | ✅ | `OMX.MS.DTS.Decoder`, full family |
| Dolby Atmos (DD+ JOC) | ⚠️ core only | 6-channel cap, no object renderer — height discarded |
| Dolby TrueHD | ❌ | no decoder, and not even a passthrough format in the audio policy |
| Dolby AC-4 | ❌ | passthrough format listed, no decoder |

### What reaches each output

| Track | → Sony (ARC) | → TV speakers |
|---|---|---|
| Dolby Digital / DD+ | **discrete 5.1** ✅ | stereo downmix |
| DTS (any) | stereo | stereo |
| Atmos (DD+ JOC) | 5.1 bed, no height | stereo |
| TrueHD | nothing — pick another track | nothing |

**Atmos does not happen on this system**, blocked twice over: the TV caps at 6 channels with no
object renderer, and the Sony has no Atmos hardware.

### Can both outputs play at once?

**No.** `sound_output_switch` is a single-select enum
(`0=BLUETOOTH 1=ALL_SCO 2=USB 3=HEADSET 4=HEADPHONE 5=HDMI_ARC 6=SPDIF 7=SPEAKER 8=AUX`),
HDMI System Audio mode actively mutes the internal speakers by design, and AudioFlinger has a
single output thread with no duplicating output. Forcing it would need root. It would also
sound bad — the ARC path needs ~70 ms of compensation the speaker path doesn't, so
unsynchronised dual output would comb-filter.

---

## 4b. Controlling the Sony soundbar

The dashboard shows a soundbar line under the format strip:

```
🔈 Sony HT-S40R — vol 57/100 · System Audio · ARC up · as of 22:24:54
```

**That number is the soundbar's own volume, not the TV's.** With System Audio Mode active the
two are different scales, and what you actually hear is the Sony's. It comes from the CEC
`<Report Audio Status>` frame the soundbar sends back after each key press, which is why a
timestamp is shown — it is last-known state, not a live poll. It refreshes whenever volume or
mute changes.

### What can be controlled, and what cannot

| Function | Reachable? | How |
|---|---|---|
| Volume up/down | ✅ | TV forwards the key as CEC `<User Control Pressed>` (0x44). Already wired to the dashboard volume control. |
| Mute | ✅ | same path |
| Power / standby | ✅ | CEC `<Standby>` / System Audio Mode |
| **Sound field / theater mode** | ❌ | Sony vendor opcode `0xA0`, no injection route available |
| **DRC on/off** | ❌ | not mapped to CEC at all |
| **Subwoofer level** | ❌ | not mapped to CEC at all |
| **Surround L/R trims** | ❌ | not mapped to CEC at all |

The last four are **IR-only**. The first three already work today.

### Why the vendor commands can't be sent

Sony's BRAVIA Sync frames are real and confirmed on this bus — the HT-S40R sits at logical
address `0x05` reporting vendor ID `524358` (`0x080046`, Sony's OUI), and `<Vendor Command With
Id>` traffic is flowing. The blocker is purely that nothing here can *inject* a frame: there is
no libCEC on Android and no `/dev/cec*` node, `cmd hdmi_control` has no shell implementation on
this build, `/dev/mik!cec` is mode `666` but SELinux-denied to the shell domain, and
`sendVendorCommand()` needs a privileged permission. Full detail in CLAUDE.md.

**If you want the remaining four**, the practical additions are a USB-CEC adapter (Pulse-Eight
or similar, for raw `0xA0` vendor frames) on the router or a Pi, and/or an IR blaster speaking
Sony SIRC for the settings that have no CEC representation at all. Either one plugs into the
existing dashboard the same way the other endpoints do.

## 5. Getting the most out of it

**Prefer Dolby tracks in your library.** This is the single highest-impact habit. Remuxes on
the DLNA share frequently ship DTS-HD MA as track 1 and AC-3 5.1 as track 2 — defaulting to
track 1 silently costs you the surround channels. Switching the audio track in VLC/Plex is the
difference between stereo and real 5.1, and no setting on the TV or router can substitute for
it.

**Use the TV Speakers profile when the soundbar is off.** On that path DTS stops being
second-class (everything is stereo anyway), and the full DSP chain engages — Virtual:X is the
only spatial processing this system can produce, and the Sony cannot do it at all.

**If a DTS file ever plays silent** rather than in stereo, the fallback is
`mitv-api?cmd=surround_mode&param=never`, which forces the TV to decode everything. Cost:
Dolby drops to stereo too, so switch back afterwards with `param=auto`.

---

## 6. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Badge shows amber `custom` | Sink and DSP disagree. Click either profile button. |
| Expected 5.1, strip says PCM | The track is DTS (or the app is decoding internally). Switch audio track to a Dolby one. |
| Audio out of sync on the soundbar | `sound_spdif_delay` should be **70** on the Sony profile; re-apply the profile. It has been found reset to 0 before. |
| Dialogue hard to follow on TV speakers | Apply the TV profile — it sets dialogue enhancer 2 plus the 2.5 kHz EQ lift. |
| Soundbar keeps dropping/reconnecting ARC | See CLAUDE.md §14: **reboot the TV first**, before any protocol-level diagnosis. |
| Nothing at all from the soundbar | `mitv-api?cmd=resync_arc`, then check `hdmi_control_enabled` / `hdmi_arc_control_enabled` are 1. |

### Don't do this

- **`encoded_surround_output = 3` (MANUAL)** — this firmware silently clamps it to
  `FORCE_ENCODED_SURROUND_ALWAYS`, which force-passes *every* format including the DTS the
  Sony can't decode. It writes and reads back cleanly; only `dumpsys media.audio_policy`
  reveals the clamp. The `surround_mode` endpoint deliberately refuses to offer it.
- **The DBX / Sonic Emotion / DTS-StudioSound keys** — all inert on this firmware despite
  appearing in the settings APK. See the verification table in the hardware matrix.
- **Don't burn time trying to send Sony BRAVIA Sync vendor CEC frames from the router or
  ADB.** The frames are real (the soundbar is on the bus at logical address `0x05` with Sony's
  `0x080046` vendor ID), but every injection route is closed: no libCEC on Android, no
  `cmd hdmi_control` shell command on this build, `/dev/mik!cec` blocked by SELinux despite
  mode 666, and `sendVendorCommand()` behind a privileged permission. A USB-CEC adapter is the
  only realistic path. Full detail in CLAUDE.md.

---

## 7. Tests

Any change to the audio surface must be followed by the full suite (`/test` skill):

```bash
ssh root@192.168.0.12 'cd /usr/bin && ucode test-poller-lib.uc'   # incl. parseActiveAudioFormat
ssh root@192.168.0.12 ucode /usr/bin/regression-test.uc           # §3d/§3e/§3f cover audio
cd tv-telemetry-agent && ./gradlew test
```

- **§3d** — encoded-surround mode is never the clamped MANUAL value, and requested vs
  effective policy agree.
- **§3e** — profile coherence, Virtual:X matches the sink, ARC carries a non-zero lip-sync
  delay, optical maps to `6` not `1`.
- **§3f** — format readout schema, PCM is never reported as passthrough, passthrough is never
  claimed on a non-ARC sink, no idle-placeholder (8000 Hz) thread leaks through.

The HUD is asserted at source level rather than fired, so running the suite does not flash an
overlay on the TV.
