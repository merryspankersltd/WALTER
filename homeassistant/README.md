# WALTER — Home Assistant integration

WALTER is a self-contained ESPHome device: it polls the Shelly EM3 Pro over
HTTP itself and does not need HA to regulate. Home Assistant is used for:

1. **Dashboard / history** — the ESPHome integration exposes the router state
   (router level, real power, target, safety, counts) automatically.
2. **Temperature safety** — the HA temperature-limiter package reads a tank
   temperature entity from HA.
3. **Tariff / mode logic** — the peak/off-peak automation examples in this
   folder.

## Files

| File | Purpose |
|------|---------|
| `templates.yaml` | Template sensors: 3-phase sum (verification), available surplus, tank temperature passthrough. Include it in `configuration.yaml`. |
| `automations.yaml` | Mode selector (OFF / SOLAR / HC), off-peak override, hot-tank emergency stop; reference examples to adapt. |

## Setup

1. Flash WALTER (see repo README), add it in HA → ESPHome; the API-encryption
   key is in your `secrets.yaml`.
2. Include `templates.yaml`, adapt the three `sensor.shelly_em3_pro_*_active_power`
   entity_ids to your actual Shelly integration names.
3. Set `sensor.water_tank_probe` to your real tank probe (or set the
   `input_number` during bench).
4. Create `input_select.walter_mode` (OFF / SOLAR / HC) and include
   `automations.yaml` (adapt IDs / names to your instance).

## Entities send by WALTER (main ones)

- `switch.activate_solar_routing` — master enable (upstream "Activate Solar
  Routing")
- `number.router_level` — 0–100 % opening (auto or forced)
- `number.target_grid_exchange` — W, control setpoint (default 0)
- `sensor.real_power` — the live 3-phase arithmetic sum (W, +/-)
- `switch.activate<...>_scheduler` + scheduler numbers — forced-run window
- `binary_sensor.safety_limit_reached` & the red-LED
- `sensor.total_energy_diverted` — if the energy counter package is enabled
- Phase A/B/C powers (debug, internal unless `show_phase_power: "True"`)

## Other loads on the same phases

Any other appliance (second water heater, EV charger, ...) is transparent to
the loop: WALTER only reads the net 3-phase sum, and large steps caused by
other loads are absorbed by the engine's up/down reactivity and hysteresis
(see `docs/en/strategy.md`).