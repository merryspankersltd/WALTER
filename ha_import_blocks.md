# WALTER - Home Assistant deployment record

Deployed programmatically via HA API/WS. Entity IDs are the ACTUAL ones on this
instance (device entities have the `salon_` prefix).

## Modes (4-position selector `input_select.walter_mode`)

| Mode  | HP (peak) hours                | HC night window (20:00-06:00) | Meter polling |
|-------|-------------------------------|-------------------------------|---------------|
| OFF   | 0 %                           | 0 %                           | no            |
| SOLAR | auto routing (capped)         | max_level (100 %)             | HP only       |
| HC/HP | 0 %                           | max_level (100 %)             | no            |
| MANUAL| max_level                     | max_level                     | no            |

- One global cap: `input_number.walter_max_level` (0-100, default 100). In
  MANUAL mode it IS the dial; in HC-window modes it is the heating level; it
  also caps SOLAR auto-routing (firmware side once `router_max_level` clamp is
  flashed).
- `HC/HP` is the debug / "for some reason" classic mode (e.g. isolating a
  conflict with other line equipment). It intentionally does NOT do solar
  routing.
- **Forcing is NIGHT-ONLY** (band 20:00-06:00). Between 06:00 and 20:00
  nothing ever forces 100 %: SOLAR keeps auto-routing, HC/HP stays 0 %.
  A daytime HC window deliberately does NOT trigger grid heating - the day
  overlaps the solar-production window, and a day-vs-night decision is
  deferred (would be marginal anyway).
- The panel **contactor must be forced ON permanently** on HC/HP-tariff
  installations (or bypassed): WALTER now owns the off-peak schedule
  (`input_datetime.walter_hc_start` / `walter_hc_end`, defaults 22:00/06:00).
  Option Base subscribers and the bench have no contactor - nothing to do.
  Verify with: mode MANUAL + max_level 100 -> boiler heats in HP hours proves
  the line is live.
- **TIC (Teleinfo / HPHC) optional override** (`input_boolean.walter_tic_use`,
  default off): when armed, the REAL-TIME off-peak signal is authoritative
  within the 20:00-06:00 band - the static schedule is ignored (dashboard
  greys the schedule inputs out). Outside the band TIC is never consulted, so
  a daytime off-peak signal can never force heating. Wire it by making
  `binary_sensor.tic_offpeak` track your TIC PTEC sensor; entity missing/off
  = schedule rules. Users without a TIC device are unaffected.
- Daytime HC window (`input_datetime.walter_hc_day_start/end`, defaults
  12:00/14:00) exists in the data model as `binary_sensor.walter_hc_day_active`
  but is INERT ("planned"): no mode reacts to it in this release.
- Enedis HC reform (2025-2027, seasonal windows): the optional TIC override
  (above) absorbs real-world off-peak shifts at night; daytime seasonal slots
  stay deferred.
- v2 (parked): tune HC behavior with weather forecast + household behaviour
  prediction.

## Hardening

- `input_boolean.walter_tank_lockout` - emergency latch. Set by the emergency
  automation; while ON, `walter_mode_control` is a no-op (no mode / HC flip /
  cap change can re-assert 100 % on a hot tank). Manual clear re-enables.
- `automation.walter_mode_control` triggers: mode, `binary_sensor.walter_hc_active`
  (the EFFECTIVE night-forcing signal, see template sensors), `walter_max_level`,
  `walter_tank_lockout`. OFF explicitly zeroes the level (avoids stale 100 %).
- Sanitary / thermostat-cut alarm (works with NO tank probe):
  - `binary_sensor.walter_full_power_pumping`:
    router_level >= max(80, max_level-5) AND load current > 0.5 A
  - `sensor.walter_full_power_hours_24h`: history_stats, "on" time / 24 h
  - `automation.walter_tank_likely_never_reached_thermostat_setpoint`: fires a
    persistent notification when >= 6 h of max-power pumping in 24 h with no
    thermostat cut => tank almost certainly below setpoint. If repeated over
    days this is a legionella risk (water kept < 55 °C).
  - Gated by `input_boolean.walter_thermostat_detection` (OFF on the bench).
    6 h threshold is tuned for the final 2400 W element; user accepted the
    occasional false positive while the factory 1200 W element is in place.

## Emergency stop

- `automation.walter_emergency_stop_when_tank_is_hot`: tank > 68 °C ->
  lockout ON + routing switch OFF + level 0. No auto-recovery.
- Sensor-gated: today `sensor.water_tank_temperature` reads the bench
  `input_number.walter_tank_probe_bench` (default 60). Inert until a real probe
  is installed (firmware `temperature_limiter_DS18B20.yaml` package exists).
  Without a probe, the boiler's own thermostat is the production protection.

## Helpers
- `input_select.walter_mode` - OFF | SOLAR | HC/HP | MANUAL
- `input_number.walter_max_level` (0-100, default 100) - global cap / manual dial
- `input_datetime.walter_hc_start` (22:00) / `walter_hc_end` (06:00)
- `input_datetime.walter_hc_day_start` (12:00) / `walter_hc_day_end` (14:00) -
  daytime HC window, INERT this release
- `input_boolean.walter_tic_use` - master toggle for the optional TIC override
- `input_boolean.walter_tank_lockout`
- `input_boolean.walter_thermostat_detection` (master switch for the sanitary alarm)
- `input_boolean.walter_sanitary_notified` (dedupe latch, hidden from the UI)
- `input_number.walter_tank_probe_bench` (0-80 °C, default 60) - bench tank temp

## Template sensors (config-entry based)
- `binary_sensor.walter_hc_active` - EFFECTIVE night-forcing signal:
  ON only within the 20:00-06:00 band, and there: TIC signal if
  `walter_tic_use` is on, else the scheduled night window (overnight wrap on
  `walter_hc_start/end`). Daytime is always OFF. Consumed by the mode control
  and shown on the dashboard.
- `binary_sensor.walter_hc_day_active` - daytime HC window, informational
  only (INERT).
- `binary_sensor.walter_tic` - TIC off-peak signal gated to the night band
  (display: "Signal TIC actif"). Reference entity: `binary_sensor.tic_offpeak`
  (not wired on this instance).
- `binary_sensor.walter_full_power_pumping` - max-power pumping signal
- `sensor.walter_full_power_hours_24h` - history_stats of "on" time (24 h)
- `sensor.walter_available_surplus` - max(0, -real_power)
- `sensor.water_tank_temperature` - reads input_number.walter_tank_probe_bench
  (later: point at the real tank probe)

## Automations
- `automation.walter_mode_control`
- `automation.walter_emergency_stop_when_tank_is_hot`
- `automation.walter_tank_likely_never_reached_thermostat_setpoint`
- `automation.walter_reset_sanitary_notification`

## WALTER entity IDs (actual, on this instance)
- switch.salon_walter_activate_solar_routing
- number.salon_walter_router_level
- number.salon_walter_up_reactivity / down_reactivity / target_grid_exchange
- sensor.salon_walter_real_power / consumption
- sensor.salon_walter_dimmer_heatsink_temperature
- sensor.salon_walter_dimmer_load_current
- binary_sensor.salon_walter_dimmer_safety_active / triac_stuck_on
- binary_sensor.salon_walter_boiler_not_powered / current_sensor_failure

## Firmware follow-up (this instance is a fork)
- Add `router_max_level` number + `std::min` clamp in `energy_regulation`
  (`engine_1dimmer.yaml`) so SOLAR auto-routing respects the cap. Then have
  `walter_mode_control` sync firmware `router_max_level` <- `walter_max_level`.