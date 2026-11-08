# Wojtek — EXTREME MM32 performance profile

## Selected hardware profile

- MOSFET identity: UNKNOWN
- Cooling: PASSIVE / original board cooling
- Objective: highest combined usable torque and speed
- Current release state: FOC modes 4–6 blocked at startup and through UART configuration
- Current limits: do not increase until the power stage and analog scaling are identified and measured

This profile is deliberately a commissioning profile, not a high-current preset. With unknown MOSFETs and passive cooling there is no defensible numeric current limit or thermal ceiling to enter into firmware.

## What "EXTREME" means for this build

The target is maximum repeatable output without relying on MOSFET avalanche, thermal overload, shunt overload, driver overstress or battery sag. Peak current is not a useful performance target if the board overheats or trips after a few seconds.

Performance work is split into independent controls:

1. **Torque:** verified signed q-axis current control, current sensing, current slew limits and measured peak/RMS current.
2. **Speed:** verified speed feedback, speed-loop tuning, voltage headroom and motor mechanical limits.
3. **Thermal:** board temperature, MOSFET case/PCB temperature, motor winding temperature and cooling airflow.
4. **Power:** battery voltage under load, battery/BMS current capability, connector and cable losses.
5. **Reliability:** independent hardware overcurrent/break, watchdog, Hall plausibility, dual-controller failsafe and deliberate re-arm.

Torque and top speed are not the same thing. Increasing q-axis current mainly raises torque; raising top speed needs voltage headroom and a validated speed/commutation strategy. Field weakening is not enabled by this profile.

## Immediate firmware policy

- Keep drive modes 0–3 as the only modes selectable through normal UART configuration.
- Keep startup fallback to COM_VOLT for stored drive-mode values above SINE_SPEED.
- Do not change `SOFT_ILIMIT`, `AWDG`, PWM_RES, battery limits, watchdog settings or PI gains based on guessed MOSFET identity.
- Do not use the sample `pinstorage[64]` initializer as proof of the installed board's calibrated analog scaling.
- Do not change the q-axis lower PI bound to a negative value until current polarity, Hall direction and Park transform signs are verified.
- Do not enable FOC_SPEED until a tested speed-loop output actually updates the q-axis reference.
- Do not enable FOC_TORQUE until signed q-axis current control has passed bench tests in both directions.
- Do not add field weakening, overmodulation or increased PWM frequency as a shortcut to speed.

## Required hardware identification

Before numeric tuning, record:

| Item | Required evidence |
|---|---|
| MOSFETs | Exact top marking and package, all devices on both boards |
| Gate driver | Exact IC marking and datasheet limits |
| Current shunts | Resistance marking or four-wire measured resistance |
| Current amplifier | IC marking, gain and input/output topology |
| Battery | Maximum charged voltage, chemistry, BMS continuous/peak current |
| DC link | Capacitor voltage rating, capacitance and condition |
| Cooling | Heatsink arrangement, thermal interface and measured airflow |
| Motor | Phase resistance, winding configuration, Hall sequence and temperature |
| PCB | Board revision, copper condition, connector and wire ratings |

Photograph both sides of each board and close-ups of MOSFET and driver markings. Left and right boards must be checked separately; do not assume they are identical.

## Measurement-led limit calculation

Do not derive a firmware current limit from a MOSFET headline current rating. That rating assumes specified case temperature, mounting and thermal conditions and does not describe a hoverboard PCB.

For the actual board, determine:

- Shunt resistance (R_s), amplifier gain (G), ADC reference and ADC counts per volt.
- Actual current conversion from measured current and ADC delta.
- Hardware analog-watchdog threshold and its polarity.
- MOSFET conduction loss using measured RMS phase current and hot (R_{DS(on)}).
- Switching loss using measured bus voltage, switching frequency, gate resistance and device switching data.
- Board thermal resistance and steady-state temperature rise in the installed enclosure.

Use measured values and worst-case datasheet values. The firmware threshold must remain below the weakest verified limit among MOSFETs, driver, shunts, PCB, connectors, battery/BMS and motor. Retain margin for sensor error, transient current and temperature rise.

## Staged tuning plan

### Stage 1 — identify and baseline
- Confirm exact MOSFET and gate-driver markings on both boards.
- Measure shunts and validate current/voltage scaling with calibrated instruments.
- Confirm the existing hardware break and ADC watchdog actually disable gate drive.
- Record baseline temperatures, no-load current and loaded current in an established legacy mode.
- No current or PWM increase.

### Stage 2 — improve legacy control safely
- Verify UART timeout and neutral-command behavior.
- Measure speed feedback noise and latency.
- Tune only legacy speed-loop response after logging setpoint, measured RPM, PWM and current.
- Avoid raising the speed limit to hide poor feedback or motor saturation.
- Regression-test modes 0–3 after each change.

### Stage 3 — FOC bench commissioning
- Validate current sensor offsets and polarity.
- Validate Hall states, direction and electrical angle.
- Validate Clarke/Park/inverse-Park and SVPWM with captured data.
- Verify d-axis control with the power stage disabled first.
- Establish signed q-axis current at very low measured current.
- Add current slew limiting, PI anti-windup and fault resets.
- FOC remains disabled in release builds until the full test plan passes.

### Stage 4 — torque and speed optimization
- Tune current-loop bandwidth from measured ADC/PWM timing and motor response.
- Tune speed-loop bandwidth below the inner current-loop bandwidth.
- Add controlled acceleration/deceleration ramps.
- Validate battery sag and thermal equilibrium at each load step.
- Evaluate speed increase separately from torque increase.
- Do not use field weakening until voltage, motor and rotor-angle limits are characterized.

### Stage 5 — dual-board vehicle validation
- Verify independent communication and unique board identity.
- Loss of RC or either board must stop both sides.
- Require neutral input and deliberate re-arm.
- Test physical emergency stop independently of firmware.
- Vehicle testing only after bench acceptance and supervised low-speed validation.

## Required logging

Log timestamp, requested command, measured RPM, bus voltage, battery current, phase currents, q/d references and feedback, PWM duty, Hall state/angle, fault flags and board temperature. Keep a separate record for each controller.

## Release decision

With MOSFET identity unknown and passive cooling, this branch must not increase current limits or unlock FOC. The next useful step is component identification and current-sensor calibration. After that, limits can be calculated for the actual hardware rather than guessed.
