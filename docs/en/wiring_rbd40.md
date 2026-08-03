# Wiring — ESP32 + RobotDyn AC dimmer 40A "with current sensor"

> ⚠️ This is a **230 V** installation. Isolate the circuit, follow local
> regulations, and have the final connection checked by a qualified electrician.
> The mains side of the dimmer and the boiler are live — never touch them
> energized.

## Module used

RobotDyn-style triac dimmer, 1 channel, **40 A**, premium variant "with
current sensor" (heatsink NTC + 5 V fan + built-in CT current sensor). It
includes:

* **Isolated mains side**: `AC-IN` / `AC-OUT` (connected in series with the load)
* **Zero-cross detector**: `Z-C` output (a pulse at each mains zero-crossing)
* **Triac gate input**: `DIM` — the phase-control ("firing") signal
* **Control side**: `VCC` (3.3 V logic — the board supports 3.3/5 V), `GND`
* **Heatsink NTC**: `TEMP` analog output (heatsink temperature)
* **Cooling fan**: `5V` + `FAN` (speed control) + 2-pin `FAN CON` (fan wires)
* **Current sensor**: `CUR` analog output (load current, CT) + a small
  "current precision" trimmer to calibrate its gain

### Low-voltage side pinout (as marked on the board)

Group of 8 pins:

| `TEMP` | `FAN` | `CUR` | `5V` |
|---|---|---|---|
| `VCC` | `GND` | `Z-C` | `DIM` |

plus:

* a screwdriver trimmer labeled **"current precision"** — CT gain calibration
  (set on the bench, see `bench_test.md`)
* a 2-pin connector labeled **`FAN CON`** — the fan itself (power from the
  board's 5 V rail)

### Connections (ESP32)

| Dimmer pin | ESP32 pin (example) | ESPHome role |
|---|---|---|
| `Z-C` | `GPIO23` | `regulator_zero_crossing_pin` (interrupt input) |
| `DIM` | `GPIO17` | `regulator_gate_pin` (triac gate) |
| `VCC` | `3V3` | module logic + sensors supply — **use 3.3 V** so `TEMP`/`CUR` stay within ADC range |
| `GND` | `GND` | common ground, isolated side |
| `TEMP` | `GPIO34` | `dimmer_temp_pin` (ADC input, heatsink NTC) |
| `CUR` | `GPIO35` | `dimmer_current_pin` (ADC input, CT) |
| `5V` | `5V` | 5 V rail for the fan section |
| `FAN` | tied to `5V` | **hardwired always-on cooling** (see below) |
| `FAN CON` | — | fan connector (already wired to the board) |

> **Fan at 100 % all the time — by design.** Tie the `FAN` pin to the `5V` pin
> so the fan runs at full speed even before the ESP firmware boots (and even if
> the ESP or the WiFi dies). A firmware-driven fan adds a failure mode (stuck
> fan controller = silent overheating); the always-on fan removes it. Noise and
> lifespan are non-issues in a boiler closet.

> **`VCC` at 3.3 V.** The `TEMP` and `CUR` outputs are dividers/CT outputs
> referenced to `VCC`; feeding the board with 5 V can push them above 3.3 V and
> damage the ESP32 ADC pins. The board officially supports 3.3 V logic.

### Power side (230 V)

| AC-IN | fixed supply (line conductor, upstream of the boiler) |
|---|---|
| AC-OUT | to the boiler element (series) |
| N | neutral straight through to the boiler (never dim the neutral) |

The boiler's own thermostat stays in the circuit and keeps protecting at
100 % opening.

## Recommended circuit

- 16 A breaker and a dedicated circuit (the boiler is a permanent wet area
  feed per NF C 15-100; a 30 mA RCD as required)
- Wire cross-section ≥ 2.5 mm² (10.4 A nominal at 2400 W, heavily derated)
- Fan 5 V rail: a small regulated 5 V supply, separate from ESP power and from
  the mains

## ESP32 pins used in the example config

| GPIO | Function |
|------|----------|
| `GPIO17` | triac gate (`regulator_gate_pin`) |
| `GPIO23` | zero-cross detector (`regulator_zero_crossing_pin`) |
| `GPIO34` | heatsink NTC (`dimmer_temp_pin`, ADC1 input-only) |
| `GPIO35` | load current CT (`dimmer_current_pin`, ADC1 input-only) |
| `GPIO18` | green LED (regulation active / diverting) |
| `GPIO19` | yellow LED (network / meter status) |
| `GPIO21` | red LED (dimmer safety active: heatsink overheat / overcurrent) |

Notes:

- Any ESP32 GPIO can sense zero-cross, but the gate needs a driving-capable pin
  (use classic GPIOs 0-33, not the input-only 34–39).
- `TEMP` and `CUR` are analog — use ADC-capable pins (ESP32 ADC1: 32–39; the
  example uses the input-only 34/35).
- Keep the Z-C wire short and away from the AC side to avoid false triggers
  from triac switching spikes; the ESPHome `ac_dimmer` component debounces
  them.
