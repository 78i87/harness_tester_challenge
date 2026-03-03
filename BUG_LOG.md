# BUG_LOG.md — Harness Tester Challenge Audit

**Auditor**: Cloud Agent  
**Date**: 2026-03-03  
**Confirmed bugs**: 22

---

## HT-001
- **ID**: HT-001
- **Domain**: Schematic
- **Severity**: Fatal
- **Status**: Confirmed
- **Location**: U2 (Teensy 4.1) pins 2,3 ↔ U3 (NEO-M8N) pins 20,21; nets `UBX-TXD`, `UBX-RXD`
- **What's wrong**: UART TX/RX lines are swapped. GPS TXD (output, pin 20) connects to Teensy TX1 (output, GPIO 1/pin 3). GPS RXD (input, pin 21) connects to Teensy RX1 (input, GPIO 0/pin 2). Two outputs drive each other; two inputs float.
- **Evidence**: Netlist: `Net [UBX-TXD]: U2.3(1_TX1_CTX2_MISO1), U3.20(TXD/SPI_MISO)` and `Net [UBX-RXD]: U2.2(0_RX1_CRX2_CS1), U3.21(RXD/SPI_MOSI)`. Per NEO-M8N datasheet, TXD=output, RXD=input. Per Teensy4.1, pin 3=TX1(output), pin 2=RX1(input).
- **Why it matters**: No UART communication possible. GPS NMEA data never reaches the MCU; time lock never achieved; device permanently stuck in "waiting for GPS" state.
- **Quick verification**: Trace UBX-TXD and UBX-RXD nets in the schematic. Confirm both endpoints are same direction (output-output and input-input).
- **Minimal fix idea**: Swap the two net connections: Teensy TX1→GPS RXD, Teensy RX1→GPS TXD.

---

## HT-002
- **ID**: HT-002
- **Domain**: Schematic
- **Severity**: Fatal
- **Status**: Confirmed
- **Location**: R3 (4k7), net `CY_SDA` and `GND`
- **What's wrong**: I2C SDA pull-up resistor R3 is connected to GND instead of +3.3V. R3 pin 1 is on GND, pin 2 is on CY_SDA.
- **Evidence**: Netlist: `Net [GND]: ..., R3.1(), ...` and `Net [CY_SDA]: R3.2(), U2.17(25_A11_RX6_SDA2), U4.28(SDA)`. Compare R2 (SCL pull-up): `Net [+3.3V]: ..., R2.1(), ...`
- **Why it matters**: SDA line is held LOW through 4.7kΩ to GND. I2C communication is impossible — ACK/data cannot be driven HIGH. The CY8C9560A I/O expander will never respond.
- **Quick verification**: Check R3 pin 1 net — it's GND, not +3.3V. Compare with R2 (correct pull-up to +3.3V).
- **Minimal fix idea**: Connect R3 pin 1 to +3.3V instead of GND.

---

## HT-003
- **ID**: HT-003
- **Domain**: Schematic
- **Severity**: Major
- **Status**: Confirmed
- **Location**: Q1 (SiSS27DN P-ch MOSFET), reverse polarity protection circuit
- **What's wrong**: P-channel MOSFET Source and Drain are swapped relative to the standard high-side reverse polarity protection topology. Source connects to the load rail (+12V), Drain connects to the input (J1). In the standard topology, Source should be on the input and Drain on the load. With this swap, when reverse polarity is applied to a pre-charged board, Vgs remains strongly negative (gate at GND, source at cap voltage), keeping the FET ON and allowing destructive current flow from the charged load cap back through the reversed supply.
- **Evidence**: Netlist: Q1.1/2/3(S)→+12V (load side), Q1.5(D)→J1.1 (input side), Q1.4(G)→R1→GND. SiSS27DN is P-channel (Vishay datasheet confirms). Standard P-FET reverse polarity: Source=Input, Drain=Load, Gate=GND. Here S and D are reversed. Additionally, D2 (Schottky, cathode on +12V, anode on gate) is permanently reverse-biased and serves no purpose.
- **Why it matters**: Reverse polarity protection fails when load capacitors are pre-charged. FET stays ON during reverse polarity event → large current surge through the circuit → potential damage to components.
- **Quick verification**: Confirm SiSS27DN is P-channel (Vishay product page). Note Source is on load side, Drain is on input side — opposite of standard topology.
- **Minimal fix idea**: Swap Source and Drain connections: Source→J1 input, Drain→+12V load rail.

