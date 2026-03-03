# Harness Tester Challenge — Bug Audit Report

**Date**: 2026-03-03  
**Auditor**: Cloud Agent  
**Repository**: commaai/harness_tester_challenge  

---

## 1. Executive Summary

**Total confirmed bugs: 22** (HT-005 removed after datasheet verification — SiSS27DN Vds=30V > TVS clamp 26V)

| Domain | Count |
|--------|-------|
| Schematic | 7 |
| PCB | 4 |
| Firmware | 10 |
| Cross-domain | 1 |
| **Total** | **22** |

| Severity | Count |
|----------|-------|
| Fatal | 7 |
| Major | 13 |
| Minor | 2 |
| **Total** | **22** |

The board has multiple **fatal** defects in every domain that would prevent basic operation. The UART lines to the GPS are swapped, the I2C SDA pull-up is wired to GND, the I/O expander uses the wrong-size TQFP-100 footprint, and the firmware has several logic inversions and missing initialization calls. Even if the hardware were corrected, the firmware would still fail due to inverted test logic, undefined-behavior integer shifts, and a missing `cy.begin()` call.

### Corrections from initial report

| Bug | Original claim | Correction | Result |
|-----|---------------|------------|--------|
| HT-003 | N-ch MOSFET used | SiSS27DN is **P-channel** (Vishay datasheet). Real bug: Source/Drain swapped | Bug revised |
| HT-005 | Vds 20V < TVS 25.8V | SiSS27DN Vds = **30V**; SMAJ16A Vc = **26V**; 30 > 26 | **Removed** |
| HT-006 | Wrong package type (QFN-68) | CY8C9560A IS TQFP-100 but **14×14mm/0.5mm**; footprint is **12×12mm/0.4mm** | Bug revised |

---

## 2. Bug Table (22 Confirmed)

