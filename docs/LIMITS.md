# Known limits

Honest list, so nobody is surprised after flashing.

## Dual microphones: not active

The board has two microphones (ES7210). Today they are used for **direction
finding** (the head turns toward whoever is speaking) and for the wake word tap,
which averages both. The **recording that goes to Muse is single-mic**.

There is an experimental AFE path (`CONFIG_K151_AFE_RESIDENT`, esp-sr
`SE(BSS) + VAD` over `MM` input). It creates successfully — about 2 KB of
internal RAM and 390 KB of PSRAM with `AFE_MEMORY_ALLOC_MORE_PSRAM` — but once
wired into the live voice path the recording develops dense clicking after a few
seconds, the detector reads it as continuous sound and the turn runs to the 15 s
cap. It is therefore **off by default**.

If you want to work on it: the suspicion is the feed/fetch pacing (the AFE
returns 512-sample frames for 1024-sample feeds, and the current loop fetches
once per feed) or input-ring overruns. An offline bench — a known stereo WAV fed
through `muse_afe_feed()`, with fed/fetched byte counters, ring-overflow counts
and per-frame RMS — is the way to settle it. Details of what was tried are in
`DUALMIC_AFE_BRIEF.md` in the author's tree.

## Echo cancellation: off

AEC needs the speaker reference wired into the AFE input and a delay
calibration. With it on, the AFE costs ~17.5 KB of internal RAM and the DMA pool
drops below the 16 KB the tunnel keeps in reserve, so the Muse session cannot
connect. Off until the memory question has an answer.

Consequence: **you cannot interrupt it while it is speaking** — the microphones
are not being read during playback.

## Network

- The device needs a working DNS resolver. A resolver that answers with
  fake-IP addresses (some proxy setups do) works only if the device also routes
  through that proxy; otherwise every TLS connect fails with
  `getaddrinfo() returns 202`.
- A static IP is supported (`CONFIG_HOMEHUB_STATIC_IP/GATEWAY/DNS`) and is the
  recommended setup on networks whose DHCP hands out a proxy host as the
  gateway — a laptop that reboots takes the device offline otherwise.

## Bluetooth

- Started only on an unprovisioned device, and only after the display is up.
- Released once provisioning completes (`esp_bt_mem_release`), which cannot be
  undone without a reboot: re-pairing means power-cycling.

## Voice turn length

A single turn is capped at 15 s (`MAX_SECS` in `muse_voice.c`, mirrored by a
20 s upload buffer in `muse_chat_session.cpp`). The reply side waits up to 180 s
for the first message, and a whole turn is capped at 480 s.

## TTS

The default endpoint points at a local TTS server on your LAN
(`http://192.168.1.10:8880/v1/t2a_v2` is the shipped placeholder, MiniMax T2A v2
compatible) — set it to your own host. It is a Kconfig setting
(`CONFIG_MUSE_TTS_MINIMAX_URL`); if it is wrong you get the reply as text only,
with `muse_tts: can't reach MiniMax` in the log after a timeout.

## Not tested

- 24-hour soak on this build (the 10-minute run was clean: tunnel up the whole
  time, `kb=0/0`, no audio glitches).
- Muse app device list visibility after a fresh pairing (the device reports
  `community pairing: no manufacturer attestation`, which may or may not matter
  to the app).