---

## HT-004
- **ID**: HT-004
- **Domain**: Schematic
- **Severity**: Major
- **Status**: Confirmed
- **Location**: D3 (ASMB-KTF0-0A306 RGB LED), nets `LED_R`, `LED_G`, `LED_B`
- **What's wrong**: No current limiting resistors between the LED cathodes and the Teensy GPIO pins. LED common anode is on +3.3V; cathodes connect directly to U2 GPIOs (pins 7, 8, 9).
- **Evidence**: Netlist: `Net [+3.3V]: ..., D3.1(A), ...`; `Net [LED_R]: D3.4(RK), U2.7(5_IN2)`; `Net [LED_G]: D3.3(GK), U2.8(6_OUT1D)`; `Net [LED_B]: D3.2(BK), U2.9(7_RX2_OUT1A)`. No resistor on any LED net.
- **Why it matters**: LED current is limited only by GPIO output impedance and LED forward voltage. Red LED (Vf≈2.0V) sees 1.3V across GPIO → excessive uncontrolled current → potential GPIO damage. Green/Blue (Vf≈3.0V) may barely light due to only 0.3V headroom.
- **Quick verification**: Search the netlist for any resistor on LED_R/G/B nets — none found.
- **Minimal fix idea**: Add 100-330Ω series resistors between each LED cathode and the GPIO pin.

---

## HT-005
- **ID**: HT-005
- **Domain**: —
- **Severity**: —
- **Status**: REMOVED — FALSE POSITIVE
- **Location**: —
- **What's wrong**: Originally claimed SiSS27DN Vds=20V exceeded by TVS clamp. Web search confirms SiSS27DN Vds=30V, SMAJ16A Vc=26V. 30V > 26V → no issue.
- **Evidence**: Vishay SiSS27DN datasheet: Vds=30V. Littelfuse SMAJ16A: Vc=26V.

---

## HT-006
- **ID**: HT-006
- **Domain**: Cross-domain (Schematic + PCB)
- **Severity**: Fatal
- **Status**: Confirmed
- **Location**: U4 (CY8C9560A-24AXIT), footprint `Package_QFP:TQFP-100_12x12mm_P0.4mm`
- **What's wrong**: The CY8C9560A-24AXIT comes in a TQFP-100 package with 14×14 mm body and 0.5 mm pin pitch. The PCB footprint assigned is `TQFP-100_12x12mm_P0.4mm` — a different TQFP-100 variant with a 12×12 mm body and 0.4 mm pitch. The pad locations do not match the real IC pin locations.
- **Evidence**: Schematic footprint property: `Package_QFP:TQFP-100_12x12mm_P0.4mm`. CY8C9560A-24AXI product listings (RS Components, SicStock, Tanotis) all specify TQFP-100 14×14 mm body. A 100-pin TQFP at 14mm uses 0.5mm pitch (25 pins × 0.5mm = 12.5mm span); the 12mm/0.4mm footprint has smaller body and tighter pitch.
- **Why it matters**: The IC pins do not align with the PCB pads. The chip cannot be soldered. Complete manufacturing failure for the I/O expander.
- **Quick verification**: Compare the assigned footprint dimensions (12×12mm, 0.4mm pitch) with the CY8C9560A-24AXI datasheet package (14×14mm, 0.5mm pitch).
- **Minimal fix idea**: Change footprint to `TQFP-100_14x14mm_P0.5mm` (or create a custom footprint matching the CY8C9560A datasheet dimensions).

---

