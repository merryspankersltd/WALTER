# Bench test — do this before letting WALTER drive the boiler

WALTER (and the upstream Solar Router for ESPHome engine it is built on) can
switch real power. Perform the whole procedure on a bench first, with a dummy
resistive load (e.g. a halogen heater or incandescent lamps, far away from the
final boiler), and only move to the boiler once every step is clean.

> ⚠️ The temperature-limiter safety logic may contain bugs: validate behaviour
> carefully before letting the system run unattended (upstream warning).

## 1. Static checks (ESP only, no AC load)

Power the ESP on USB only.

- [ ] ESP connects to Wi-Fi and the Home Assistant API shows the node
- [ ] `http://<walter-ip>/` (web_server) responds
- [ ] No red safety LED is lit

## 2. Meter bring-up (no routing)

Enable the power meter (switch "Power Meter" in HA, or set
`power_meter_activated_at_start: "1"`).

- [ ] Logs show `EM3 response status: 200`
- [ ] `sensor.real_power` tracks the installation — it is the **3-phase
  arithmetic sum**, so compare it to the sum of the three Shelly channel powers
  at a moment when the phases differ
- [ ] Sign convention correct: positive = import, negative = export. If
  inverted, set `power_sign: "-1"` in the meter vars
- [ ] Toggle a few hundred watts in the house; `real_power` reacts within a
  second or two

## 3. Manual dimmer test (bench load)

- [ ] `Regulator Opening` at 0 % → lamp off
- [ ] 30 % → lamp clearly dimmed, no flicker / buzzing
- [ ] 100 % → lamp fully on
- [ ] Sweep 0-100 %; below ~5 % the triac may drop out (normal)
- [ ] Watch the triac heatsink and fan: warm but not hot at 100 % for 10 min

## 4. Automatic mode (bench load)

- [ ] Connect the bench load instead of the boiler
- [ ] Enable `Activate Solar Routing`
- [ ] First set `Target grid exchange` to a small **positive** value (e.g.
      `+100`) to watch it react
- [ ] When the installation nets negative (PV surplus after cross-phase
      netting), WALTER ramps the load up
- [ ] Interrupt the surplus (e.g. overload a phase to make the sum positive):
      WALTER must back the load off within a few seconds
- [ ] Kill the meter (unplug / power off the Shelly): `real_power` becomes
      `NaN`, the router drops to 0 % within ~1 s (fail-safe)
- [ ] Restore the meter: regulation resumes

## 5. Temperature limiter test

- [ ] Feed a test value (e.g. an `input_number`) into the
      `temperature_sensor` entity
- [ ] Raise it above `Stop temperature` → `Safety limit reached` = ON, red LED,
      and the router must drop to 0 %
- [ ] Lower it below `Restart temperature` → regulation resumes

## 6. Before the real boiler

- [ ] On the bench, confirm the power margin: the boiler element is rated
      2400 W max, and the router must never command more than that. Per-phase
      6 kVA limits are satisfied automatically because the router only fires
      when the whole 3-phase sum is negative
- [ ] Swap the bench load for the boiler element only after a track record:
      several hours of clean auto behaviour across changing sun/cloud
- [ ] Once installed on the boiler, point the temperature limiter at the real
      tank temperature

Do not put your system back to unattended mode: the first days, check the HA
dashboard / `energy_diverted` and the tank temperature curve before trusting it.