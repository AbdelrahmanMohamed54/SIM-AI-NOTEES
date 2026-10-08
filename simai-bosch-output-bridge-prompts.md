# SIM.AI Output Bridge: hardware integration with Bosch (DICENTIS / Integrus)

Four prompts, run in order. Each ends with a commit; migrate and push after each, as usual.

| # | Prompt | Result | Effort |
|---|---|---|---|
| H0 | Audio output spike | A measured decision on *how* the bridge plays 7+ discrete channels for 8 hours without drift | 1 day |
| H1 | Server side | Bridge keys, channel plan, health, dashboard "Hardware output" card, printable channel card | 2–3 days |
| H2 | Output Bridge app (MVP) | Windows app: connect, map languages → outputs, meters, loudness, silence/floor fallback, test tones | 3–4 days |
| H3 | Hardening and field kit | 8-hour soak, network-loss recovery, standby bridge, installer and auto-update, technician docs | 2–3 days |

## Before you start (manual)

1. **A Windows laptop with a wired Ethernet port.** On Route A, Dante must use wired Ethernet. If the laptop also needs internet, use a second network adapter or Wi-Fi.
2. **Route A test kit:** a **Dante Virtual Soundcard** licence (Audinate) and the free **Dante Controller**. If you have no Bosch DICENTIS to test with, a second computer running Dante Virtual Soundcard or Dante Via can act as the receiver.
3. **Route B test kit:** a USB interface with **8+ line outputs** (for example MOTU UltraLite-mk5 or Focusrite Scarlett 18i20), plus headphones to check each output.
4. **Later:** a test day with a real Bosch system, ideally borrowed or rented from your AV partner.

---

## PROMPT H0 – Audio output spike (decide the technology with measurements)

```
Read CLAUDE.md, packages/engine and services/translation-worker (how workers join LiveKit
and handle 48 kHz PCM), and the ingest agent's capture code.

GOAL
We will build an "Output Bridge": a venue app that subscribes to a session's lang:* tracks
and plays each language on its OWN discrete output channel of an audio device (Dante
Virtual Soundcard as a WDM/ASIO device, or a USB interface with 8+ outputs), for 8+ hours.
Browsers mix to stereo and cannot address discrete channels reliably, so this needs a
desktop app. Before building it, decide the technology with measurements. Spike code only
(spikes/output-bridge/), no production code.

CANDIDATES (build the smallest working prototype of each that is feasible on Windows):
 a) Electron: LiveKit JS client in the renderer, Web Audio graph with one MediaStream
    source per track → ChannelMergerNode → AudioContext destination with
    channelCount = device max, output device chosen with setSinkId.
 b) Node: @livekit/rtc-node to receive PCM frames per track + a native multichannel
    output library (RtAudio/PortAudio binding supporting WASAPI and ASIO), writing an
    interleaved N-channel buffer.
 c) Anything clearly better you find (state why).

MEASURE FOR EACH (write spikes/output-bridge/FINDINGS.md):
- Discrete routing: play a different tone on each of 8 channels (440 Hz × channel number)
  and verify by recording each output (loopback on the interface, or a second machine on
  Dante) that tone n is ONLY on output n.
- Device support: Dante Virtual Soundcard in WDM and ASIO mode, and one USB interface.
- Latency added by the bridge (LiveKit frame received → audio out).
- Clock drift: network audio is paced by the sender's clock, the device by its own. Run
  60 minutes and measure buffer growth/underruns per channel; the design must include
  adaptive resampling or buffer control so 8 hours never drifts or glitches.
- CPU and memory with 8 channels.
- Recovery: unplug the network for 30 s, then reconnect; device unplug/replug.
- Packaging effort (installer, auto-update, code signing).

DECIDE and write the recommendation with numbers. Commit the spike and findings, then
report. Do not build the product yet.
```

---

## PROMPT H1 – Server side: bridge access, channel plan, health and dashboard

```
Read CLAUDE.md, spikes/output-bridge/FINDINGS.md, the session model and targets, the
connection key ("ST1.") and publisher-token code, participant identities and listener
counting, the health data topic, the session monitor, Azure output modes (captions/audio),
the event card and PDF/print code, and session_audit.

GOAL
Everything the cloud needs so an Output Bridge can serve a session. No bridge app yet.

1. BRIDGE KEY
- Per session, the admin can create a "bridge key" (format like the connection key, e.g.
  "SB1." + base64url JSON with LiveKit URL + a SUBSCRIBE-ONLY token for lang:* tracks and
  the data topics the bridge needs; no publish except its own health topic). Revocable;
  regenerating invalidates the old one. Shown with copy button and QR on the session page.
- Bridge participants have identity "bridge:<id>", are NOT counted as listeners, not in
  analytics or the listener cap, and appear separately on the monitor.

2. CHANNEL PLAN (stored on the session, editable before and during the event)
- Rows: output channel number (1–32) → language (one of the session's targets) →
  Bosch channel number shown to guests (default = output number) → fallback when the
  language is unavailable: "silence" (default) or "floor".
- Validation: each language at most once; warn if a mapped language is captions-only
  (Azure output "captions"): the bridge needs audio, so offer "Turn audio on for mapped
  languages" (uses the existing output toggle).
- Changes are pushed live to connected bridges over the control topic; audit every change.

3. HEALTH
- Define the bridge health message (every 2 s): app version, device name and channel
  count, per output: level (RMS/peak), silent-for seconds, buffer/drift state, underruns;
  network state; standby/active role.
- Dashboard session monitor: a "Hardware output" card: bridge status (active/standby,
  last seen), one meter per mapped channel with language name, red alert when a channel
  whose language is producing audio has been silent at the bridge for > 5 s, or when no
  bridge heartbeat for 10 s.

4. GUEST CHANNEL CARD
- Printable "Channel card" (A6 and A4 PDF, Sim Trans branding, every script and RTL
  correct): "Channel 0 · Floor, Channel 1 · English …" from the channel plan; optional
  QR for the phone service on the same card. Add the channel numbers to the event card
  as an option.

TESTS
Bridge key scope (cannot publish audio, cannot join other sessions), revoke/regenerate,
not counted as listener; channel plan validation and live push; captions-only warning and
enable; health card states and alerts; channel card PDF with Arabic and Chinese.
No change to the attendee page or its AudioContext unlock sequence. No keys in logs.
Additive migrations only. Typecheck, lint, all tests green. Update CLAUDE.md STATUS and
DECISIONS LOG. Commit, then report with screenshots.
```

