# Wojtek — MM32SPIN05PF safe FOC bring-up procedure

## Important

This firmware controls a high-current motor inverter. A compileable image can still produce dangerous torque. This procedure is for a supervised engineering bench, not a rider-carrying vehicle.

## Before power-up

1. Secure the motor mechanically. Keep wheels clear of the floor and remove any rider.
2. Use a current-limited bench supply appropriate for the board and motor.
3. Install a physical disconnect that removes motor-bus power independently of firmware.
4. Keep the power stage disabled while checking wiring and sensor signals.
5. Verify phase order, Hall wiring, current-sensor wiring, voltage divider and board-specific pin configuration.
6. Confirm the hardware break input polarity and gate-driver shutdown path from the schematic/measurement. Do not assume the source default is correct for every board.
7. Confirm the correct board target and firmware configuration.

## Initial firmware state

- Use a legacy mode (COM_VOLT or another already validated mode) for baseline tests.
- FOC modes 4–6 remain blocked from normal UART configuration.
- An invalid persisted mode must fall back to COM_VOLT at startup.
- Do not raise `SOFT_ILIMIT`, `AWDG`, PWM resolution or battery thresholds as part of bring-up.
- Do not write sample EEPROM values blindly; pin mapping and scaling are board-specific.

## Sensor checks with PWM disabled

- Measure current ADC offsets after the board has reached thermal stability.
- Verify both phase-current channels respond with the expected sign to a known low test current.
- Verify total current and battery voltage scaling against calibrated instruments.
- Rotate the motor manually and capture Hall sequence. Confirm six valid states and direction.
- Confirm illegal Hall states are treated as invalid.
- Confirm no PWM/gate activity appears during reset, disarm or fault.

## Energized tests

Proceed only after the sensor checks pass.

1. Use a current-limited supply and a secured motor.
2. Start with legacy mode and the lowest practical command.
3. Verify direction, current and temperature before increasing the command.
4. For FOC development, keep a separate experimental build gate disabled by default.
5. Verify transforms and PWM duties without energizing the power stage first.
6. Energize only at a very low measured current limit.
7. Validate d-axis behavior, then q-axis polarity, then speed-loop behavior as separate tests.
8. Test UART timeout, Hall loss, ADC watchdog/break and physical power disconnect.
9. Stop immediately on unexpected motion, current spikes, abnormal noise or heating.

## Two-board system requirements

- Do not tie TX outputs together unless the electrical bus is explicitly designed for it.
- Verify each board's slave ID and communication wiring.
- The ESP32 must monitor both boards independently.
- Loss of RC input or either board's communication must command both boards to stop.
- Re-arm only after neutral input and deliberate operator action.
- A UART zero command is not a substitute for a physical emergency stop.

## Release gate

FOC is not approved for rider operation until signed q-axis torque, reverse direction, current limiting, thermal behavior, timeout handling, Hall faults, reset behavior and the physical emergency stop have all been validated on the actual hardware with recorded results.
