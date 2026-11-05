# StackChan × Muse — wake word edition

On-device wake word for the **M5Stack StackChan (CoreS3)** running the
[Muse Gadget SDK](https://github.com/facebookincubator/muse-gadget-sdk): say
**“Hi Muse”** and the robot starts a conversation, hands free. The Muse cloud
session, the phone pairing and the avatar all keep working — this is an addition
to the existing port, not a replacement.

```
   "Hi Muse" ──▶ esp-sr WakeNet10 on chip ──▶ wake wakes the screen,
                                              opens a turn, streams the note
                                              to the Muse VM as before
```

## What it is

- **Wake word**: esp-sr 2.6.1 WakeNet10 `wn10_himuse` ("Hi / Hey Muse"), running
  on the ESP32-S3, fed from a second microphone tap. No network, no cloud ASR.
- **Muse path untouched**: BLE provisioning, the `hatch.metaaivm.com` session,
  captions, TTS playback and the avatar all behave as before.
- **BLE on demand**: the Bluetooth stack is started *after* the display is up and
  released once the device is provisioned, giving back ~39 KB of internal RAM.
  Re-pairing is a button press away.
- **Configurable TTS endpoint**, so replies can be spoken by a local TTS server
  instead of the cloud. The shipped default is a placeholder — point it at your
  own MiniMax-T2A-v2-compatible server.

## What it is not

- **Not dual-microphone noise suppression.** Both microphones exist and are used
  for direction finding, and there is an experimental AFE branch, but it is off
  by default: its data stream is not yet stable in the live voice path. See
  `docs/LIMITS.md`.
- Not a Muse product, not affiliated with Meta or M5Stack.

## Requirements

| | |
|---|---|
| Board | M5Stack StackChan (K151, CoreS3 / ESP32-S3, 16 MB flash, 8 MB PSRAM) |
| SDK | [StackChan-Muse](https://github.com/WillLingCom/StackChan-Muse) fork of the Muse Gadget SDK |
| ESP-IDF | v6.0.1 (v6.0.x should work) |
| Wake model | esp-sr 2.6.1, `wn10_himuse`, in a 2 MiB `model` partition at `0x830000` |
| Token | a Muse SDK token from <https://gadgets.muse.ai/settings/sdk-tokens> |

## Build

```sh
git clone https://github.com/WillLingCom/StackChan-Muse
cd StackChan-Muse

# 1) apply this patch
git apply /path/to/stackchan-wake.patch

# 2) your Muse SDK token: private/k151.local.cfg is read as an sdkconfig default
#    and is gitignored, so the token never enters the tree
printf 'CONFIG_GADGET_SDK_TOKEN="mgst_..."\n' >> private/k151.local.cfg

# 3) build (enable "K151 wake word" in menuconfig the first time)
cd firmware/esp32
idf.py -B build-k151-wake set-target esp32s3     # first time only
idf.py -B build-k151-wake menuconfig             # Component config → Muse → K151 wake word

idf.py -B build-k151-wake build
idf.py -B build-k151-wake -p /dev/cu.usbmodem* flash
```

The build writes the wake model into the `model` partition (`0x830000`) as part
of `idf.py flash`; no separate step is needed.

## First run

1. The device advertises as `MuseGadget-XXXXXX`; pair it with the Muse app.
2. Once paired it reboots, joins your Wi-Fi and connects to the Muse VM.
3. Say **“Hi Muse”** — the screen wakes and a turn starts. Talk; it ends on
   ~0.8 s of silence and answers out loud.

If the wake word is ignored, see `docs/WAKE_WORD.md` (sensitivity, gain, and the
`wk=` counter in the heartbeat).

## The patch

Everything is in one patch against the StackChan-Muse fork. The interesting
files:

| File | What |
|---|---|
| `components/muse/muse_wake.c` | model load, microphone tap, debounce, deferred start |
| `components/muse/muse_input.c` | wake → turn, `MUSE OFFLINE` caption when the link is down |
| `components/muse/muse_audio.c` | second tap next to the recorder/DOA tap |
| `components/muse/Kconfig` | `CONFIG_K151_WAKE_WORD` |
| `components/muse/idf_component.yml` | conditional esp-sr dependency |
| `partitions_muse.csv` | `model` data partition, 2 MiB at `0x830000` |
| `main/app.c` | heartbeat counters (`wk=`, `kb=`, `min=`), boot reset reason |
| `main/ble_server.c`, `main/app.c` | BLE started after LVGL, released when provisioned |
| `components/muse/muse_tts.c`, `Kconfig` | TTS endpoint as a setting |

## Licence

Apache-2.0, same as the SDK it patches. See `NOTICE`.
