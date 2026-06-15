# Espresso machine — ESPHome packages

Reusable [ESPHome](https://esphome.io) packages for an ESP32-based espresso
machine. Two independent packages you can include separately or together:

| Package | File | What it gives you |
|---------|------|-------------------|
| **Boiler controller** | [`packages/espresso_controller.yaml`](packages/espresso_controller.yaml) | model-predictive boiler control + on-device auto-tuning (no manual PID) |
| **Pump / brew** | [`packages/pump_controller.yaml`](packages/pump_controller.yaml) | pump power ladder + pre-infusion/brew sequence and HA controls |

Each machine is a thin device file that supplies only the hardware and its tuned
values, then includes the package(s) it needs. Machines without a pump (e.g. a
bare test rig) just include the controller; a full machine includes both.

---

## Boiler controller

Replaces the stock PID with an explicit two-state thermal model, so the boiler
reaches the setpoint **without overshoot** and holds it tightly — and it measures
its own physical parameters, so there is no manual tuning.

- **Model-predictive throttle.** A two-state model (heating coil `Ts` → boiler `Tb`)
  predicts "if I cut power now, where does the peak land" and throttles so the peak
  hits the setpoint. Smooth approach from below, no bang-bang, no overshoot.
- **Sensor-lag compensation** via a lookahead, plus a rate-driven momentum offset
  that vanishes in hold by construction.
- **Overnight auto-tune** (one button). Runs 3 heat/coast cycles and a verification
  heatup, fits the full cycle curve on-device (P, `tau_coil`, lookahead via a
  pole-split, `tau_cool` from the cooling phases, `off_tau_lag` from the verification),
  and writes the results to the HA number entities.
- **Daily adaptive tune** (a switch). Watches the first cold (morning) heatup and
  gently blends the freshly fitted parameters in (30 % EMA) — tracks drift (e.g.
  mains voltage) with no intervention.
- **Diagnostics:** projected peak, coil heat in transit, ETA to setpoint, real
  output %, power/energy, and loop-time / heap headroom.

**Contract — the device file must provide, with these exact IDs:**

| Component | id | Notes |
|-----------|----|-------|
| an `output` | `heater` | the SSR driver — `slow_pwm` (zero-cross SSR) or `ledc` (1 Hz) |
| a `sensor` | `boiler_temperature` | an `ntc` sensor + its source chain (resistance + ADC/ADS1115) |

Everything else (the tunable `number:` entities, the climate shell, the tune
state machine, diagnostics) comes from the package.

### Substitutions

All optional — the package ships sensible defaults; override per machine. The
plant parameters are first-boot defaults only; the auto-tune writes the live
values and `restore_value` keeps them across reboots.

| Substitution | Default | Meaning |
|--------------|---------|---------|
| `base_power_w` | `0.7` | idle (controller/electronics) draw, W |
| `device_power_w` | `1200` | heater power, W |
| `max_safe_temp_c` | `110.0` | absolute emergency-stop temperature, °C |
| `default_heating_power` | `0.543` | P — heating power, °C/s |
| `default_coil_tau` | `48.86` | `tau_coil` — coil→boiler transfer constant, s |
| `default_lookahead_s` | `2.5` | sensor lag / projection lookahead, s |
| `default_off_tau_lag` | `20.0` | momentum-offset time constant, s |
| `default_off_cap` | `2.5` | momentum-offset clamp, °C |
| `default_tau_cool` | `7751.0` | boiler free-cooling constant, s |
| `tune_heat_timeout_s` | `300` | auto-tune heat-phase timeout, s |
| `tune_coast_timeout_s` | `600` | auto-tune coast-phase timeout, s |
| `adapt_observe_timeout_s` | `1200` | daily-adaptive observation timeout, s |

> The default plant values are for a typical full-size boiler. A small/fast boiler
> (e.g. a thermoblock test rig) is closer to P ≈ 0.7, `tau_coil` ≈ 9–18,
> `tau_cool` ≈ 1900–3100 — but just run the auto-tune and let it measure.

### Home Assistant entities

- **Climate:** `Boiler controller` (setpoint, HEAT/OFF).
- **Numbers (CONFIG):** the parameters above — editable, kept across reboots.
- **Button:** `Auto-tune (full, overnight)` (multi-hour; ends holding the setpoint),
  `Auto-tune ABORT`.
- **Switch:** `Adaptive tune (daily)` — the gentle morning self-correction.
- **Sensors:** boiler temperature, output %, projected peak, coil heat in transit,
  time to setpoint, power, daily energy, loop time, heap free, RSSI.
- **Text:** `Auto-tune status`, `Control mode` (HEATUP / HOLD / AUTO-TUNE).

### Tuning workflow

1. Flash with starting-estimate substitutions (the defaults are fine).
2. Press **Auto-tune (full, overnight)** with a cold machine; let it finish.
3. (Optional) Turn on **Adaptive tune (daily)** to track drift automatically.
4. Watch the first cold heatup: it should reach the setpoint without overshoot.

---

## Pump / brew controller

Drives a pump SSR with a smooth power ladder (each step lands on a whole number of
mains half-cycles, for a zero-cross SSR) and runs a pre-infusion → brew sequence.

- **`Pump power` (number)** — set 0–100 %; snapped to the ladder and applied live.
  The script publishes the *snapped* value back, so every change is in HA history.
- **Pre-infusion → brew** — pre-infuse at a low power for a set time, then full
  power for the brew time, then stop.
- Driven from HA (buttons), from physical buttons (your device's GPIO
  `binary_sensor`s call the scripts), or directly via the `Pump power` slider.

**Contract — the device file must provide, with this exact ID:**

| Component | id | Notes |
|-----------|----|-------|
| an `output` | `pump_ssr` | a `ledc` output on the pump SSR pin (the script sets its frequency + duty) |

**Provides:** scripts `set_pump_pct` (pct → ladder → SSR) and `preinfusion_brew`
(callable from your binary_sensors); numbers `Pump power`, `Pre-infusion power`,
`Pre-infusion duration`, `Brew duration`; buttons `Start brew (with pre-infusion)`
and `Stop brew`. (Assumes 50 Hz mains; the ladder is in the package.)

---

## Usage

Reference the packages straight from GitHub — no need to copy them locally. Pin a
tag for reproducibility (a push to `main` could otherwise change your next build).

### Heater only (no pump) — e.g. a test rig

```yaml
substitutions:
  # this machine's tuned values (overwritten at runtime by the auto-tune)
  base_power_w: "0.7"
  device_power_w: "1200"
  default_heating_power: "0.78"
  default_coil_tau: "18.0"
  default_lookahead_s: "9.0"
  default_off_tau_lag: "10.7"
  default_tau_cool: "1908.0"

packages:
  controller: github://gyimesibalazs/iespresso/packages/espresso_controller.yaml@v1.2

esphome:
  name: my-rig
esp32:
  board: esp32dev
  framework: { type: arduino }

# --- the two things the controller package needs ---
output:
  - platform: ledc          # or slow_pwm for a zero-cross SSR
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

### Heater + pump — a full machine

Add the pump package, a `pump_ssr` output, and (optionally) physical buttons that
call the pump scripts:

```yaml
packages:
  controller: github://gyimesibalazs/iespresso/packages/espresso_controller.yaml@v1.2
  pump:       github://gyimesibalazs/iespresso/packages/pump_controller.yaml@v1.2

output:
  - platform: slow_pwm        # heater SSR
    period: 1s
    id: heater
    pin: 14
  - platform: ledc            # pump SSR — required by the pump package
    pin: 19
    channel: 1
    frequency: 50Hz
    id: pump_ssr

# Optional: physical buttons wired to GPIOs, calling the pump scripts
binary_sensor:
  - platform: gpio            # momentary jog button: full power while pressed
    id: pump_button
    pin: { number: 32, inverted: true, mode: { input: true, pullup: true } }
    on_state:
      then:
        - script.stop: preinfusion_brew
        - script.execute: { id: set_pump_pct, pct: !lambda 'return x ? 100 : 0;' }
  - platform: gpio            # brew switch: start pre-infusion/brew on press
    id: main_switch
    pin: { number: 25, inverted: true, mode: { input: true, pullup: true } }
    on_press:
      then: { script.execute: preinfusion_brew }
    on_release:
      then:
        - script.stop: preinfusion_brew
        - script.execute: { id: set_pump_pct, pct: 0 }
```

A complete, working device example (ESP32 + ADS1115 NTC + heater + pump) is in
[`example-machine.yaml`](example-machine.yaml).

> **Versions.** The controller exists from `v1.0`; the pump package was added in
> `v1.1`, and `v1.2` makes the pump publish its snapped level to HA history. Use
> `@v1.2` for both, or pin a later tag as the repo evolves.

### Local include

If you'd rather vendor a package, drop the file under `packages/` next to your
device config and use `controller: !include packages/espresso_controller.yaml`.

## License

MIT — see [`LICENSE`](LICENSE).
