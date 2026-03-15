# AQI-R — Safety Guidelines

## ⚡ High-Voltage Warning

**This project involves switching 120 V / 240 V mains AC electricity. Mains voltage is
lethal. Incorrect wiring can cause electric shock, fire, serious injury, or death.**

Read and understand this document fully before wiring the relay or connecting the device
to an AC outlet.

---

## Who Should Build This

This project is suitable for builders who:
- Have prior experience working with mains wiring, OR
- Are supervised by a qualified electrician, OR
- Use the "safe alternative" option described below

If you are unsure whether you are comfortable with mains voltage, **use the safe alternative.**

---

## Safe Alternative: Pre-Enclosed Relay Module

Instead of wiring a bare SSR to mains, use a **pre-enclosed smart switch module** such as:

- **Inkbird IHC-200 (or similar)** — DIN-rail relay with fully enclosed AC side
- **PowerSwitch Tail II** — USB-controlled mains switch with fully shrouded terminals
- **Smart plug with dry-contact trigger** — GPIO-controlled via opto-isolated input

These options provide the same functionality with no exposed AC wiring. They are strongly
recommended for makers who are new to mains voltage work.

---

## If You Wire the SSR Directly

### Before You Start
- [ ] Work with the device **completely unplugged** from mains
- [ ] Verify unplugged with a non-contact voltage tester
- [ ] Use wire rated for mains voltage (min 18 AWG, 300 V rated, stranded)
- [ ] Tin all stranded wire ends or use ferrules before inserting into screw terminals
- [ ] Do not leave any bare conductors exposed; use heat-shrink tubing

### Wiring Sequence
1. Wire **low-voltage (DC) side first** — 5 V trigger from Arduino D3 to SSR input terminals
2. Power on and verify software triggers the SSR (LED on SSR should illuminate)
3. Power off and disconnect USB before proceeding to step 4
4. Wire **AC side** — Live and Neutral through the SSR. Wire Earth (ground) directly through, bypassing the SSR
5. Use a NEMA 5-15 (US) or IEC C13/C14 (EU) plug and socket for the AC connections
6. Enclose all AC wiring in a sealed compartment within the project box

### Inside the Enclosure
- The AC side must be **physically separated** from the DC side by a barrier or separate sub-enclosure
- Use cable ties or DIN rail to route AC cables away from DC wiring
- Ensure the enclosure has **strain relief** on all cables entering through the wall
- Label the enclosure: **"CONTAINS MAINS VOLTAGE — DO NOT OPEN WHILE ENERGISED"**

### First Power-On
1. Double-check all connections one more time before energising
2. Do not touch the enclosure during the first power-on test
3. Connect a **low-power test load** (e.g., a desk lamp) before the air filter
4. Verify relay triggers and releases correctly before connecting the actual filter

---

## Regulatory Notes

- This device is a **DIY project** and has not been safety-certified (UL, CE, etc.)
- It should not be used in commercial or rental properties without appropriate certification
- Electrical regulations vary by jurisdiction — consult local codes if in doubt
- Consider RCD/GFCI protection on the outlet circuit feeding this device

---

## Disclaimer

The authors of this project accept no liability for personal injury, property damage, or
any other harm resulting from the construction or use of this device. You build and operate
this project entirely at your own risk.
