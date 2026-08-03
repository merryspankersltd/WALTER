# WALTER — Solar Router for ESPHome

**[🇬🇧 English](README.md) · [🇫🇷 Français](README.fr.md)**

**(WALTER Always Loves Tiny Export Rates)**

> **WALTER** is a fork of
> [hacf-fr/Solar-Router-for-ESPHome](https://github.com/hacf-fr/Solar-Router-for-ESPHome)
> adding native support for the **Shelly EM3 Pro / Pro 3EM** three-phase meter,
> aimed at diverting photovoltaic surplus to a **hot-water boiler**.

WALTER does exactly what he says: it redirects the photovoltaic surplus
to a boiler and keeps grid injection as low as possible.

## Why the 3-phase sum?

On a three-phase contract the meter adds the three phases and bills only the
**net** value. When one phase produces more (photovoltaic) than the other two
consume, that's the right time to heat water.

Based on smart-meter readings, WALTER drives the boiler element as close as
possible to that value. See [docs/strategy.md](docs/en/strategy.md) for the
full reasoning.

## Hardware

| Part | Model |
|------|-------|
| MCU | ESP32 (esp32dev, esp-idf framework) |
| Regulator | RobotDyn RBD-340 triac dimmer 40 A "fan version" |
| Meter | Shelly EM3 Pro / Pro 3EM (HTTP RPC, polled locally — no HA needed) |
| Load | Pure resistive load up to 3000 W (water boiler) |
| Safety | boiler thermostat, optionally a temperature limiter (DS18B20 or HA temperature) |

- Wiring: [docs/en/wiring_rbd40.md](docs/en/wiring_rbd40.md)
- Safety procedure:
  [docs/en/bench_test.md](docs/en/bench_test.md)

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

## Credits & license

Built on [hacf-fr/Solar-Router-for-ESPHome](https://github.com/hacf-fr/Solar-Router-for-ESPHome)
(GPL-3.0). See `LICENSE`. Read the upstream
[disclaimer](https://hacf-fr.github.io/Solar-Router-for-ESPHome/disclamer/)
before doing anything with 230 V.
