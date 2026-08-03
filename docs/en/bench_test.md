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

- [ ] Fan already runs at 100 % at boot (FAN hardwired to 5 V — no firmware
      involved)
- [ ] `Dimmer Heatsink Temperature` reads ambient (see calibration below)
- [ ] `Dimmer Load Current` ≈ 0 A with the regulator closed
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

## 5. Board safety calibration (heatsink NTC + current sensor)

- [ ] **Heatsink NTC**: with the board at ambient, compare `Dimmer Heatsink
      Temperature` to a thermometer on the heatsink; if off, adjust the
      `ntc_*` vars (series resistor / divider configuration) in
      `dimmer_safety.yaml`. Verify at a second point: warm the heatsink (hair
      dryer) to ~50-60 °C and re-check
- [ ] **Current sensor**: at 100 % opening with the bench load, compare
      `Dimmer Load Current` with a clamp meter on the AC-OUT wire; adjust the
      "current precision" trimmer (and/or `current_calibration_factor`) until
      they match. Re-check at ~30 % (phase-cut is non-sinusoidal — RMS matters)
- [ ] `Current Sensor Failure` is OFF once the CT reads
- [ ] `Triac Stuck ON` is OFF with the regulator closed and the load connected

## 6. Safety cutout tests

- [ ] **Heatsink overheat**: heat the heatsink near the NTC above
      `heatsink_stop_temperature` (default 80 °C) → `Dimmer Safety Active` = ON,
      red LED, router drops to 0 %; once it cools under
      `heatsink_restart_temperature` (default 60 °C) → regulation resumes
- [ ] **Overcurrent**: temporarily set `overcurrent_current` below the bench
      load's real current (e.g. `"3.0"`) → router drops to 0 % within ~1 s,
      red LED on; restore the default afterwards
- [ ] **Not-powered alarm**: with the router forced to 100 % (manual
      `Router Level`) and the load disconnected, `Boiler Not Powered` = ON
- [ ] **Sensor-loss fail-safe**: unplug the `TEMP` wire → safety engages
      (red LED, 0 %); reconnect → recovers at the next reading

## 7. Before the real boiler

- [ ] On the bench, confirm the power margin: the boiler element is rated
      2400 W max, and the router must never command more than that. Per-phase
      subscription limits are satisfied automatically because the router only
      fires when the whole 3-phase sum is negative
- [ ] Swap the bench load for the boiler element only after a track record:
      several hours of clean auto behaviour across changing sun/cloud
- [ ] Once installed on the boiler, monitor the tank temperature in HA — the
      boiler's own thermostat remains the final protection

Do not put your system back to unattended mode: the first days, check the HA
dashboard / `energy_diverted` and the tank temperature curve before trusting it.
