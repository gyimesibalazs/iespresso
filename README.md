# Espresso boiler controller — ESPHome package

A reusable [ESPHome](https://esphome.io) package that turns an ESP32 + SSR + NTC
into a **model‑predictive espresso boiler controller** with on‑device auto‑tuning.
It replaces the stock PID with an explicit two‑state thermal model, so the boiler
reaches the setpoint **without overshoot** and holds it tightly — and it measures
its own physical parameters, so there is no manual PID tuning.

The controller lives entirely in [`packages/espresso_controller.yaml`](packages/espresso_controller.yaml).
Each machine is a thin device file that supplies only the hardware and its tuned
values, then includes this package.

## What it does

- **Model‑predictive throttle.** A two‑state model (heating coil `Ts` → boiler `Tb`)
  predicts "if I cut power now, where does the peak land" and throttles so the peak
  hits the setpoint. Smooth approach from below, no bang‑bang, no overshoot.
- **Sensor‑lag compensation** via a lookahead, plus a rate‑driven momentum offset
  that vanishes in hold by construction.
- **Overnight auto‑tune** (one button). Runs 3 heat/coast cycles and a verification
  heatup, fits the full cycle curve on‑device (P, `tau_coil`, lookahead via a
  pole‑split, `tau_cool` from the cooling phases, `off_tau_lag` from the verification),
  and writes the results to the HA number entities.
- **Daily adaptive tune** (a switch). Watches the first cold (morning) heatup and
  gently blends the freshly fitted parameters in (30 % EMA) — tracks drift (e.g.
  mains voltage) with no intervention.
- **Diagnostics:** projected peak, coil heat in transit, ETA to setpoint, real
  output %, power/energy, and loop‑time / heap headroom.

## Requirements (the device file's contract)

The including device config must provide, with these exact IDs:

| Component | id | Notes |
|-----------|----|-------|
| an `output` | `heater` | the SSR driver — `slow_pwm` (zero‑cross SSR) or `ledc` (1 Hz) |
| a `sensor` | `boiler_temperature` | an `ntc` sensor + its source chain (resistance + ADC/ADS1115) |

Everything else (the tunable `number:` entities, the climate shell, the tune
state machine, diagnostics) comes from the package.

## Usage

Reference the package straight from GitHub — no need to copy it locally:

```yaml
substitutions:
  # this machine's tuned values (overwritten at runtime by the auto‑tune)
  base_power_w: "1.7"
  device_power_w: "1450"
  default_heating_power: "0.700"
  default_coil_tau: "9.0"
  default_lookahead_s: "9.0"
  default_off_tau_lag: "11.9"
  default_tau_cool: "3100.0"

packages:
  controller: github://gyimesibalazs/iespresso/packages/espresso_controller.yaml@main

esphome:
  name: my-machine
esp32:
  board: esp32dev
  framework: { type: arduino }

# --- the two things the package needs ---
output:
  - platform: ledc          # or slow_pwm for a zero‑cross SSR
    frequency: 1Hz
    channel: 0
    id: heater
    pin: 27

sensor:
  - platform: ntc
    sensor: resistance_sensor
    calibration: { b_constant: 3950, reference_temperature: 25°C, reference_resistance: 10kOhm }
    id: boiler_temperature
    name: Boiler Temperature
  # ...resistance + ADC/ADS1115 chain feeding `resistance_sensor`

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
api:
  encryption: { key: !secret api_key }
ota:
  platform: esphome
  password: !secret ota_password
```

A complete, working device example is in [`example-machine.yaml`](example-machine.yaml).

> **Pin the ref.** `@main` re‑fetches per `refresh` (default 1 day) and a push can
> change your next firmware build. For reproducibility point at a tag or commit:
> `...espresso_controller.yaml@v1.0`.

### Local include

If you'd rather vendor it, drop the file under `packages/` next to your device
config and use `controller: !include packages/espresso_controller.yaml`.

## Substitutions

All are optional — the package ships sensible defaults; override per machine.
The plant parameters are first‑boot defaults only; the auto‑tune writes the live
values and `restore_value` keeps them across reboots.

| Substitution | Default | Meaning |
|--------------|---------|---------|
| `base_power_w` | `0.7` | idle (controller/electronics) draw, W |
| `device_power_w` | `1200` | heater power, W |
| `max_safe_temp_c` | `110.0` | absolute emergency‑stop temperature, °C |
| `default_heating_power` | `0.543` | P — heating power, °C/s |
| `default_coil_tau` | `48.86` | `tau_coil` — coil→boiler transfer constant, s |
| `default_lookahead_s` | `2.5` | sensor lag / projection lookahead, s |
| `default_off_tau_lag` | `20.0` | momentum‑offset time constant, s |
| `default_off_cap` | `2.5` | momentum‑offset clamp, °C |
| `default_tau_cool` | `7751.0` | boiler free‑cooling constant, s |
| `tune_heat_timeout_s` | `300` | auto‑tune heat‑phase timeout, s |
| `tune_coast_timeout_s` | `600` | auto‑tune coast‑phase timeout, s |
| `adapt_observe_timeout_s` | `1200` | daily‑adaptive observation timeout, s |

> The default plant values are for a typical full‑size boiler. A small/fast boiler
> (e.g. a thermoblock test rig) is closer to P ≈ 0.7, `tau_coil` ≈ 9–18,
> `tau_cool` ≈ 1900–3100 — but just run the auto‑tune and let it measure.

## Home Assistant entities

- **Climate:** `Boiler controller` (setpoint, HEAT/OFF).
- **Numbers (CONFIG):** the parameters above — editable, kept across reboots.
- **Button:** `Auto-tune (full, overnight)` — runs the full identification (multi‑hour;
  the machine ends holding the setpoint). `Auto-tune ABORT` stops it.
- **Switch:** `Adaptive tune (daily)` — enable the gentle morning self‑correction.
- **Sensors:** boiler temperature, output %, projected peak, coil heat in transit,
  time to setpoint, power, daily energy, loop time, heap free, RSSI.
- **Text:** `Auto-tune status`, `Control mode` (HEATUP / HOLD / AUTO‑TUNE).

## Tuning workflow

1. Flash with starting‑estimate substitutions (the defaults are fine).
2. Press **Auto-tune (full, overnight)** with a cold machine; let it finish.
   It writes P, `tau_coil`, lookahead, `tau_cool`, `off_tau_lag`.
3. (Optional) Turn on **Adaptive tune (daily)** to track drift automatically.
4. Watch the first cold heatup: it should reach the setpoint without overshoot.

## License

MIT — see [`LICENSE`](LICENSE).
