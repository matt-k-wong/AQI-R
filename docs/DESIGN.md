# AQI-R — Design Document & Theory of Operation

## Overview

The AQI-R system is a closed-loop air quality remediation controller. It continuously samples
particulate matter concentration, evaluates the data against configurable thresholds, and
physically actuates a relay to run an external air filter. All readings are timestamped and
logged to an SD card for long-term trend analysis.

## Sampling Strategy

### Sensor Polling vs. Log Interval

The SPS30 is polled **every 1 second** on Core 0. This high-frequency reading drives:
- The OLED display (near real-time PPM readout)
- The FSM threshold comparisons (instantaneous relay decisions)

However, logging every 1 second would wear out the SD card and produce unwieldy CSV files.
Instead, a rolling average is calculated over the last 30 readings, and only this average
(plus the raw PM values at the moment of writing) is written to the SD card every **30 seconds**.

This gives the best of both worlds: fast display updates and clean long-term data.

### Warmup State

The SPS30 datasheet specifies that the sensor fan and laser need approximately **8–10 seconds**
to stabilise after power-on. During this window, readings are unreliable.

The `WARMUP` state enforces a 10-second hold on boot. The relay is forced OFF during warmup
to prevent false-positive activations.

## Hysteresis — Preventing Relay Chatter

Without hysteresis, if the air quality is hovering right at the threshold value, the relay
would oscillate rapidly (ON/OFF/ON/OFF) as the sensor reading fluctuates by ±1–2 μg/m³.
This is called "relay chatter" and causes:
- Audible clicking (not an issue with SSR, but logically undesirable)
- Unnecessary filter wear
- Ugly log data

### Solution: Dual Thresholds + Hold Timer

```
ON  threshold : PM2.5 > 50 μg/m³  → relay turns ON, enter AUTO_ALERT
OFF threshold : PM2.5 < 40 μg/m³  → start 5-minute hold timer
              : Timer expires      → relay turns OFF, return to AUTO_CLEAN
```

The 10 μg/m³ gap creates a "dead band." The system will not switch off the filter until
the air quality is meaningfully cleaner than the trigger point — and even then, it keeps
running for 5 minutes to fully clear the remaining particulates.

## Non-Blocking Architecture

A naive implementation would use `delay()` calls, which freeze the CPU during waits. This
design uses **millis()-based state timing** throughout:

```
lastSensorRead  : tracks when to next poll the SPS30 (1 s interval)
lastLogWrite    : tracks when to next write to SD (30 s interval)
alertHoldStart  : tracks when PM2.5 dropped below OFF threshold (for 5 min hold)
```

The main loop runs as fast as possible, checking these timers on every iteration. No blocking
delays exist in the sensor or logging path.

## SD Card Data Integrity

SD card writes are the most common failure point in embedded data loggers. Mitigations:

1. **Open → Write → Flush → Close** on every write cycle. The file is never left open
   between write cycles. If power is cut between cycles, the last write is always clean.
2. **Card presence check** before every write. If `SD.begin()` fails, the system enters
   `SYSTEM_ERROR` but continues operating (filtering and displaying) without logging.
3. **Watchdog Timer** — if a write hangs (corrupted card, bad contact), the hardware WDT
   reboots the system within 8 seconds. The FSM re-enters `WARMUP` cleanly.

### CSV Format

```
timestamp,pm1_0,pm2_5,pm10,relay_state,mode
2026-03-15 14:32:00,12.4,18.7,24.1,OFF,AUTO_CLEAN
2026-03-15 14:32:30,14.1,52.3,61.0,ON,AUTO_ALERT
```

Fields:
- `timestamp` — DS3231 RTC, ISO 8601 format (YYYY-MM-DD HH:MM:SS)
- `pm1_0`, `pm2_5`, `pm10` — μg/m³, two decimal places
- `relay_state` — ON or OFF
- `mode` — current FSM state name

## Display Layout

The OLED displays different content depending on the current FSM state:

```
┌────────────────┐       ┌────────────────┐
│ AQI-R  AUTO ✓ │       │ AQI-R  ⚠ ALERT│  ← inverted bg
│                │       │                │
│  PM2.5  18.7  │       │  PM2.5  52.3  │
│  PM10   24.1  │       │  PM10   61.0  │
│ ■ SD  14:32   │       │ ■ SD  14:32   │
└────────────────┘       └────────────────┘
  Normal / Clean            ALERT state
```

The `■` icon blinks on each successful SD write (the "heartbeat").

## Configuration Constants

All tunable values are defined in a single `config.h` header (to be created in `src/`):

```cpp
#define PM25_ON_THRESHOLD    50.0f   // μg/m³ — trigger relay ON
#define PM25_OFF_THRESHOLD   40.0f   // μg/m³ — begin relay-off countdown
#define RELAY_OFF_HOLD_MS    300000  // 5 minutes in milliseconds
#define SENSOR_POLL_MS       1000    // Sensor read interval
#define LOG_INTERVAL_MS      30000   // SD write interval
#define WARMUP_MS            10000   // Sensor stabilisation period
#define WDT_TIMEOUT_MS       8000    // Watchdog timeout
```
