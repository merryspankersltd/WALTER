# WALTER — Solar Router for ESPHome

**[🇬🇧 English](README.md) · [🇫🇷 Français](README.fr.md)**

**(WALTER Always Loves Tiny Export Rates)**

> **WALTER** is a fork of
> [hacf-fr/Solar-Router-for-ESPHome](https://github.com/hacf-fr/Solar-Router-for-ESPHome)
> adding native support for the **Shelly EM3 Pro / Pro 3EM** three-phase meter,
> aimed at diverting photovoltaic surplus to a **hot-water boiler**.

It does exactly what its name says: it redirects the photovoltaic surplus to
the boiler and keeps the tiny export rate — actually the *net* grid exchange —
close to **0 W**, driving the **3-phase arithmetic sum** (the quantity that is
billed).

## Why the 3-phase sum?

On a three-phase contract the meter adds the three phases and bills only the
**net** value, so `+500 +800 −1200 = +100` is a 100 W import — and the right
time to heat water is when that sum goes negative, i.e. a real surplus exists
after cross-phase netting:

```
S_grid = P_A + P_B + P_C     (positive = import, negative = surplus)
```

Controlling this sum (instead of a single phase) also protects the phase
limits: the router only fires when the *whole installation* is in surplus, so
it can never push any phase beyond its rating. See `docs/en/strategy.md` for the
full reasoning.

## Hardware

| Part | Model |
|------|-------|
| MCU | ESP32 (esp32dev, esp-idf framework) |
| Regulator | RobotDyn RBD-340 triac dimmer 40 A "fan version" |
| Meter | Shelly EM3 Pro / Pro 3EM (HTTP RPC, polled locally — no HA needed) |
| Load | 2400 W immersion element (water boiler) |
| Safety | temperature limiter (DS18B20 or HA temperature) |

Wiring: `docs/en/wiring_rbd40.md` · Safety procedure: `docs/en/bench_test.md`

## Software

- **Firmware**: ESPHome. `esp32-walter-solar-heater.yaml` + the new package
  `solar_router/power_meter_shelly_em3.yaml` (3-phase meter poller).
- **Home Assistant**: helper templates & automations in `homeassistant/`
  (tank-temp safety, peak/off-peak mode, dashboard).

### ESPHome config (quick glance)

```yaml
packages:
  solar_router:
    url: https://github.com/hacf-fr/Solar-Router-for-ESPHome/
    ref: main
    refresh: 1d
    files:
      - path: solar_router/common.yaml
      - path: solar_router/power_meter_shelly_em3.yaml
        vars:
          power_meter_ip_address: "192.168.1.42"
      - path: solar_router/regulator_triac.yaml
        vars:
          regulator_gate_pin: GPIO17
          regulator_zero_crossing_pin: GPIO23
      - path: solar_router/engine_1dimmer.yaml
        vars:
          green_led_pin: GPIO18
          yellow_led_pin: GPIO19
      - path: solar_router/temperature_limiter_home_assistant.yaml
        vars:
          temperature_sensor: "sensor.water_tank_temperature"
          red_led_pin: GPIO21
```

## Known limits (v1)

- Energy accounting (`total_energy_diverted`) requires the tank temperature
  limiter to be running, otherwise the counter over-counts while the boiler
  thermostat is open.
- The engine is the upstream progressive regulator; a PID / forecast wiring-up
  is parked in `docs/en/strategy.md`.

## Credits & license

Built on [hacf-fr/Solar-Router-for-ESPHome](https://github.com/hacf-fr/Solar-Router-for-ESPHome)
(GPL-3.0). See `LICENSE`. Read the upstream
[disclaimer](https://hacf-fr.github.io/Solar-Router-for-ESPHome/disclamer/)
before doing anything with 230 V.