# Wiring — ESP32 + RBD-340 (Dimmer 40A "fan version")

> ⚠️ This is a **230 V** installation. Isolate the circuit, follow local
> regulations, and have the final connection checked by a qualified electrician.
> The mains side of the dimmer and the boiler are live — never touch them
> energized.

## Module used

RobotDyn-style triac dimmer, 1 channel, **40 A**, with heatsink + cooling fan
(referenced as "RBD-340 / dimmer 40A fan version" or similar). It includes:

* **Isolated mains side**: `AC-IN` / `AC-OUT` (connected in series with the load)
* **Zero-cross detector**: `Z-C` output (a pulse at each mains zero-crossing)
* **Triac gate input**: `DIM` / `PWM` — the phase-control ("firing") signal
* **Control side**: `VCC` (3.3 V logic — genuine RobotDyn supports 3.3 V),
  `GND`
* **Cooling fan**: powered from a **separate 5 VDC rail** on the 40 A fan
  version; on board revisions the fan is thermally auto-managed

| Dimmer pin | ESP32 pin (example) | ESPHome role |
|---|---|---|
| `Z-C` | `GPIO23` | `regulator_zero_crossing_pin` (interrupt input) |
| `DIM` | `GPIO17` | `regulator_gate_pin` (triac gate) |
| `VCC` | `3V3` | module logic supply |
| `GND` | `GND` | common ground, isolated side |
| `5V` (fan) | `5V` | auxiliary supply of the fan section only |
| `FAN`* | `GPIO25` (optional) | fan gate on revisions that expose it |

\* the fan is usually auto-managed by the on-board thermal controller; if the
board exposes a `FAN` PWM input instead, connect it to any PWM-capable GPIO and
turn it on whenever the load switches.

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
| `GPIO18` | green LED (regulation active / diverting) |
| `GPIO19` | yellow LED (network / meter status) |
| `GPIO21` | red LED (temperature safety limit reached) |

Notes:

- Any ESP32 GPIO can sense zero-cross, but the gate needs a driving-capable pin
  (use classic GPIOs 0-33, not the input-only 34–39).
- Keep the Z-C wire short and away from the AC side to avoid false triggers from
  triac switching spikes; the ESPHome `ac_dimmer` component debounces them.