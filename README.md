# ESPHome Devices

ESPHome device configurations for my home automation setup, integrated with Home Assistant.

All devices share the same base setup: ESP32 (esp-idf framework), encrypted Home Assistant native API, OTA updates, a local web server on port 80, and a WiFi fallback hotspot (`<device> Fallback Hotspot`) that comes up if the configured network can't be reached. A "Heartbeat" button blinks the onboard status LED twice, mainly to visually confirm a device is reachable/responsive. Diagnostic WiFi signal sensors are disabled by default in Home Assistant (can be enabled per-device if needed).

## Devices

| Folder | Device | Description |
|---|---|---|
| `garage-door/` | ESP Garage Door | Garage door position (open/closed via two limit reed switches) and control relay |
| `mailbox/` | ESP Mailbox | Detects mail deliveries via slot/flap sensors, ambient light on delivery |
| `water-meter/` | ESP Water Meter | Pulse-counting water flow meter with persistent total consumption |
| `radiator-fan/` | ESP Kitchen Radiator Fan | Temperature-controlled PWM fan boosting a kitchen radiator |
| `irrigation/` | ESP Irrigation | Garden sprinkler controller (5 valves + pump) via the `sprinkler` component |

---

## garage-door

Tracks the garage door position via two limit reed switches and drives the door through a single relay wired in parallel to the physical garage opener's button (the opener only has one toggle input, not separate open/close/stop inputs).

**Wiring**
- `GPIO18` – reed switch at the *closed* end position (`door_closed`)
- `GPIO21` – reed switch at the *open* end position (`door_open`)
- `GPIO19` – relay output that pulses the opener's button input (`door_drive`)
- `GPIO04` – status LED

**Logic**
- `door_closed` / `door_open` are debounced with a `delayed_on_off` filter (2s/1s and 4s/2s) since the reed switches can bounce while the door is moving through the end positions.
- Two template binary sensors, `Door Opening` and `Door Closing`, track the movement direction while the door is between both end positions: leaving the *closed* position sets `Door Opening`, leaving the *open* position sets `Door Closing`. Reaching either end position clears both flags again.
- The `cover` entity only reports `open`/`closed` (via `door_closed`), it does **not** feed `Door Opening`/`Door Closing` into the cover's `current_operation` — ESPHome's template cover only supports `COVER_OPEN`/`COVER_CLOSED` in its state lambda; showing a live "opening/closing" animation in Home Assistant would require calling `cover.template.publish` with `current_operation` separately. Left as-is for now (not tested).
- `open_action`, `close_action` and `stop_action` all trigger the exact same relay pulse (`door_drive`) — this is intentional, matching the opener's single-button toggle behavior (press = start/stop/reverse depending on current state), not a bug.
- The `Trigger Door` switch is intentionally exposed in Home Assistant in addition to the cover entity, as a direct manual control path.

---

## mailbox

Detects mail deliveries using two sensors: a `slot` sensor that triggers when something passes through the mail slot, and a `flap` sensor on the box's access door (used to retrieve mail).

**Wiring**
- `GPIO23` – slot sensor (`slot`)
- `GPIO18` – access flap sensor (`flap`)
- `GPIO13` – PWM output driving the ambient light (`ambient_light`, controlled manually from Home Assistant, no automation triggers it in this config)
- `GPIO04` – status LED