## HT-007
- **ID**: HT-007
- **Domain**: Schematic
- **Severity**: Major
- **Status**: Confirmed
- **Location**: C7 (10µF), footprint `Capacitor_SMD:C_0402_1005Metric`, net `+12V`
- **What's wrong**: 10µF ceramic capacitor assigned to an 0402 (1005 metric) footprint. 10µF is not available in 0402 package size — the smallest typical package for 10µF MLCC is 0603 (at 6.3-10V) or 0805 (at 16V+). Additionally, even if a hypothetical 10µF 0402 existed, it could not handle 12V.
- **Evidence**: Component list: `C7: 10u, fp=Capacitor_SMD:C_0402_1005Metric`. Net: `+12V: C7.1(), ...`. No 10µF 0402 MLCC exists from any major manufacturer at >6.3V rating.
- **Why it matters**: Part cannot be sourced or assembled. The input filter capacitor is missing, degrading regulator stability and transient handling.
- **Quick verification**: Search distributor catalogs for 10µF 0402 capacitors rated ≥16V — none exist.
- **Minimal fix idea**: Change footprint to 0805 or larger; use a 10µF 16V or 25V rated cap.

---

## HT-008
- **ID**: HT-008
- **Domain**: Schematic
- **Severity**: Major
- **Status**: Confirmed
- **Location**: C5 (100nF), GPS RF signal path, net `Net-(U3-RF_IN)` / `Net-(C5-Pad2)`
- **What's wrong**: C5 (100nF) is in series with the GPS RF signal path (between MAX2679 LNA output via L1 and NEO-M8N RF_IN). At GPS L1 frequency (1.575 GHz), 100nF has impedance of ~0.001Ω — far below the 50Ω system impedance. Its self-resonant frequency is in the low MHz range; at 1.575 GHz it behaves inductively and unpredictably.
- **Evidence**: Netlist: `Net [Net-(U3-RF_IN)]: C5.1(), U3.11(RF_IN)` and `Net [Net-(C5-Pad2)]: C5.2(), L1.1(1)`. C5 value: 100nF. Standard GPS DC-blocking caps are 10-100pF.
- **Why it matters**: GPS signal severely attenuated or distorted at RF frequency. GPS may fail to acquire satellites or have very poor sensitivity.
- **Quick verification**: Calculate C5 impedance at 1.575 GHz: Z = 1/(2π×1.575e9×100e-9) ≈ 0.001Ω. Compare to required ~50Ω matching.
- **Minimal fix idea**: Replace C5 with 10-100pF (e.g., 47pF C0G/NP0).

---

## HT-009
- **ID**: HT-009
- **Domain**: Firmware
- **Severity**: Fatal
- **Status**: Confirmed
- **Location**: `firmware/firmware.ino`, line 144, function `loop()`
- **What's wrong**: `uint64_t output_mask = 1 << i;` — the literal `1` is a 32-bit `int`. When `i ≥ 32`, the shift `1 << i` is undefined behavior in C/C++ (shift amount equals or exceeds width of type). Same bug on line 152: `(values & (1 << j))`.
- **Evidence**: Line 144: `uint64_t output_mask = 1 << i;` with `i` ranging 0..39 (NUM_HARNESS_PINS=40). For i=32..39, `1 << i` is UB. The literal `1` is `int` (32-bit on ARM). Line 152: `(1 << j)` with j in 0..39 — same issue.
- **Why it matters**: Pins 32-39 (upper 8 harness pins) will not be tested correctly. The output_mask and comparison mask will be wrong (typically wraps to low bits). Harness testing gives incorrect results for ~20% of pins.
- **Quick verification**: Note `1` is `int` (32-bit). `1 << 32` is UB per C++ standard [expr.shift]. Should be `(uint64_t)1 << i` or `1ULL << i`.
- **Minimal fix idea**: Change `1 << i` to `1ULL << i` on lines 144 and 152.

---

