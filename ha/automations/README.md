# HA-side automations for the Smart Puck

**HA is the source of truth** — these are exported snapshots for review/history, not a deploy
target. Edit in HA (or via `ha_config_set_automation`), then re-export here.

| Automation | Purpose |
|---|---|
| `voice_automations.yaml` | Voice: play-music handler + hard-wired media controls |
| `doorbell_porthole.yaml` | Doorbell press → puck shows a Frigate frame of the door |
| `alert_modal.yaml` | Any notification sent to your phone → puck interrupt modal |

## Design note: the puck is the passive side

Both modals are driven by HA calling an ESPHome **action** (`esphome.smart_puck_show_*`).
Policy — *what is worth interrupting the user for* — lives in HA, where he can see and change it.
The puck only renders. Rewiring a trigger must never require an OTA.
