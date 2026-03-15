# Bill of Materials — AQI-R

Machine-readable version: [`BOM.csv`](BOM.csv)

## Component Selection Rationale

### MCU — Arduino Nano ESP32
The Nano ESP32 was chosen over the classic Uno or Nano for three reasons:
- **Native 3.3 V logic** — eliminates the need for level shifters on SPI and I²C buses.
- **Dual-core processor** — allows SD logging on Core 1 without blocking FSM/sensor loop on Core 0.
- **Native USB-C** — no separate USB-to-serial chip; cleaner power delivery.

### PM Sensor — Sensirion SPS30
- Laser particle counter with factory calibration (unlike MQ-series which are analog and drift).
- Reports PM1.0, PM2.5, and PM10 simultaneously.
- Communicates over I²C or UART (I²C selected for this design).
- Has a built-in self-cleaning function (fan blows at high speed to clear contamination).
- Requires 5 V supply but 3.3 V-compatible logic lines.

### Relay — Solid State Relay (SSR)
A mechanical relay was explicitly **not** chosen because:
- SSRs have no moving parts → silent, no contact wear, longer lifespan.
- SSRs offer superior optical isolation between the 5 V trigger side and the 120/240 V AC load.
- Zero-cross detection variants (G3MB series) reduce EMI on the AC line at switch-on.

### RTC — DS3231
- ±2 ppm accuracy (much better than DS1307).
- Has a built-in temperature-compensated oscillator.
- CR2032 backup cell maintains time when the main system is unpowered.
- The common breakout modules include an AT24C32 EEPROM on the same I²C bus (address 0x57)
  which can be used for storing calibration offsets in future firmware revisions.

### Display — SH1106 OLED
- 1.3" 128×64 provides enough real estate for a 4-line display (state, PM2.5, PM10, SD status).
- I²C interface keeps pin count low (2 wires shared with sensor and RTC).
- Library support: `U8g2` recommended for full font and graphics support.

### Dedicated 3.3 V LDO
The Arduino Nano ESP32's onboard 3.3 V rail is rated for the MCU but may struggle with the
SD card's current spikes during write cycles (up to 100 mA peak). A dedicated AMS1117-3.3
(800 mA capable) powered from the 5 V USB rail gives the SD card a clean, stable supply.

## Sourcing Notes

| Supplier | Good For |
|----------|----------|
| **Mouser / DigiKey** | Sensirion SPS30, quality capacitors, LDO regs |
| **Arduino Store** | Arduino Nano ESP32 (official) |
| **Adafruit** | SD breakout module (5 V regulator onboard — bypass it) |
| **Amazon** | DS3231, OLED, button, enclosure (lead time 1–2 days) |
| **LCSC / AliExpress** | Cost reduction for passives and SD module (10–30 day shipping) |

## Tools Required

- Soldering iron + solder
- Multimeter (for continuity check before first power-on)
- Breadboard (for prototyping phase)
- Fritzing or KiCad (for schematic capture — to be added to `/hardware/`)
- USB-C cable (data-capable, not charge-only)