## HT-010
- **ID**: HT-010
- **Domain**: Firmware
- **Severity**: Fatal
- **Status**: Confirmed
- **Location**: `firmware/firmware.ino`, line 138, function `loop()`
- **What's wrong**: `if (digitalRead(PIN_BTN_TEST) == LOW) return;` — with R4 (10kΩ) pull-up to 3.3V, the button reads HIGH when not pressed and LOW when pressed. The code returns (skips test) when LOW (pressed) and proceeds when HIGH (not pressed). This is inverted logic.
- **Evidence**: Line 138: `if (digitalRead(PIN_BTN_TEST) == LOW) return;`. Schematic: R4 (10k) pulls BTN_TEST to +3.3V; SW1 shorts BTN_TEST to GND when pressed. Comment on line 137 says "Start testing only if the button is pressed" but code does the opposite.
- **Why it matters**: The test runs continuously when the button is NOT pressed, and STOPS when the button IS pressed. Completely inverted user interface.
- **Quick verification**: Read line 138 and the pull-up configuration. LOW=pressed → return (skip). HIGH=released → test runs.
- **Minimal fix idea**: Change to `if (digitalRead(PIN_BTN_TEST) == HIGH) return;`.

---

## HT-011
- **ID**: HT-011
- **Domain**: Firmware
- **Severity**: Fatal
- **Status**: Confirmed
- **Location**: `firmware/firmware.ino`, function `setup()` (lines 97-116)
- **What's wrong**: The `cy.begin()` method is never called. The CY8C9560 object `cy` is constructed at line 63 but `setup()` never calls `cy.begin()`, which initializes Wire2 I2C, resets the device, and verifies device ID.
- **Evidence**: Search `setup()` (lines 97-116) — no call to `cy.begin()`. The `begin()` method (CY8C9560.cpp:3-15) calls `WIRE.begin()`, `WIRE.setClock()`, resets the IC, and reads the device ID. Without it, `Wire2` is never initialized.
- **Why it matters**: All I2C communication with the CY8C9560A will fail. The harness testing subsystem is completely non-functional.
- **Quick verification**: Grep for `cy.begin` in setup() — not present.
- **Minimal fix idea**: Add `cy.begin();` in `setup()` after GPIO initialization.

---

## HT-012
- **ID**: HT-012
- **Domain**: Firmware
- **Severity**: Fatal
- **Status**: Confirmed
- **Location**: `firmware/CY8C9560.cpp`, function `begin()`, lines 6-8
- **What's wrong**: Reset sequence is inverted. The code drives RESET_N HIGH (line 6, releasing reset), waits 10ms, then drives LOW (line 8, asserting reset), waits 100ms, then immediately tries I2C communication. The device is held in active reset during communication.
- **Evidence**: Lines 6-9: `digitalWrite(CY_RST, HIGH); delay(10); digitalWrite(CY_RST, LOW); delay(100);`. RESET_N is active-low: LOW=reset, HIGH=normal. After this sequence, the pin is LOW → device in reset → I2C read_id() on line 14 will fail.
- **Why it matters**: Even if `cy.begin()` were called, the CY8C9560A would remain in reset and not respond to any I2C commands.
- **Quick verification**: Read begin() — final state of CY_RST is LOW (reset asserted). Should end HIGH.
- **Minimal fix idea**: Swap the sequence: drive LOW first (assert reset), then HIGH (release reset).

---

## HT-013
- **ID**: HT-013
- **Domain**: Firmware
- **Severity**: Major
- **Status**: Confirmed
- **Location**: `firmware/firmware.ino`, lines 118-131, function `loop()`
- **What's wrong**: NMEA buffer `nmea_buf[64]` has no bounds checking. `nmea_idx` is incremented without limit. If a line exceeds 63 characters before CR/LF, the write `nmea_buf[nmea_idx++]` overflows the buffer. Maximum NMEA sentence is 82 characters per the standard, exceeding this 64-byte buffer.
- **Evidence**: Line 118: `char nmea_buf[64];`. Line 123: `nmea_buf[nmea_idx++] = UBX_SERIAL.read();` — no check `if (nmea_idx >= sizeof(nmea_buf))`. NMEA 0183 allows sentences up to 82 chars ($....*XX\r\n).
- **Why it matters**: Buffer overflow corrupts adjacent stack/global memory. Can cause crashes, data corruption, or unpredictable behavior.
- **Quick verification**: Note buffer is 64 bytes; NMEA allows 82 chars. No bounds check before index increment.
- **Minimal fix idea**: Add `if (nmea_idx >= sizeof(nmea_buf) - 1) { nmea_idx = 0; continue; }` before the write.