| ID | Domain | Sev | Location | What's wrong | Evidence | Why it matters |
|----|--------|-----|----------|-------------|----------|----------------|
| HT-001 | Schematic | Fatal | U2↔U3 nets UBX-TXD/RXD | UART TX/RX swapped: GPS TXD→Teensy TX1, GPS RXD→Teensy RX1 | Netlist: TX-to-TX, RX-to-RX | No GPS data; stuck in "waiting for time" |
| HT-002 | Schematic | Fatal | R3 (4k7) pin 1 → GND | SDA pull-up wired to GND instead of +3.3V | Net [GND] contains R3.1(); R2.1() on +3.3V | I2C bus held LOW; CY8C9560A unreachable |
| HT-003 | Schematic | Major | Q1 (SiSS27DN) S/D swap | P-FET Source/Drain swapped vs standard topology; reverse-polarity protection fails when caps pre-charged | SiSS27DN is P-ch (Vishay). S→+12V load, D→J1 input (should be opposite) | FET stays ON during reverse polarity → destructive current |
| HT-004 | Schematic | Major | D3 LED_R/G/B nets | No current limiting resistors on RGB LED | No resistor on LED_R/G/B nets | GPIO overcurrent (red) or LEDs too dim (green/blue) |
| HT-006 | Cross-domain | Fatal | U4 footprint | Footprint TQFP-100 12×12mm/0.4mm; real part is 14×14mm/0.5mm | CY8C9560A-24AXI datasheet: 14×14mm TQFP-100 | Pins don't align; chip can't be soldered |
| HT-007 | Schematic | Major | C7 (10µF) 0402 on +12V | 10µF doesn't exist in 0402; voltage too high | C7=10u, fp=C_0402; +12V net | Unsourceable part; missing input filter cap |
| HT-008 | Schematic | Major | C5 (100nF) GPS RF path | 100nF far too large for 1.575 GHz; should be pF range | C5 in series on RF_IN path; SRF in low MHz | GPS sensitivity severely degraded |
| HT-009 | Firmware | Fatal | firmware.ino:144,152 | `1 << i` UB for i≥32; literal is 32-bit int | NUM_HARNESS_PINS=40; pins 32-39 UB | Upper 8 pins untestable; wrong results |
| HT-010 | Firmware | Fatal | firmware.ino:138 | Button logic inverted: test runs when NOT pressed | `if(LOW) return` with pull-up to 3.3V | Continuous testing; button stops instead of starts |
| HT-011 | Firmware | Fatal | firmware.ino setup() | `cy.begin()` never called; I2C not initialized | No call in setup(); Wire2 never started | I/O expander completely non-functional |
| HT-012 | Firmware | Fatal | CY8C9560.cpp:6-8 | Reset sequence inverted: ends LOW (reset asserted) | HIGH→delay→LOW; active-low RESET_N | Device held in reset; I2C always fails |
| HT-013 | Firmware | Major | firmware.ino:118-131 | NMEA buffer (64B) no bounds check; max NMEA=82 chars | nmea_idx++ without limit check | Buffer overflow → memory corruption |
| HT-014 | Firmware | Major | firmware.ino set_status() | LED pins never set to OUTPUT; digitalWrite on INPUT only toggles pull-up | No `pinMode(PIN_LED_*, OUTPUT)` | LEDs invisible (~90µA through 37kΩ pull-up) |
| HT-015 | Firmware | Fatal | firmware.ino:142-157 | Pass/fail logic inverted: passes if ANY pin matches | `passed=false; if(match) passed=true;` never reset | Bad harnesses pass; testing is meaningless |
| HT-016 | Firmware | Minor | firmware.ino:106-107 | SAFEBOOT/RST_N pins not set to OUTPUT before driving | No `pinMode` for pins 3, 4 | Pins float instead of being driven; works by accident via GPS internal pull-ups |
| HT-017 | PCB | Major | F.Cu +3.3V tracks | Broken +3.3V trace: unconnected track segments | DRC: 1 unconnected item at (167.3, 35.8) | Components lose power; partial failure |
| HT-018 | PCB | Minor | C6 / U5 footprints | Courtyard overlap between C6 and U5 (MAX2679) | DRC: courtyards_overlap | Assembly risk; rework difficulty |
| HT-019 | PCB | Major | All layers; 500 violations | Systematic trace-to-zone clearance violations (0.15mm vs 0.20mm required) | DRC: 500 clearance errors | Manufacturing DRC failure; risk of shorts |
| HT-020 | Firmware | Major | CY8C9560.cpp:77-84 | set_output() makes ALL pins outputs; brief shorts through harness | PIN_DIRECTION=0x00 for all ports | Contention current; potential I/O damage |
| HT-021 | Schematic | Major | C1 (1µF) 0402 on +12V | 0402 1µF caps rated 6.3-10V max; used at 12V | C_0402 on +12V net | Capacitor derated to ~0; inadequate filtering |
| HT-022 | Firmware | Major | firmware.ino:74 | `buf[len]=0` off-by-one: writes past buffer when len=64 | nmea_buf[64]; process_nmea called with len up to 64 | One-byte OOB write; memory corruption |
| HT-023 | PCB | Major | In1.Cu +3.3V zone | +3.3V pour fragmented into 12 isolated islands | DRC: 12 isolated_copper on In1.Cu +3.3V | Dead copper; poor power delivery; EMI risk |

---

## 3. Top 5 Most Critical Bugs

1. **HT-001 (UART TX/RX swap)** — GPS TXD wired to Teensy TX1 (output→output). No NMEA data can reach the MCU. The device never acquires a time lock and is permanently stuck. **Symptom**: Blue LED stays on forever (if LEDs worked); serial output stuck at "Waiting for GPS time lock..."

2. **HT-002 (SDA pulled to GND)** — The I2C SDA bus is clamped to GND through 4.7kΩ. Even with correct firmware, the CY8C9560A I/O expander can never communicate. **Symptom**: I2C ACKs never occur; the harness test loop produces garbage or all-zero readings.

3. **HT-006 (Wrong TQFP-100 footprint dimensions)** — The CY8C9560A package is 14×14mm/0.5mm pitch but the PCB has 12×12mm/0.4mm pitch pads. The IC pins will not align. **Symptom**: Board cannot be assembled; I/O expander absent.

4. **HT-015 (Pass/fail logic inverted)** — The firmware declares a harness "passed" if even one pin out of 40 matches. Defective harnesses routinely pass. **Symptom**: Nearly all harnesses test as "passed", including ones with massive wiring errors.

5. **HT-009 (1 << i undefined behavior for i≥32)** — The upper 8 harness pins (32-39) are tested with a corrupted bitmask due to shifting a 32-bit integer by ≥32 positions. **Symptom**: Pins 32-39 always report as pass or fail regardless of actual connectivity; results are non-deterministic.
