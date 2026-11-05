# Wake word — how it works, how to tune it

## Enabling

`CONFIG_K151_WAKE_WORD=y` (menuconfig: *K151 wake word*). It pulls in
`espressif/esp-sr 2.6.1` and expects the `model` partition from
`partitions_muse.csv`:

```
model, data, 0x40, 0x830000, 2M
```

The build copies `esp-sr`'s model image there, so `idf.py flash` is enough. If
the partition is missing or empty the engine logs `no model partition, or it
holds no model` and the device simply works without a wake word.

The model is `wn10_himuse` ("Hi / Hey Muse"), one of the WakeNet10 words that
ship with esp-sr. WakeNet10 word indices are 1-based — `get_word_name(data, 0)`
is out of range, which the engine hits once and then never again.

## How it runs

- A **second tap** in `muse_audio_read()` feeds the engine from the same I2S
  read: both microphones averaged, shifted right by `WAKE_GAIN_SHIFT` (default
  `1`, i.e. −6 dB) so that speech at 30 dB mic gain lands near the −20 dBFS the
  recogniser expects.
- The engine runs on its own task with a **PSRAM stack** and a PSRAM ring
  (16 KB, 500 ms). Its working set is about 2.6 KB of internal RAM plus pinning
  the esp-sr model in flash.
- The tap only copies and returns; it never blocks the voice task.
- Detection is debounced by 1.5 s and followed by `clean()`, so one utterance
  counts once and the next word does not fire immediately.
- The engine starts **after** Muse comes online (the link is what a turn needs,
  and it keeps `link.register`'s 5 KB internal block free at boot). If the link
  never comes up, it now waits forever and logs `wake word still waiting` once a
  minute instead of giving up — a boot that happened while Wi-Fi was slow used
  to leave the device permanently deaf.

## Tuning

Everything is observable from the serial console:

```
hb t=120s ... wk=7 kb=0/0 ...
```

- `wk=` — wake words heard since boot, after the debounce. If you speak and this
  does not move, the engine is not detecting: check `muse_wake: feed:` lines —
  `loudest` is the level reaching the recogniser.
- `muse_wake: mics: L … R …` — raw microphone levels, every 5 s.
- `wake word heard: Hi/Hey Muse (-71.2 dBFS, peak 31)` — the dBFS here is the
  *triggering frame*, which is usually the tail of the word; it looks quiet even
  for a perfect detection. Read it as a label, not as a level.

| Symptom | Knob |
|---|---|
| Misses normal speech | `WAKE_GAIN_SHIFT` in `muse_wake.c`: `1` (−6 dB) is the default; `0` is +0 dB and may clip |
| Fires on the TV / its own voice | `DET_MODE_95` → `DET_MODE_90` in `create()`; raise the debounce |
| Slow to respond | The engine starts only after Muse is online (~10 s after boot) |
| Wake ignored, `wake word: Muse is offline` in the log | The link is down; the screen wakes and shows `MUSE OFFLINE` instead of starting a turn |

## Memory

Measured on a CoreS3 with the display, Wi-Fi and the Muse session up:

| | Internal | PSRAM |
|---|---|---|
| Wake engine resident | ~2.6 KB + esp-sr statics | 16 KB ring + 4 KB stack |
| Whole firmware | ~33 KB free / 20 KB largest block | ~2 MB free |

The internal RAM is the scarce resource on this board: the display's DMA buffers
need a large block at boot, and `link.register` needs 5 KB contiguous. Both are
why the engine starts late and why BLE is released after provisioning (~39 KB
back).