---

## HT-014
- **ID**: HT-014
- **Domain**: Firmware
- **Severity**: Major
- **Status**: Confirmed
- **Location**: `firmware/firmware.ino`, lines 67-71, function `set_status()`; lines 97-116, function `setup()`
- **What's wrong**: LED GPIO pins (PIN_LED_R=5, PIN_LED_G=6, PIN_LED_B=7) are never configured as OUTPUT via `pinMode()`. The `set_status()` function calls `digitalWrite()` on these pins, but without `pinMode(pin, OUTPUT)`, the pins remain as inputs. Per Arduino/Teensy documentation, `digitalWrite()` on an INPUT pin only toggles the internal pull-up resistor (~37kΩ on Teensy 4.1) rather than driving the pin.
- **Evidence**: Search `setup()` for `pinMode(PIN_LED_R` or `PIN_LED_G` or `PIN_LED_B` — not found. `set_status()` is called at line 103 before any LED pin mode is set. Per PJRC documentation and Arduino reference: `digitalWrite(HIGH)` on INPUT → enables pull-up; `digitalWrite(LOW)` on INPUT → disables pull-up.
- **Why it matters**: LEDs receive only ~90µA through the 37kΩ pull-up (if HIGH) → invisible or barely visible. No meaningful status indication for the user.
- **Quick verification**: Grep firmware.ino for `pinMode.*LED` — no matches. Check Arduino docs on `digitalWrite` without `pinMode(OUTPUT)`.
- **Minimal fix idea**: Add `pinMode(PIN_LED_R, OUTPUT); pinMode(PIN_LED_G, OUTPUT); pinMode(PIN_LED_B, OUTPUT);` in setup().

---

## HT-015
- **ID**: HT-015
- **Domain**: Firmware
- **Severity**: Fatal
- **Status**: Confirmed
- **Location**: `firmware/firmware.ino`, lines 142-157, function `loop()`
- **What's wrong**: Harness pass/fail logic is inverted. `passed` is initialized to `false` (line 142) and set to `true` if ANY single pin matches (line 155-157). It is never set back to `false` on mismatch. The correct logic should start `true` and set `false` if any pin fails.
- **Evidence**: Line 142: `bool passed = false;`. Lines 155-157: `if (values == EXPECTED_CONNECTIONS[i]) { passed = true; }`. No `else { passed = false; }`. If pin 0 matches but pins 1-39 all fail, result is still "passed".
- **Why it matters**: Defective harnesses will be reported as "passed" if even one pin happens to match. Complete invalidation of the testing purpose.
- **Quick verification**: Read the loop logic — confirm `passed` is only ever set true, never false during iteration.
- **Minimal fix idea**: Initialize `passed = true;` and add `else { passed = false; break; }` after the comparison.

---

## HT-016
- **ID**: HT-016
- **Domain**: Firmware
- **Severity**: Minor
- **Status**: Confirmed
- **Location**: `firmware/firmware.ino`, lines 106-107, function `setup()`
- **What's wrong**: GPIO pins for UBX_SAFEBOOT (pin 3) and UBX_RST_N (pin 4) are driven with `digitalWrite()` but never configured as OUTPUT via `pinMode()`. Without `pinMode(pin, OUTPUT)`, `digitalWrite()` only toggles the internal pull-up, not the output driver.
- **Evidence**: Line 106: `digitalWrite(PIN_UBX_SAFEBOOT, LOW);` Line 107: `digitalWrite(PIN_UBX_RST_N, HIGH);`. No preceding `pinMode(..., OUTPUT)` anywhere. Per Arduino/Teensy docs, this toggles pull-ups on INPUT pins rather than driving them.
- **Why it matters**: SAFEBOOT: `digitalWrite(LOW)` on INPUT disables pull-up → pin floats → relies on NEO-M8N internal pull-up to stay HIGH (normal mode). RST_N: `digitalWrite(HIGH)` on INPUT enables weak pull-up → weak HIGH. Pins are not firmly driven as intended. Works by accident due to GPS module internal pull-ups, but fragile.
- **Quick verification**: Grep for `pinMode.*SAFEBOOT` or `pinMode.*RST_N` — no matches.
- **Minimal fix idea**: Add `pinMode(PIN_UBX_SAFEBOOT, OUTPUT);` and `pinMode(PIN_UBX_RST_N, OUTPUT);` before the digitalWrite calls.

