# WALTER — control strategy

WALTER is a **solar router** for a 2400 W hot-water boiler. It is a fork of
[hacf-fr/Solar-Router-for-ESPHome](https://github.com/hacf-fr/Solar-Router-for-ESPHome)
(ESPHome / Home Assistant). It adds native support for the **Shelly EM3 Pro /
Pro 3EM** three-phase meter.

## The problem (three-phase contracts)

Residential three-phase installations spread their loads over three phases
(living spaces, kitchen, heating, cold storage, PV inverters...) which almost
never balance to zero.

In a three-phase subscription the billing engine treats **one arithmetic,
bi-directional register**: the three phase powers are added and only the
**net** value is billed. An export of −1200 W on one phase fully offsets a
+1300 W import on the other two. Reference: metering / billing rules for
three-phase connections, [Enedis (FR grid operator)](https://www.enedis.fr/media/2035/download).

Therefore the only control signal that matches billing is:

```
S_grid = P_A + P_B + P_C      (W, '+' = import, '−' = surplus)
```

This is exactly what the Shelly EM3 Pro reports as `total_act_power`, and
what WALTER zeroes:

- `S_grid > 0` → the installation is globally importing → the boiler stays off,
  even if a local "spill" exists on one phase — the phases net at the meter.
- `S_grid < 0` → net surplus after cross-phase netting → WALTER dims the boiler
  to absorb exactly that surplus, driving `S_grid → 0`.

**Phase-balance safety comes for free**: since the router only fires when the
sum is negative, it can never push any single phase over its limit — in
that situation the sum would simply not be negative.

## Diverting logic

The engine is a **progressive proportional regulator** (upstream):

```
delta = −(S_grid − Target) * reactivity / 1000      [%]
new_level = clamp(0..100, level + delta)
```

- The meter is polled once per second.
- `Target` (default 0 W) is the wanted grid exchange. Positive values keep a
  small import (useful in off-peak hours to top the tank fast); negative
  values allow that much export to spill (only makes sense with a
  feed-in-tariff contract).
- **Up/Down reactivity**: asymmetric gains are adjustable numbers — prevents
  oscillation when the boiler thermostat steps change the sum.
- Fail-safe: meter unreachable / stale → `real_power = NaN` → 0 %.

The triac opening (%) → delivered power relationship is **non-linear**
(phase-cut). It does not matter: the loop closes on the *measured* `S_grid`,
not on an open-loop power estimate, so it converges to the target whatever the
load is.

## Tariff strategy (peak / off-peak)

On typical residential peak/off-peak schedules, PV production peaks during the
most expensive hours — the ideal moment to self-consume.

- **Peak (expensive)**: WALTER is the primary way to heat water, preferably
  on PV surplus (target 0 W → the boiler absorbs the whole net surplus, up to
  its rated power).
- **Off-peak (cheap)**: two possible layouts

  1. **Classic night-contact upstream of the dimmer** (the boiler's existing
     contact): during off-peak the contact bypasses WALTER and supplies the
     boiler directly — no forced run needed. Recommended for resilience
     (boiler still heats even if WALTER/ESP is offline).
  2. **Dimmer in series only**: use WALTER's forced-run scheduler at 100 %
     between Begin and End (off-peak) to replace the old night-contact relay.
     Downside: no heating if WALTER is offline.

- With a **feed-in-tariff (vente du surplus)** contract you may prefer
  `Target ≈ −100…−500 W` (let a small spill to the grid rather than forcing the
  boiler from the grid). Without a feed-in contract keep `Target = 0` (max
  self-consumption).
- Weather-forecast pre-heating is a possible v2 (pre-heat during off-peak
  before sunny days to free up the morning sun for the rest of the house).

## Safety layers

1. **Dimmer board safety** (`dimmer_safety.yaml`): heatsink NTC thermal
   cutout — drops to 0 % at `heatsink_stop_temperature` (default 80 °C),
   resumes below `heatsink_restart_temperature` (default 60 °C); works with no
   WiFi / HA. Sensor loss (NaN) also trips the cutout (fail-safe).
2. **Overcurrent cutout**: load current ≥ `overcurrent_current` (default
   12 A) → 0 % until it falls back under the restart threshold.
3. **Health alarms** (no power cut, for information): "Triac Stuck ON"
   (current flowing while the regulator is closed), "Boiler Not Powered"
   (regulator open, no current), "Current Sensor Failure".
4. **Meter fail-safe**: no / stale data → boiler OFF.
5. **Boiler own thermostat** stays in the circuit (also protects at 100 %).
6. Bench-track before installing (see `bench_test.md`).

Note: the `dimmer_safety` package owns the shared `safety_limit` flag — the
temperature limiter packages (tank / DS18B20) are mutually exclusive with it;
pick one per configuration.

## Not in v1 (parked)

- Additional heavy loads (e.g. a second water heater) — each with its own
  router or a surplus-sharing strategy.
- Weather-forecast pre-heat and a PID-style engine.
- PV production sensor (needs a spare EM channel or the micro-inverter API) to
  compute the true pre-boiler consumption and improve energy accounting.
- Grid-limit supervision.
- Fan speed control from the heatsink temperature (the fan is hardwired
  always-on; a PWM control is possible but adds a failure mode).
- Real diverted-power metering from the CT (I_RMS × voltage) to replace the
  theoretical energy counter.

## Why fork instead of starting from scratch

- **hacf-fr/Solar-Router-for-ESPHome** — closest to the need: ESPHome, split
  packages (power meter / engine / regulator / safety / scheduler); only the
  3-phase EM3 support is missing.
- **frtz13/Zero-Surplus-Dimmer** — simplest ESP8266 loop, no safety layers.
- **x-real-ip/zero-grid** — PID + DAC + external voltage regulator, different
  hardware family; its "P_grid → 0" concept equals WALTER's target.
- **robotdyn-dimmer/ACRouter** — native ESP-IDF (not ESPHome), nice mode
  concept (OFF / AUTO / ECO / BOOST); kept as a mental model only (see the
  Home Assistant automations for the equivalent OFF / AUTO / FORCED-run modes).