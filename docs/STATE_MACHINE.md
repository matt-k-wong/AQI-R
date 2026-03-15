# AQI-R — Finite State Machine Reference

See [`assets/state_machine.svg`](../assets/state_machine.svg) for the visual diagram.

## States

### WARMUP
| Property | Value |
|----------|-------|
| Relay | **OFF** (forced, regardless of sensor reading) |
| Display | "Initialising…" + progress indicator |
| Logging | Suspended |
| Entry | Power-on or hardware watchdog reboot |
| Exit | 10-second timer expires and sensor returns valid data |

The WARMUP state exists to prevent false relay activations during the sensor stabilisation
period. The SPS30 fan must spin up before readings are reliable.

### AUTO_CLEAN
| Property | Value |
|----------|-------|
| Relay | **OFF** |
| Display | Current PM2.5 / PM10 values + "AUTO ✓" |
| Logging | Active (30 s interval) |
| Entry | WARMUP complete, or AUTO_ALERT hold timer expired, or SYSTEM_ERROR cleared |
| Exit | PM2.5 exceeds ON threshold → AUTO_ALERT; Button press → MANUAL_ON; Fault → SYSTEM_ERROR |

This is the nominal idle state. The system is monitoring but the air is clean.

### AUTO_ALERT
| Property | Value |
|----------|-------|
| Relay | **ON** |
| Display | "⚠ POOR AIR" header (inverted colours, flashing at 1 Hz), PPM values |
| Logging | Active (30 s interval, relay_state=ON) |
| Entry | PM2.5 > 50 μg/m³ (ON threshold) while in AUTO_CLEAN |
| Exit | PM2.5 < 40 μg/m³ sustained for 5 min → AUTO_CLEAN; Button press → MANUAL_ON; Fault → SYSTEM_ERROR |

The 5-minute hold is implemented as a countdown timer that resets if PM2.5 rises back above
the OFF threshold during the hold period.

### MANUAL_ON
| Property | Value |
|----------|-------|
| Relay | **ON** (forced, regardless of sensor reading) |
| Display | "MANUAL ●" header, current PPM values (still displayed for information) |
| Logging | Active (30 s interval, mode=MANUAL_ON) |
| Entry | Override button pressed while in AUTO_CLEAN, AUTO_ALERT, or re-press from MANUAL_ON |
| Exit | Override button pressed again → returns to AUTO_CLEAN (FSM re-evaluates AQI immediately) |

MANUAL_ON is a pure toggle. A second button press always returns to AUTO_CLEAN, which will
immediately transition to AUTO_ALERT if the AQI is still above the threshold.

### SYSTEM_ERROR
| Property | Value |
|----------|-------|
| Relay | **OFF** (fail-safe — do not run filter indefinitely unattended on a fault) |
| Display | Error description: "SENSOR FAIL" or "SD FULL" or "SD ERR" |
| Logging | Suspended |
| Entry | Sensor returns invalid data for > 3 consecutive reads, or SD card absent/full |
| Exit | Fault condition is resolved → auto-returns to AUTO_CLEAN |

The system continuously attempts sensor re-initialisation and SD card re-mounting in the
background. Recovery is automatic with no user intervention required.

## Interrupt Handling

The override button is connected to Pin D2 with a hardware interrupt (FALLING edge trigger).

The ISR is minimal:
```cpp
volatile bool buttonPressed = false;
void IRAM_ATTR onButtonPress() {
    buttonPressed = true;  // Set flag only; do not change state in ISR
}
```

State transitions triggered by the button are handled in the main loop on the next iteration.
This avoids calling non-ISR-safe functions (like Serial, SD writes) from inside the ISR.

## Safety Invariants

The following relay rules are enforced in hardware logic **regardless of state**:

1. Relay is always OFF during WARMUP — no exceptions.
2. Relay is always OFF during SYSTEM_ERROR — no exceptions.
3. If `millis()` overflow causes a timer error, the FSM resets to WARMUP.