---

## HT-017
- **ID**: HT-017
- **Domain**: PCB
- **Severity**: Major
- **Status**: Confirmed
- **Location**: +3.3V net, F.Cu layer, near coordinates (167.3, 35.8) and (158.1, 32.9)
- **What's wrong**: Broken +3.3V power trace — two track segments on F.Cu that should be connected are not joined, creating a gap in the +3.3V power delivery.
- **Evidence**: DRC unconnected item: `"Missing connection between items"` — `Track [+3.3V] on F.Cu, length 0.6845 mm` at (167.32, 35.77) and `Track [+3.3V] on F.Cu, length 0.8556 mm` at (158.1, 32.9). This is the only unrouted net in the DRC.
- **Why it matters**: Some components on the +3.3V rail may not receive power, causing partial or complete system failure depending on which branch is disconnected.
- **Quick verification**: Run DRC → 1 unconnected item on +3.3V net.
- **Minimal fix idea**: Add a trace connecting the two +3.3V track segments.

---

## HT-018
- **ID**: HT-018
- **Domain**: PCB
- **Severity**: Minor
- **Status**: Confirmed
- **Location**: C6 and U5 footprints
- **What's wrong**: Courtyard overlap between C6 (0402 capacitor) and U5 (MAX2679 WLP-4 BGA). Component courtyards intersect, indicating they are placed too close together.
- **Evidence**: DRC: `"Courtyards overlap" — Footprint C6, Footprint U5`.
- **Why it matters**: Assembly risk — pick-and-place may have difficulty placing components. Rework/inspection access compromised. Potential solder bridging between the two parts.
- **Quick verification**: Run PCB DRC → courtyard overlap reported for C6 and U5.
- **Minimal fix idea**: Move C6 slightly to eliminate courtyard overlap.

---

## HT-019
- **ID**: HT-019
- **Domain**: PCB
- **Severity**: Major
- **Status**: Confirmed
- **Location**: Multiple layers (F.Cu, In1.Cu, In2.Cu, B.Cu); 500 clearance violations
- **What's wrong**: 500 trace-to-zone clearance violations across the board. The actual clearance (0.1505 mm) is below the design rule minimum (0.2000 mm). Most violations are between signal traces (CBL_xx, LED, UART, I2C) and copper pour zones (+3.3V on In1.Cu, GND on F.Cu/In2.Cu/B.Cu).
- **Evidence**: DRC: `500 error: clearance` violations. Example: `"Clearance violation (netclass 'Default' clearance 0.2000 mm; actual 0.1505 mm)"`. Top offenders: +3.3V vs GND (164), GND vs signal traces, +3.3V vs CBL signals.
- **Why it matters**: PCB will fail manufacturing DRC at most board houses. Risk of copper shorts between signal traces and power/ground planes, causing random failures.
- **Quick verification**: Run DRC → 500 clearance errors.
- **Minimal fix idea**: Re-route traces with proper clearance or adjust zone pour clearance rules if acceptable for the stackup.

---

## HT-020
- **ID**: HT-020
- **Domain**: Firmware
- **Severity**: Major
- **Status**: Confirmed
- **Location**: `firmware/CY8C9560.cpp`, function `set_output()`, lines 77-84
- **What's wrong**: `set_output()` sets ALL port pins as outputs (`write_register(REG_PIN_DIRECTION, 0x00)` for all 8 ports) before `set_pd_inputs()` reconfigures non-selected pins as inputs. During this window, all 60+ I/O expander pins briefly drive their output register values with strong drive mode. The output register has the selected pin HIGH and all others LOW, so connected harness pins are momentarily shorted (one driven HIGH, another driven LOW through the harness wire).
- **Evidence**: Lines 80-82: `for (int i=0; i<8; i++) { write_register(REG_PORT_SELECT, i); write_register(REG_PIN_DIRECTION, 0x00); ... }`. Direction 0x00 = all outputs.
- **Why it matters**: Brief contention current through harness connections each test cycle. Repeated stress may damage the I/O expander pins or produce unreliable readings.
- **Quick verification**: Read set_output() — note PIN_DIRECTION=0x00 (all output) for every port, not just the target pin.
- **Minimal fix idea**: Only set the target pin's port direction bit as output; keep others as inputs.

