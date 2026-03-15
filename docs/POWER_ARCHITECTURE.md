# AQI-R — Power Architecture

## Power Tree

```
USB-C (5 V, up to 3 A)
│
├─── 5 V Rail ──────────────────────────────────────────────────────
│    │
│    ├── SPS30 VCC (5 V supply, up to 80 mA peak during self-clean)
│    ├── SSR trigger coil (< 10 mA)
│    └── AMS1117-3.3 LDO input
│
└─── 3.3 V Rail (via AMS1117-3.3) ─────────────────────────────────
     │
     ├── Arduino Nano ESP32 VIN (internal regulator bypassed)
     ├── SH1106 OLED VCC (10–15 mA)
     ├── MicroSD card VCC (up to 100 mA during write)
     └── DS3231 RTC VCC (< 1 mA active, μA in backup mode)
```

## Power Budget

| Component | Supply | Typical (mA) | Peak (mA) |
|-----------|--------|--------------|-----------|
| Arduino Nano ESP32 (active, Wi-Fi off) | 3.3 V | 80 | 240 |
| Sensirion SPS30 (fan running) | 5 V | 50 | 80 |
| OLED SH1106 (full brightness) | 3.3 V | 15 | 20 |
| DS3231 RTC | 3.3 V | 0.2 | 2 |
| MicroSD (write cycle) | 3.3 V | 30 | 100 |
| SSR trigger | 5 V | 8 | 10 |
| **Total 3.3 V rail** | | **125** | **362** |
| **Total 5 V rail (incl. 3.3V via LDO)** | | **235** | **480** |

USB 2.0 provides 500 mA at 5 V. USB 3.0+ and USB-C PD sources provide ≥ 900 mA.
The peak draw of ~480 mA is within USB 2.0 limits but leaves minimal headroom.

**Recommendation:** Use a USB-C power supply rated at ≥ 1 A (a phone charger is sufficient).

## Why a Dedicated SD Card LDO?

The Arduino Nano ESP32's onboard 3.3 V regulator is designed primarily for the MCU. MicroSD
cards draw short, sharp current spikes during write operations (up to 100 mA for 50–200 ms).
These spikes can cause the onboard regulator to brownout, resetting the MCU mid-write —
resulting in corrupted log files.

The AMS1117-3.3 (rated 800 mA, dropout 1.3 V at full load) is powered from the clean 5 V
USB rail. Its lower dropout at the 100–130 mA operating range means the 3.3 V rail barely
sags even during SD write peaks.

## Decoupling Strategy

| Location | Capacitor | Purpose |
|----------|-----------|---------|
| AMS1117-3.3 output | 100 μF electrolytic + 0.1 μF ceramic | Bulk storage for SD write spikes |
| SD card VCC pin | 0.1 μF ceramic (as close as possible) | High-frequency decoupling |
| ESP32 3.3 V pin | 0.1 μF ceramic (onboard, but add external if instability seen) | HF decoupling |
| SPS30 5 V pin | 47 μF electrolytic (if long wires used) | Suppress fan motor inrush noise |

## EMI Mitigation

The air filter's AC motor generates electromagnetic interference when switched. Mitigations:

1. **RC Snubber** — a 100 Ω resistor in series with a 0.1 μF / 250 V capacitor, placed
   directly across the relay's AC output terminals. This absorbs the inductive kickback
   from the motor at switch-off.

2. **Ferrite Bead** — a ferrite bead on the USB-C cable (clamp-on type) reduces conducted
   RF noise from the AC side finding its way back through the power supply ground.

3. **Physical Separation** — the SSR and AC wiring should be on one side of the enclosure.
   The MCU, sensor, and low-voltage wiring should be on the opposite side.
