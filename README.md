# Smart Puck

A round smart-display + voice satellite for Home Assistant, built on the
**Waveshare ESP32-S3-Touch-LCD-1.85C** (~$25) with ESPHome + LVGL. No cloud, no
custom firmware toolchain — one YAML file.

## What it does

- **Forecast ring** — the bezel is a 12-hour hourly forecast. Each segment is one hour
  (12 o'clock = now), colored on an absolute °F scale (freezing gets its own color), with
  per-hour precipitation glyphs: 1–3 drops for intensity, a bolt for storms, a flake for snow.
- **Clock face** — time/date dead-center; status badges beneath (music playing, package
  waiting, air-quality warning colored by EPA band).
- **Voice satellite** — `micro_wake_word` on-device wake word → your HA Assist pipeline,
  with a full-screen voice UI (listening/thinking/speaking states), music **ducking** while
  it listens, a watchdog so a hung pipeline can't wedge the screen, and a self-heal loop
  that re-arms the wake word if it ever silently dies.
- **Ears handoff** — an HA-exposed **"Wake word" switch** hands wake-word duty to a
  dedicated satellite (e.g. an HA Voice PE) while this device keeps push-to-talk and the
  converse mic. Every wake-word arm site (including the self-heal) routes through one
  gate script, so flipping the switch is the whole story — revert is one switch flip, no
  reflash. Ships OFF: flip it ON in HA if this device should be the wake-word listener.
- **Voice music that actually works** — HA automations (included) that own the "play X /
  play X by Y" sentence, search Music Assistant with artist qualifiers, and refuse to play
  garbage when speech-to-text mishears (token-overlap match guard).
- **Timer dial** — "set a timer for 10 minutes" covers the forecast ring with a dark arc
  that peels back as it drains, revealing the forecast underneath. Full red ring + DONE when
  it fires.
- **Camera portholes** — a doorbell press (or any event you choose) seizes the screen with
  a full-bleed Frigate camera frame, auto-closing after 45 s. One generic
  `show_camera(camera, title)` action; all policy lives in HA.
- **Alert modals** — mirror phone notifications, or weather-dressed alerts (storm / rain /
  snow / haze art) including a device-side "rain incoming" edge trigger and an outdoor-AQI
  alert fed by Pirate Weather v2.
- **Kind/tier alert modals** — a `show_kind_alert(kind, tier, title, message, art_url)`
  action where the *kind* (door / delivery / security / environment / fyi) picks the glyph
  and the *tier* picks the mood: **ambient** (calm scrim, 12 s), **notable** (amber, 30 s),
  **urgent** (alarm red, sticky until dismissed). Optional artwork is fetched at runtime
  from any URL the device can reach — per-tier mood images from HA's `/local/` are the
  starting point, and swapping `art_url` **per kind** in the automation is the upgrade path
  (a parcel still-life for delivery, a threshold for door, …) with zero firmware changes.
  A **live camera frame URL is a valid `art_url`** too (e.g. a porch camera for delivery
  alerts); on any failed fetch — including a dead camera — the modal honestly keeps the
  glyph-only look, never a stale frame. The included notify-mirror automation classifies
  your phone notifications into kind/tier for free.
- **Talks first** — system-initiated conversation: the included HA converse script shows
  the text on the puck, then a single `assist_satellite.start_conversation` call speaks it
  on your Assist satellite AND opens the reply window natively (no announce timing math —
  your music speaker stays music-only). Silence closes the exchange gracefully, and literal
  goodbye phrases ("no thanks", "that's all") end it in milliseconds. A `converse_listen`
  action (this device's mic, no wake word) remains for satellite-less installs. Build
  rituals on top — e.g. a once-daily presence-edge greeting.
- **Thermostat + media pages** — tap to cycle. Thermostat shows a dial where the fill's
  leading edge is the room, the white tick is the target, and heating rises clockwise while
  cooling descends counter-clockwise. Media page shows album art with a track-progress ring.
  Now-playing is **sticky**: Music Assistant's transition frames (empty titles mid-pause)
  never wipe it; the page only reverts to idle art after 36 h without playback. The
  play/pause tap drives a smart HA script that controls the Music Assistant entity in both
  directions (native `media_play` silently no-ops on MA-owned queues).

- **Side button (BOOT)** — the button furthest from USB-C: click = push-to-talk (no wake
  word), click again = cancel, double-click = dismiss any modal, long-press = voice volume
  cycle with an on-screen toast (persisted across reboots). The USB-C-side button is RESET.
- **Media idle art** — a vinyl record fills the round display when nothing plays.

## Hardware

- [Waveshare ESP32-S3-Touch-LCD-1.85C](https://www.waveshare.com/esp32-s3-touch-lcd-1.85c.htm)
  — 360×360 round LCD, ESP32-S3 N16R8, mic + speaker, CST816 touch.
- A Home Assistant install with the Assist pipeline you want to use.
- Optional integrations: Music Assistant (voice music), Frigate (portholes),
  Pirate Weather (forecast ring + AQI).

## Install

1. Copy `secrets.yaml.example` → `secrets.yaml`, fill it in.
2. Edit the `substitutions:` block at the top of `smart-puck.yaml` — your HA/Frigate URLs
   and entity IDs. That block is the entire per-install surface.
3. **First flash must be WIRED** (the config re-partitions flash; OTA can only write the
   app slot, not the partition table):
   ```
   esphome compile smart-puck.yaml
   esptool.py --chip esp32s3 write_flash 0x0 .esphome/build/smart-puck/.pioenvs/smart-puck/firmware.factory.bin
   ```
   Note: Docker Desktop on macOS cannot pass USB through — compile in Docker anywhere,
   but run esptool natively on the machine the cable is plugged into.
4. Every later update is OTA: `esphome run smart-puck.yaml`.
5. Drop `ha/smart_puck_package.yaml` into HA's `/config/packages/` (enable packages if you
   haven't) and reload templates. Import the automations in `ha/automations/` and point
   them at your entities.

## Repo layout

| Path | What |
|---|---|
| `smart-puck.yaml` | The entire device: display, touch, LVGL UI, voice, actions |
| `ha/smart_puck_package.yaml` | HA-side feeds + scripts: forecast ring, track progress, outdoor AQI, smart play/pause, converse |
| `ha/automations/` | Voice music handler, media controls, porthole + alert triggers, goodbye |
| `backgrounds/` | AI-generated art for weather / thermostat / voice states |
| `GOTCHAS.md` | The traps this build hit so you don't have to |
| `secrets.yaml.example` | Template for your credentials |

## Read GOTCHAS.md before you customize

Everything non-obvious this build learned the hard way is in there: why rotation goes in
the `lvgl:` block and nowhere else, why you verify icon codepoints by *rendering* them, how
to diagnose a garbled screen that fixed itself, why `online_image` must never fall back to
its buffer, and more.

## License

MIT
