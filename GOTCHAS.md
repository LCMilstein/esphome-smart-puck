# Gotchas

Everything below was paid for with a real on-device failure during this build.


## Architecture rule

**The device renders; Home Assistant decides.** Expose `api: actions:` (e.g. `show_camera`,
`show_alert`) and put ALL policy — what triggers them, thresholds, cooldowns — in HA
automations. Changing policy must never require an OTA.

## Rotation + touch (agents reliably get this wrong)

- Rotation goes in the **`lvgl:` block**: `rotation: 180`. NOT on the display (rejected/ignored
  in LVGL mode) and NOT via `touchscreen: transform:` mirrors.
- **Never mirror the touchscreen to "fix" touch** — LVGL already rotates touch for its own
  widgets; a touchscreen transform double-flips every interactive widget.
- **`on_touch:` lambdas run OUTSIDE LVGL** and still see raw chip coordinates. Compensate
  locally: for 180°, `visual = (dim-1) - raw` on both axes.

## Render-fault diagnosis (tofu glyphs, solid ring, garbled UI)

1. **`Reset Reason` + uptime FIRST.** Splits crash-reboot from live render fault instantly —
   don't hunt a crash that never happened.
2. The `debug`/`sensor` diagnostics (**Loop Time, Heap Free, Heap Min Free, Fragmentation,
   Free PSRAM**) land in HA's recorder → past incidents are diagnosable retroactively.
3. Discriminate by what's on the panel: **tofu boxes were DRAWN** (LVGL rendered fallback ⇒
   draw-time failure, e.g. glyph/mask alloc) vs **frame FROZEN** (loop starved ⇒
   `lv_timer_handler` never ran). Both recover without reboot. Loop Time spiking (43→1770ms)
   = starvation evidence; flat heap does NOT exonerate memory — 30s samples miss transient
   allocs, and check largest-free-block, not just free.
4. `logger: level: ERROR` (common on these builds) **suppresses the LVGL warnings that name
   the failing component** — raise it while hunting, and keep a reconnecting log capture
   (`esphome logs` in a loop) because ESPHome retains nothing.

## Quick-reference traps

