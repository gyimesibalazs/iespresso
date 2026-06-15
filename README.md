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

### Why the power ladder (vibration-pump physics)

Espresso machines use a **vibratory (solenoid) pump** — e.g. an Ulka. An AC-driven
electromagnet pulls a piston against a spring; an internal diode lets the coil pull
on one half of each mains cycle, so the piston makes **one fixed-volume stroke per
mains cycle** (50 strokes/s on 50 Hz). It is not a motor you can dim — flow is set
by **how many strokes you allow**, not by voltage.

So power is controlled by **integral-cycle (burst) switching**: a zero-cross SSR
passes or blocks *whole* mains half-cycles, letting the pump stroke on some cycles
and skip others. Average flow ≈ the fraction of cycles passed. This is smooth and
gentle on the pump, unlike phase-angle dimming (which produces partial strokes,
noise and heat).

That is what the ladder encodes. Each level is a small fraction **M/N** of the
cycles (e.g. 1/2 = 50 %, 1/3 = 33 %, 1/6 = 17 %). To realise it exactly, the LEDC
output runs at **frequency = 50/N Hz** with **duty = M/N**, so one PWM period spans
exactly *N* half-cycles and the "on" part is exactly *M* of them — every period
delivers the same whole number of strokes, with no drift. The ladder only contains
fractions whose on- and off-times are **≥ 20 ms (one full mains cycle)**, the time
the piston needs to complete a stroke and return; finer patterns would clip strokes.

Practical consequence: **`Pump power` is a flow / stroke-rate setting, not a
pressure setpoint.** Pressure is whatever that flow builds against the puck and the
OPV. Pre-infusion at a low % = a low stroke rate = gentle low-flow wetting before
full pressure.

**Contract — the device file must provide, with this exact ID:**

| Component | id | Notes |
|-----------|----|-------|
| an `output` | `pump_ssr` | a `ledc` output on the pump SSR pin (the script sets its frequency + duty) |

**Provides:** scripts `set_pump_pct` (pct → ladder → SSR) and `preinfusion_brew`
(callable from your binary_sensors); numbers `Pump power`, `Pre-infusion power`,
`Pre-infusion duration`, `Brew duration`; buttons `Start brew (with pre-infusion)`
and `Stop brew`. (Assumes 50 Hz mains; the ladder is in the package.)

### Optional: 3-way solenoid valve

Machines whose brew group has a 3-way solenoid valve (to bleed group pressure
after the shot) can add [`packages/pump_valve.yaml`](packages/pump_valve.yaml).
It opens the valve while the pump runs (pump power > 0) and closes it when the
pump stops — driven automatically off the pump level, on change only.

Include it **alongside** the pump package and give it a GPIO pin:

```yaml
substitutions:
  valve_pin: "26"
packages:
  pump:  github://gyimesibalazs/iespresso/packages/pump_controller.yaml@v1.4
  valve: github://gyimesibalazs/iespresso/packages/pump_valve.yaml@v1.4
```

It exposes a `3-way valve` switch in HA (state follows the pump automatically).
Machines without a valve simply don't include this file — nothing else changes.

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
  controller: github://gyimesibalazs/iespresso/packages/espresso_controller.yaml@v1.4

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
  controller: github://gyimesibalazs/iespresso/packages/espresso_controller.yaml@v1.4
  pump:       github://gyimesibalazs/iespresso/packages/pump_controller.yaml@v1.4

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
> `v1.1`, `v1.2` makes the pump publish its snapped level to HA history, `v1.3` adds
> the optional 3-way valve add-on, and `v1.4` makes that valve edge-triggered. Pin
> `@v1.4` (or a later tag as the repo evolves) so a push to `main` can't change your
> next build.

### Local include

If you'd rather vendor a package, drop the file under `packages/` next to your
device config and use `controller: !include packages/espresso_controller.yaml`.

## License

MIT — see [`LICENSE`](LICENSE).