---

## PROMPT H2 – Output Bridge app (MVP)

```
Read CLAUDE.md, spikes/output-bridge/FINDINGS.md (use the chosen technology), and the H1
bridge key, channel plan and health message.

GOAL
apps/output-bridge: a Windows desktop app for the venue laptop that plays each SIM.AI
language on its own discrete audio output, for 8+ hours, safely.

FEATURES
- Connect: paste the bridge key (or scan its QR with the laptop camera if easy); show
  session name, languages and the channel plan from the server.
- Audio device: pick the output device (Dante Virtual Soundcard, USB interface…), show its
  channel count, sample rate 48 kHz; remember the choice.
- Channel map: shows the server channel plan; local override allowed with a clear
  "differs from plan" warning; output n plays only its language.
- Per-channel: level meter, mute, gain trim (±12 dB), "Test tone" and "Spoken ident"
  (pre-generated TTS: "Channel 3, French" in that language) to verify mapping on a real
  receiver before the event. One button "Test all channels" plays them in sequence.
- Loudness: each channel normalised to a common target (e.g. −23 LUFS short-term with a
  limiter at −1 dBTP), so guests don't re-adjust volume when switching channels.
- Silence handling: when a language has no speech, output true silence (no hiss, no
  clicks). When a language's track disappears or its worker fails, apply the plan's
  fallback (silence or floor audio from the floor track) within 2 s and back again when
  it returns.
- Clock/drift control per the spike (adaptive resampling or buffer control); never let
  latency grow over hours.
- Reconnect automatically (LiveKit), keep the device open, never crash the audio path on
  network loss; show "Reconnecting…" and keep outputs silent meanwhile.
- Health: send the H1 health message every 2 s; local log file for support.
- Big, simple UI for technicians: one row per channel (number, language, meter, status
  dot), red banner for any silent or failed channel, no hidden menus during the event.
- Prevent sleep while running; warn on close.

TESTS
Unit: channel mapping, loudness normaliser, fallback state machine, drift controller.
Integration (automated where possible): a fake 8-channel output device (or loopback)
verifying discrete routing and levels; LiveKit test room with 7 synthetic language tracks
(different tones) → each appears only on its output; fallback to floor and back;
reconnect after network loss. Manual checklist for Dante Virtual Soundcard and a USB
interface in the report.
No change to the attendee page. No keys in logs. Typecheck, lint, all tests green.
Update CLAUDE.md STATUS. Commit, then report with screenshots and measured latency.
```

---

## PROMPT H3 – Hardening, standby bridge and field kit

```
Read CLAUDE.md, apps/output-bridge and the H1 server pieces.

GOAL
Make the Output Bridge safe for paid events.

1. 8-hour soak (spend-guarded where providers are used): 7 languages looping a real
   recording, outputs recorded at intervals; report drift, underruns, memory/CPU over
   time, reconnects; pass = no glitch audible, latency flat, memory flat.
2. Standby bridge: a second laptop can connect with the same bridge key as "standby": it
   receives everything but outputs silence; the server marks one bridge active. If the
   active bridge's heartbeat stops for 5 s, the dashboard shows "Switch to standby" (one
   click) and the standby goes active. Document the physical side: Route A needs the
   Dante routes pre-made from both laptops or a Dante Controller preset to switch; Route B
   needs the standby's outputs patched through a switch or swapped cables. Never output
   from both at once.
3. Failure drills: kill the app, pull Ethernet for 60 s, unplug the USB interface, a
   language worker restart, LiveKit reconnect; record what the listener hears and the
   recovery time.
4. Packaging: Windows installer, auto-update channel, version shown in the app and in
   health; optional code signing documented.
5. Field documentation in docs/hardware/: setup checklists for Route A (Dante Virtual
   Soundcard settings, Dante Controller routing with one channel per multicast flow,
   DICENTIS language streams needing one DCNM-LDANTE licence each, same subnet as
   DICENTIS) and Route B (8+ output interface, TRS→cinch cabling, ground-loop isolators,
   INT-TX08/16/32 input-to-channel assignment, receivers with 8+ channels); a one-page
   pre-event test procedure; troubleshooting table.
Typecheck, lint, all tests green. Update CLAUDE.md STATUS and DECISIONS LOG. Commit, then
report the soak results and drill outcomes.
```

---

## After all four

Book a test day with your AV partner on a real Bosch system:
1. Connect the bridge (Route A or B).
2. Play the spoken ident on each channel.
3. Walk the room with a receiver and confirm every channel number says the right language.
4. Run a 30-minute rehearsal with a live speaker.