| Trap | Rule |
|---|---|
| Partition table | OTA writes only the app slot. Table changes need a WIRED flash of `firmware.factory.bin`. Docker Desktop on macOS cannot pass USB — build on Linux, esptool from the Mac. |
| MDI glyphs | Verify by **rendering and looking**: meta.json codepoints drift vs pinned webfont (one "package" codepoint renders as an OWL), and ink-tests prove presence, not identity. Local TTF cache: `.esphome/font/<hash>/font.ttf`. |
| First `api: actions:` | Link error `undefined reference get_execute_arg_value<StringRef>` = stale CMake glob (user_services.cpp newly required, never compiled). `esphome clean`, don't blame upstream. |
| strftime | `%-I` (glibc extension) fails on esp-idf newlib and ESPHome renders the literal string **"ERROR"** on the panel. Hand-roll with snprintf. |
| `online_image` | Retains the previous download — on fetch error, show a placeholder, never the buffer (a doorbell would show the LAST visitor's face). Resize SERVER-side (frame decode blocks the main loop ~0.5s/23KB). Frigate honours only `?height=`; `h`/`w`/`size` are silently ignored. |
| LVGL arcs | Free-position segments via `lv_arc_set_bg_angles` (background arc); colour lives on `LV_PART_MAIN`. A 3-4° background arc drawn wider than a track = a tick at any angle. Full-circle value arc: `start_angle: 270, end_angle: 269`. |
| Runtime re-align | Widgets placed with `align:` must move via `lv_obj_align(obj, LV_ALIGN_..., x, y)`, not `lv_obj_set_x`. |
| Edge-trigger alerts | Latch the condition (alert on transition only) + **boot-guard** (`millis() < 120000` seeds the latch silently) — rebooting mid-storm is not news. Remember the guard when testing: inject after 2.5 min. |
| HA trigger-templates | Top-level `variables:` render BEFORE `actions:` → any `response_variable` is undefined there. Move `variables:` inside `actions:`. |
| HA `variables:` typing | Values round-trip text→literal_eval. Dicts holding enum objects (e.g. Music Assistant results) silently degrade to STRINGS → `.uri` renders empty. Only scalars survive; read `res.tracks[0].uri` inline at the call site. |
| Pirate Weather AQI | `airQualityIndex`/`smoke` exist only with **`?version=2`**; the HA integration exposes neither. `fireIndex` is fire danger, not smoke. |
| ESPHome CLI in Docker | Compile/logs containers probe the dashboard healthcheck → always "unhealthy"; it's a false positive. Builds leave root-owned files in `.esphome/`. |
| Blocking `docker run` in a service | bash can't act on SIGTERM while blocked → systemd SIGKILLs and **orphans the container** (a 2nd log attach corrupts the capture). Fixed `--name` + `ExecStop=docker rm -f <name>`. |
| Rounded arc caps bridge gaps | LVGL arcs default to ROUNDED end caps; at large radius the caps visually bridge small inter-segment gaps and a segmented ring reads as solid. `arc_rounded: false` — also skips endcap anti-aliasing. |
| Feels laggy | Check compiler first: ESPHome esp-idf defaults to `compiler_optimization: SIZE`; `PERF` (-O2) bought this device's LVGL loop back 20% for 2.5% flash. Then shorten page-slide animations — full-page redraws per frame dominate perceived lag. |
| One assignable button | On this board BOOT (GPIO0) is a free runtime input (click/double/long via `on_multi_click`); the other button is RESET/power — firmware never sees it. Design for one button, not two. |
| MA transition frames LIE | Music Assistant flaps both state AND attributes mid-pause: frames arrive with `media_title` momentarily EMPTY (observed live 3x). Any instantaneous "revert on empty/idle" logic wipes the now-playing screen mid-song. Labels accept only NON-EMPTY updates; the only revert is a slow stale-out interval (hours — 36h here), never a transition frame. |
| Native play no-ops on MA-owned players | With Music Assistant owning the speaker's queue, NATIVE `media_play` silently no-ops (executes, state never moves — 0-for-2 live) while MA-entity play resumes every time. Drive the MA entity in BOTH directions via one HA script (`script.smart_puck_playpause`) and point every firmware tap at the script — never at `media_player.media_play_pause` on the native entity. |
| Message-body fonts tofu | A font that renders ARBITRARY text (phone mirrors, agent messages) needs full printable ASCII **plus** the typographic set — em/en dash, curly quotes, ellipsis. A curated glyph list boxes on the first "worth a glance — amber". And sanitize HA-side: `regex_replace` away anything beyond the font's coverage (emoji!) before sending, keeping the char class in sync with the firmware glyph list. |
| HISTORICAL — Sonos ANNOUNCE is invisible to state | `announce: true` clip playback NEVER appears in `media_player` state (verified live: state stayed `idle` mid-clip) — a `wait_for from: playing` dead-waits its full timeout. Still true, but OBSOLETED here: the converse script now uses `assist_satellite.start_conversation` (see below), which needs no announce and no wait at all. Only relevant if you have no assist_satellite and must fall back to the announce chain. |
| HISTORICAL — length-based reply delay | The old announce→listen chain timed the reply window off message LENGTH (~13 chars/sec + cushion, clamped) because announce playback is invisible to state. Two failure modes even when tuned: the window can open on the announce TAIL (the mic transcribes your own speaker), and it's guesswork per TTS voice. OBSOLETED by `start_conversation(start_message)` — native speak-then-listen, zero timing math. |
| `start_conversation` = native speak-then-listen | `assist_satellite.start_conversation` with `start_message` speaks AND opens the reply window in ONE call on an Assist satellite — it replaces entire announce+delay+listen chains. `preannounce: true` chimes first. Guard on the satellite not being `unavailable`/`unknown` (else it silently no-ops) and fall back to a phone push. |
| Pipelines answer where they heard | HA Assist pipelines respond ON THE SATELLITE THAT HEARD YOU. With multiple mics in a room (display device + voice satellite), decide the ears / voice / face roles DELIBERATELY — otherwise which speaker answers depends on which mic won the race. |
| Two open mics = double capture | Two devices listening on the same wake word (or one wake mic + one open reply window) = ONE utterance captured TWICE = duplicate pipeline runs, duplicate intent execution. Hand wake duty to exactly one device (see the ears-handoff switch) and never leave a second reply window open across another device's capture. |
| Agent "can't answer" = exposure, usually | LLM conversation agents only see entities EXPOSED to Assist (Settings → Voice assistants → Expose). "The agent can't answer X" is usually an exposure gap, not model intelligence — check exposure before blaming (or swapping) the model. |
| STT quality gates everything | The STT model gates the whole voice UX: base/int8 Whisper shreds conversational speech (a fine wake-word demo becomes an unusable conversation partner). Prefer `distil-small.en` or better for the pipeline you actually talk to; spend the accuracy budget on STT before anything else. |
| New modal script joins EVERY dismiss list | Every dismiss site (tap-on-modal, `hard_dismiss`, the conversing close) must `script.stop` EVERY modal script. A new modal kind added without joining those lists keeps its pending dwell alive — it fires later and yanks the user off a page they navigated to themselves. |
| Conversational silence ≠ error | `voice_assistant.on_error` fires on dead air. In a SYSTEM-initiated conversation ("anything else?" … silence) that silence is the ANSWER, not a failure — gate the error path on a `conversing` flag and close gracefully (dismiss the modal, return to the page) instead of "Sorry — try again". |
| Goodbye phrases outrank the fallback | Register close-phrases ("no thanks", "that's all", "bye") as LITERAL `conversation` triggers — custom sentences outrank the fallback conversation agent, so a goodbye is acknowledged in milliseconds instead of round-tripping an LLM while the modal lingers. |

## System-initiated conversation (the converse primitive)

The device can *start* a conversation, not just answer one. The whole pattern is three
generic parts, all policy in HA:

1. **`script.smart_puck_converse`** (in `ha/smart_puck_package.yaml`) — show the text on
   the puck (the face), then ONE native call does the spoken half:
   `assist_satellite.start_conversation` with `start_message` + `preannounce` speaks on an
   Assist satellite and opens its reply window in the same step. Routed by where the
   *person* is: speech only when they're home and the room is occupied; satellite
   unavailable → phone push (a push can't listen).
2. **`converse_listen` action** — opens THIS device's mic with NO wake word. The fallback
   listen-leg if you have no assist_satellite entity (pair it with the historical
   announce+delay chain in the table), or the natural choice when the puck itself holds
   wake-word duty. Firmware sets a `conversing` flag so the silence path closes gracefully.
3. **Goodbye automation** (`ha/automations/conversation_goodbye.yaml`) — exact close-phrases
   end the exchange like a conversation: spoken farewell, brief modal, `dismiss`.

## Ears handoff (multi-satellite rooms)

When a dedicated voice satellite (e.g. an HA Voice PE) joins the room, hand it wake-word
duty instead of running two open mics: the firmware's HA-exposed **"Wake word" template
switch** (`RESTORE_DEFAULT_OFF`) gates EVERY `micro_wake_word.start` site through one
`arm_wake_word` script — boot, both voice re-arm paths, tap-to-stop, AND the self-heal
watchdog (which is also condition-gated, so ears-off doesn't log a "re-arming" line every
interval). Push-to-talk and `converse_listen` bypass the gate by design — the display
device keeps its button mic. Revert = flip one switch in HA, no reflash. Decide the roles
explicitly: satellite = ears + voice, display device = face + push-to-talk.

**Ritual concept:** any presence edge can open a conversation. Example shape — a
once-per-day greeting: trigger on the room's occupancy going `off → on`, condition on the
person actually being home (don't trust the room sensor alone — a laptop or a latch can
fool it), guard once-a-day with `this.attributes.last_triggered`, then call the converse
script with whatever your agent composes. All schedule/content policy stays in HA; the
firmware only knows how to speak and listen.
