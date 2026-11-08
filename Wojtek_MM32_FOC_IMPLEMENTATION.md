# Wojtek — MM32SPIN05PF FOC implementation specification

## Status and scope

This document describes the work required to make the existing experimental FOC path usable on MM32SPIN05PF Gen2 boards. It is an engineering plan, not a claim that FOC is already operational. The existing legacy modes 0–3 must remain behaviorally unchanged.

Audited baseline: `3354fcbe8b621d493576b19de77f1da6d9b2388b`.

## Existing drive modes

Defined in `HoverBoardMindMotion/Inc/bldc.h`:

| Value | Mode | Current status |
|---:|---|---|
| 0 | COM_VOLT | Legacy mode |
| 1 | COM_SPEED | Legacy mode |
| 2 | SINE_VOLT | Legacy mode |
| 3 | SINE_SPEED | Legacy mode |
| 4 | FOC_VOLT | Experimental/incomplete |
| 5 | FOC_SPEED | Experimental/incomplete |
| 6 | FOC_TORQUE | Experimental/incomplete |

The UART config handler currently accepts only values through `SINE_SPEED`. Keep that restriction until the FOC commissioning work is complete.

## Confirmed gaps

1. `speedupdate()` in `Src/bldc.c` calculates `pwm` only for modes 0–3. It does not produce an FOC command/reference for modes 4–6.
2. The ADC ISR in `Src/interrupt.c` runs Clarke/Park, d/q PI, inverse Park and SVPWM for FOC modes.
3. The q-axis PI reference is assigned from `Speed.qOut`, but the inspected source has no active speed-loop path that updates `Speed.qOut`. The reference therefore remains at its initialized value (zero) in the present code path.
4. `CurIQ.qOutMin` is initialized to zero in `Src/PID.c`. Signed reverse torque is therefore not implemented by the present q-axis PI configuration.
5. The current speed PI structure exists, but its units, execution rate, gains, saturation and anti-windup behavior have not been integrated/validated as a cascaded speed-to-Iq loop.
6. Hall angle and direction are used by the FOC path, but the relationship among Hall sequence, phase-current polarity, Park transform sign and command direction has not been proven on the target hardware.
7. Invalid Hall states/transitions, startup angle acquisition and low-speed angle behavior need explicit handling and tests.
8. Existing current protection and PWM break configuration depend on board-specific analog scaling and polarity. Do not change thresholds without measurements.

## Required architecture

- Outer speed loop: nominal 1 kHz task in the main control update, only after speed units and feedback filtering are verified.
- Inner d/q current loops: ADC-triggered ISR, with deterministic execution time.
- d-axis reference: zero initially, subject to verified current-sensor offset and motor behavior.
- q-axis reference: explicit signed command with documented scaling and saturation. Do not infer sign from `speed`, `dir` or `CMDDIR` alone.
- Use a bounded, atomic reference handoff from the outer loop to the ADC ISR.
- Reset integrators and references on disarm, UART timeout, invalid Hall state, mode change, direction reversal and fault.
- Keep the PWM update path and ADC ISR free of blocking calls, dynamic allocation and floating-point work unless timing has been measured and accepted.

## Implementation stages

### Stage 0 — safe containment
- Reject or safely fall back from stored drive-mode values 4–6 at startup.
- Keep UART config writes restricted to modes 0–3.
- Do not expose FOC modes to the RC controller or normal configuration UI.
- Preserve legacy mode behavior.

### Stage 1 — instrumentation and sensor validation
- Verify ADC offsets and phase-current polarity with PWM disabled.
- Verify bus voltage/current scaling against calibrated instruments.
- Verify Hall sequence for both directions and all six legal states.
- Record ADC ISR execution time and PWM synchronization.
- Verify hardware break polarity and actual gate-disable behavior.

### Stage 2 — open-loop/voltage commissioning
- Define exact FOC_VOLT semantics and units.
- Add a compile-time-disabled-by-default development gate.
- Use a current-limited bench supply, motor mechanically secured, wheels off the ground and independent emergency power disconnect.
- Start with zero command and conservative, measured voltage limits.
- Do not permit rider operation.

### Stage 3 — closed-loop current control
- Validate Clarke/Park scaling and angle convention with captured phase currents.
- Validate d-axis response independently.
- Establish q-axis polarity at a very low, hardware-limited current.
- Add signed q-axis reference only after forward/reverse polarity is demonstrated.
- Add PI anti-windup and explicit integrator reset behavior.

### Stage 4 — speed control
- Define RPM units and sign end-to-end.
- Implement speed PI output as the signed q-axis current reference.
- Set loop rate, gains and limits from measured motor/current response, not copied constants.
- Add acceleration/deceleration limits and zero-speed handling.
- Validate loss-of-feedback and stall behavior.

### Stage 5 — release criteria
- All software tests pass.
- Legacy modes 0–3 regression-tested.
- Both splitboards independently tested.
- Fault injection passes.
- Thermal and current limits measured.
- A separate supervised vehicle test plan is approved before any riding.

## Non-goals

This change does not increase current limits, PWM resolution, battery voltage limits, speed limits or watchdog thresholds. It does not authorize road/rider testing. No PI gains are declared production-ready in this document.