---

## HT-021
- **ID**: HT-021
- **Domain**: Schematic
- **Severity**: Major
- **Status**: Confirmed
- **Location**: C1 (1µF), footprint `Capacitor_SMD:C_0402_1005Metric`, net `+12V`
- **What's wrong**: C1 (1µF) is on the +12V rail in an 0402 package. Standard 0402 1µF MLCCs are rated for 6.3V or 10V maximum. At 12V operating voltage, the capacitor exceeds its voltage rating, causing severe capacitance derating (to near zero) and risk of dielectric breakdown.
- **Evidence**: Component: `C1: 1u, fp=Capacitor_SMD:C_0402_1005Metric`, Net: `+12V: C1.1(), ...`. 0402 1µF caps from major manufacturers (Murata, Samsung, TDK) max out at 6.3-10V rated voltage.
- **Why it matters**: Capacitor provides negligible filtering at 12V due to voltage derating. Risk of cap failure (short circuit or open). Inadequate input filtering for the L7805 regulator.
- **Quick verification**: Look up 0402 1µF capacitor voltage ratings from any manufacturer catalog.
- **Minimal fix idea**: Use an 0805 or larger package with ≥16V rating.

---

## HT-022
- **ID**: HT-022
- **Domain**: Firmware
- **Severity**: Major
- **Status**: Confirmed
- **Location**: `firmware/firmware.ino`, line 74, function `process_nmea()`
- **What's wrong**: `buf[len] = 0;` writes a null terminator at index `len`. When nmea_buf is read up to index 63 (the last valid position) and that byte is a newline, `nmea_idx` is 64 when `process_nmea(nmea_buf, 64)` is called. Then `buf[64] = 0` writes one byte past the end of the 64-byte buffer.
- **Evidence**: Line 74: `buf[len] = 0;`. Buffer declared at line 118: `char nmea_buf[64]`. Trace: after writing nmea_buf[63] at line 123, nmea_idx becomes 64. If nmea_buf[63] is '\n', process_nmea is called with len=64. `buf[64] = 0` is out of bounds.
- **Why it matters**: Off-by-one write past buffer end — corrupts adjacent global variable memory (nmea_idx and other globals).
- **Quick verification**: Trace the index: write to buf[63] → nmea_idx=64 → newline check → process_nmea(buf, 64) → buf[64]=0 is OOB.
- **Minimal fix idea**: Use `buf[len-1] = 0;` to null-terminate before the newline, or add bounds check in the read loop.

---

## HT-023
- **ID**: HT-023
- **Domain**: PCB
- **Severity**: Major
- **Status**: Confirmed
- **Location**: In1.Cu (inner layer 1), +3.3V copper pour zone
- **What's wrong**: The +3.3V copper zone on In1.Cu is fragmented into 12 isolated copper islands. Signal traces routed through this layer break the pour into disconnected fragments that have no connection to the actual +3.3V power net.
- **Evidence**: DRC: 12 entries of `"isolated_copper: Zone [+3.3V] on In1.Cu"`. Plus 20 isolated GND zone fragments. The CBL_xx signal traces routed on In1.Cu slice through the +3.3V pour, creating dead copper.
- **Why it matters**: Isolated copper islands carry no useful current but can couple parasitically to signals, cause EMI issues, and violate manufacturing quality standards. The fragmented +3.3V plane has poor power delivery characteristics.
- **Quick verification**: Run DRC → 12 isolated +3.3V zones on In1.Cu.
- **Minimal fix idea**: Re-route CBL_xx signals off In1.Cu, or remove the pour from areas that cannot maintain continuity. Add vias to connect islands.
