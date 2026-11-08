# Wojtek — MM32SPIN05PF FOC regression and fault test plan

## Test policy

Run tests first with no motor power, then with a current-limited bench supply and secured motor. Never begin with a rider or a trike on the ground. Record firmware commit, board revision, motor, supply voltage, current limit, sensor calibration and test equipment for every run.

## A. Build and static checks

- [ ] Build the unmodified legacy configuration for the exact MM32 target.
- [ ] Build with warnings enabled; investigate new warnings.
- [ ] Confirm all source files referenced by `HoverBoard.uvprojx` still exist.
- [ ] Confirm FOC modes remain inaccessible through normal UART config.
- [ ] Confirm invalid stored mode values cannot enter an FOC path at startup.
- [ ] Inspect integer overflow, signedness and ISR/main shared-variable access.
- [ ] Verify no blocking calls were introduced into ADC ISR or speed loop.

## B. Legacy regression (must pass before FOC work)

For modes 0–3:
- [ ] Startup and disarm behavior unchanged.
- [ ] Positive and negative command behavior unchanged.
- [ ] UART timeout sets command to zero as expected.
- [ ] Low-battery behavior unchanged.
- [ ] Current protection and hardware break remain active.
- [ ] No unexpected PWM output after reset or invalid configuration.
- [ ] Verify both motor directions at low current on a secured bench.

## C. UART/configuration

- [ ] Valid type-0 packet accepted only for the configured slave ID.
- [ ] Valid type-1 packet accepted only for the configured slave ID.
- [ ] Valid type-2 packet can change only supported legacy modes.
- [ ] Mode values 4, 5, 6 and out-of-range values are rejected.
- [ ] Bad CRC, truncated packet, unknown type and wrong slave ID are rejected.
- [ ] Parser powers up unsynchronized: arbitrary startup bytes before `/` are ignored.
- [ ] Unknown type immediately resets parser state; following valid packet is accepted.
- [ ] Invalid/zero/stale frame length cannot reach CRC calculation or index outside the receive buffer.
- [ ] Back-to-back valid frames of types 0, 1 and 2 are accepted without parser desynchronization.
- [ ] Startup diagnostic text is ignored until a complete valid packet is received.
- [ ] Timeout is tested independently on both controllers.
- [ ] Verify TX topology before connecting two controllers to a shared bus.

## D. Sensor and power-stage validation

- [ ] Measure ADC zero offsets with motor current at zero.
- [ ] Verify phase-current sensor polarity independently for both measured phases.
- [ ] Verify total-current scaling against a calibrated current meter.
- [ ] Verify battery voltage scaling against a calibrated voltmeter.
- [ ] Verify all six valid Hall states and legal transition order.
- [ ] Verify illegal Hall states 000 and 111 are detected and cannot create torque.
- [ ] Verify Hall timeout and stopped-rotor behavior.
- [ ] Verify hardware break polarity, latency and latched/recovery behavior.
- [ ] Verify gate outputs remain disabled during reset, startup and fault.

## E. FOC math and control-loop tests

- [ ] Unit-test Clarke transform with known phase-current vectors.
- [ ] Unit-test Park/inverse-Park round trip over representative angles.
- [ ] Unit-test sine/cosine lookup table boundaries.
- [ ] Unit-test SVPWM sector selection and duty bounds.
- [ ] Test PI saturation and integrator anti-windup.
- [ ] Test integrator reset on disarm, timeout, fault, mode change and reversal.
- [ ] Verify reference handoff cannot tear between main loop and ADC ISR.
- [ ] Measure worst-case ADC ISR execution time.
- [ ] Verify no invalid/NaN/overflow values can reach PWM compare registers.

## F. FOC commissioning sequence

- [ ] FOC remains disabled by default in release configuration.
- [ ] Begin with PWM outputs physically disconnected from the power stage.
- [ ] Confirm generated duty cycles are bounded and centered as intended.
- [ ] Use a current-limited supply and secured motor for first energized tests.
- [ ] Confirm Hall angle and measured current-vector relationship at very low current.
- [ ] Confirm d-axis command response before q-axis torque tests.
- [ ] Establish q-axis sign separately for forward and reverse.
- [ ] Test zero command, command ramp, timeout, invalid Hall and emergency power removal.
- [ ] Do not proceed if any current spike, unexpected rotation or loss of control occurs.

## G. Two-controller system

- [ ] Each board has a verified unique addressing/connection strategy.
- [ ] ESP32 loss causes both boards to stop.
- [ ] Loss of either MM32 telemetry causes both sides to enter failsafe.
- [ ] One board reset cannot leave the other driving.
- [ ] Re-arm requires neutral command and deliberate operator action.
- [ ] Physical emergency stop removes motor power independently of UART software.

## Acceptance criteria

A test is PASS only with recorded evidence. A compile success is not proof of safe motor control. Any unexpected motion, current spike, Hall inconsistency, stale command or failed break test is an immediate FAIL and blocks further energized tests.