**Logic**
- `Mail Items` (`dropped`) is a counter, incremented once a delivery is detected. Because emptying the box (opening the flap) mechanically also triggers the slot sensor, a `slot_open_script` waits 10 seconds after the slot triggers before incrementing — using `mode: restart` so repeated slot triggers (e.g. several letters delivered together) only count as **one delivery**, not one per letter. This is intentional: the counter tracks *deliveries*, not individual pieces of mail.
- Opening the flap immediately sets `Flap Opened` and resets `Mail Items` to 0 (you're retrieving the mail). `Flap Opened` stays `true` for 10 seconds after the flap is closed again, so that the slot sensor bump caused by emptying the box doesn't get mistaken for a new delivery.
- `Mail Delivered` (`loaded`) is derived from `Mail Items > 0` and switches the status LED on/off accordingly.
- The `dropped` counter is **not** persisted across a reboot/power loss (no flash restore) — intentional, an inaccurate count after a power outage is not considered a problem.
- A manual `Reset` button is available to zero the counter directly.

---

## water-meter

Counts water flow pulses (1 pulse = 1 liter) and reports both a flow rate and a persistent cumulative total.

**Wiring**
- `GPIO21` – pulse input from the flow sensor (`water_meter`)
- `GPIO04` – status LED

**Logic**
- `Flow Rate` (l/min) is the `pulse_meter` platform's live rate, hardware-debounced (`internal_filter: 10ms`) plus a software `debounce: 100ms` filter, with a 60s timeout that resets the rate to 0 once flow stops.
- `Consumption` (m³) is the pulse meter's `total` sub-sensor. Its filter lambda mirrors every new raw pulse count into the `flash_total_pulses` global, which is flash-persisted (`flash_write_interval: 5min`) and restored on boot via `pulse_meter.set_total_pulses`, so the cumulative reading survives a reboot.
- A custom Home Assistant API action, `set_total`, lets you recalibrate the counter to match the physical meter's dial. Calling it also updates `flash_total_pulses` correctly, because `set_total_pulses` internally publishes the new state through the same filter pipeline as a real pulse — verified against the ESPHome source, not just assumed.
- Because flash writes only happen every 5 minutes, up to ~5 minutes of consumption can theoretically be lost on a power failure between writes. Accepted trade-off (reduces flash wear), not considered a problem.

---

## radiator-fan

Boosts a kitchen radiator with a PWM-controlled fan, adjusting fan speed based on the radiator's flow temperature between a configurable min/max range, with manual and turbo override modes.

**Wiring**
- `GPIO16` – 1-Wire bus for the Dallas temperature sensor (`tempreal`, flow temperature)
- `GPIO17` – PWM output driving the fan (`fanpwm`, 25kHz)
- `GPIO04` – status LED

**Logic**
- `Fan Speed` (`fanpower`) is computed by a lambda with this priority:
  1. If `Auto` (`mode_auto`) is off → use the `Manual` (`manualpwr`) percentage directly.
  2. Else if `Turbo` (`mode_turbo`) is on → force `max_out` (98%, the PWM value above which the fan is effectively at full speed).
  3. Else, linearly interpolate fan speed between `min_out` (10%) and `max_out` based on where the current flow temperature sits between `Min Temp.` and `Max Temp.` — below `Min Temp.` the fan is off, above `Max Temp.` it's at `max_out`.
- `Min Temp.` (20–40 °C) and `Max Temp.` (41–60 °C) have non-overlapping ranges on purpose, so `Max Temp.` can never be ≤ `Min Temp.` — this prevents a division-by-zero in the interpolation formula above.
- Both set-points are persisted across reboots via `flash_tempmin`/`flash_tempmax` globals (`flash_write_interval: 5min`) and restored on boot by publishing their saved value back into the corresponding number entity.
- `Auto` restores its last state on boot (defaults to on if never set); `Turbo` always starts off after a reboot — intentional, so the fan doesn't unexpectedly blast at full speed after a power cycle.
- The status LED lights up whenever the fan is running at/above `max_out`, or while the temperature reading is unavailable (`NaN`, e.g. right after boot before the first sensor reading arrives).

---

## irrigation

Controls a 5-zone garden sprinkler system with a shared pump, using ESPHome's `sprinkler` component for scheduling/sequencing.

**Wiring**
- `GPIO21`–`GPIO26` (`21, 22, 23, 25, 26`) – valve relays (`Valve 1`–`Valve 5`), active-low
- `GPIO27` – pump relay (`master`), active-low
- `GPIO13` – an internal-only "Option LED" output (not currently used by any automation in this config)
- `GPIO04` – status LED

**Logic**
- The `sprinkler` component (`sprinkler_ctrlr`) exposes one HA-visible zone per valve, each with its own enable switch and a configurable run-duration number: `Mountain` (valve1), `Terrace` (valve3), `North` (valve2), `Stairs` (valve5), `Plants` (valve4). It sequences zones automatically, including a 2s valve/pump overlap and 2s start/stop delays so the pump isn't switched under a closed valve.
- Independently of the `sprinkler` component's own scheduling, every valve switch has a hardcoded 1-hour auto-off, and the pump (`master`) has a hardcoded 5-hour auto-off. This is a hardware-level failsafe: even if the `sprinkler` component's logic misbehaves, no valve or the pump can stay on indefinitely.
- The pump switch (`master`) is intentionally exposed in Home Assistant (not `internal: true` like the individual valves), allowing it to be triggered directly outside of the sprinkler sequencing if needed.

---

## Setup

1. Install [ESPHome](https://esphome.io/).
2. Copy `secrets.yaml.example` to `secrets.yaml` and fill in your own WiFi credentials, API encryption keys, OTA passwords and fallback hotspot passwords.
   - Generate an API encryption key with `esphome generate-api-key` (or use the ESPHome dashboard's "encryption key" helper).
   - Generate an OTA/hotspot password with any password generator, or e.g. `openssl rand -hex 16`.
3. `secrets.yaml` is gitignored and must never be committed - it holds real credentials.
4. Flash/update a device with:
   ```
   esphome run <folder>/<file>.yaml
   ```

## Notes

- Each device's `esphome.name` matches its YAML filename.
- `friendly_name` values and all entity names are kept in English for consistency.
- This repository does not include Home Assistant automations/dashboards - only the ESPHome firmware configs.

## License

MIT - see [LICENSE](LICENSE).
