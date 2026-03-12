# Harness Tester Challenge — Bug Report

**Repository analyzed:** [commaai/harness_tester_challenge](https://github.com/commaai/harness_tester_challenge)  
**Date:** 2026-03-12  
**Analyst:** Josh Leopard  
**Total bugs found:** 20

---

## Overview

The harness tester challenge consists of three components, each containing intentional bugs:

1. **Firmware** (`firmware/firmware.ino`) — Arduino/Teensy logic
2. **CY8C9560 I2C IO Expander Driver** (`firmware/CY8C9560.cpp` / `CY8C9560.h`)
3. **Schematic** (`schematic.pdf`) — PCB hardware design

---

## 🔴 Firmware Bugs (`firmware/firmware.ino`)

---

### F1 — `1 << i` 32-bit integer overflow for pins 32–39 ⚠️ SHOW-STOPPER

**Location:** `loop()`, test harness section

```cpp
// BUGGY
uint64_t output_mask = 1 << i;

// FIXED
uint64_t output_mask = 1ULL << i;
```

**Explanation:**  
The literal `1` is a signed 32-bit `int`. For `i >= 32`, left-shifting a 32-bit int by 32 or more bits is **undefined behavior** in C/C++, and in practice produces 0 or garbage. Since `NUM_HARNESS_PINS = 40`, pins 32–39 will generate a completely wrong (or zero) `output_mask`, meaning those 8 pins are never correctly driven during testing.

**Fix:** Use `1ULL` (unsigned 64-bit literal).

---

### F2 — Same `1 << j` overflow in the serial debug print loop

**Location:** `loop()`, debug serial output

```cpp
// BUGGY
for (int j = 0; j < NUM_HARNESS_PINS; j++)
    DBG_SERIAL.printf("%d", (values & (1 << j)) ? 1 : 0);

// FIXED
DBG_SERIAL.printf("%d", (values & (1ULL << j)) ? 1 : 0);
```

**Explanation:**  
Identical overflow bug to F1. For `j >= 32`, `1 << j` overflows, producing corrupted debug output for pins 32–39.

---

### F3 — Pass/fail logic uses OR instead of AND ⚠️ SHOW-STOPPER

**Location:** `loop()`, test result logic

```cpp
// BUGGY
bool passed = false;
for (int i = 0; i < NUM_HARNESS_PINS; i++) {
    ...
    if (values == EXPECTED_CONNECTIONS[i]) {
        passed = true;  // set true on ANY single match
    }
}

// FIXED
bool passed = true;
for (int i = 0; i < NUM_HARNESS_PINS; i++) {
    ...
    if (values != EXPECTED_CONNECTIONS[i]) {
        passed = false;  // set false on ANY mismatch
        break;
    }
}
```

**Explanation:**  
The intent is to verify that ALL 40 pins have the correct connections. The current logic sets `passed = true` the moment *any single pin* matches expectations — meaning a harness with 39 wrong pins and 1 correct pin would be reported as **PASSED**. The logic must be inverted: start `true` and fail on any mismatch.

---

### F4 — `EXPECTED_CONNECTIONS` self-bits always masked out, so match is impossible ⚠️ SHOW-STOPPER

**Location:** `EXPECTED_CONNECTIONS` definition + `loop()`

```cpp
// Each row has the pin's OWN bit set, e.g. pin 0:
// 0b1000000000000000000000000010000000000000
//   ^ bit 39 = pin 0's own position

// But the read masks out the driven pin:
uint64_t values = cy.read_inputs() & ~output_mask;
// This clears pin i's bit, so values can NEVER equal EXPECTED_CONNECTIONS[i]
```

**Explanation:**  
Every row in `EXPECTED_CONNECTIONS` has the pin's own bit set (pin 0's row has bit 39 set, etc.). However, the firmware always clears the driven pin's bit from the read result to avoid self-detection. The expected and actual values are structurally incompatible — `values` will never equal `EXPECTED_CONNECTIONS[i]` for any pin, so the tester **always reports failure** regardless of harness state.

**Fix:** Remove self-bits from `EXPECTED_CONNECTIONS`, or stop masking the driven pin's bit from the comparison value.

---

### F5 — Bit ordering in `EXPECTED_CONNECTIONS` is reversed vs. mask generation ⚠️ SHOW-STOPPER

**Location:** `EXPECTED_CONNECTIONS` literals vs. `loop()`

```cpp
// Matrix written MSB-first (pin 0 = leftmost = bit 39):
// EXPECTED_CONNECTIONS[0] = 0b1000000000000000000000000010000000000000
//                                                          ^ this is bit 6, not bit 33

// But masks are generated LSB-first (pin 0 = bit 0):
uint64_t output_mask = 1ULL << i;  // pin 0 = bit 0
```

**Explanation:**  
The binary literals in `EXPECTED_CONNECTIONS` are written with pin 0 in the most-significant-bit position (the leftmost digit). But `1ULL << i` places pin 0 at bit 0 (LSB). These two conventions are **completely reversed** — the expected connection matrix and the actual I/O operations operate on mirrored bit orderings, so even if bugs F3 and F4 were fixed, the comparison would still be wrong for almost every pin.

---

### F6 — `cy.begin()` is never called ⚠️ SHOW-STOPPER

**Location:** `setup()`

```cpp
// setup() initializes serial, GPIO, and SD — but never:
cy.begin();   // <-- THIS IS MISSING

// begin() is what starts Wire2, resets the chip, and verifies the device ID
```

**Explanation:**  
`CY8C9560::begin()` initializes the `Wire2` I2C bus, pulses the reset line, and reads the device ID register to confirm the chip is alive. Without this call, the I2C peripheral is never started, and every subsequent `cy.set_output()`, `cy.set_pd_inputs()`, and `cy.read_inputs()` call silently fails over an uninitialized bus.

---

### F7 — GPS control pins never configured as OUTPUT ⚠️ SHOW-STOPPER

**Location:** `setup()`

```cpp
// BUGGY — no pinMode for these two pins:
digitalWrite(PIN_UBX_SAFEBOOT, LOW);
digitalWrite(PIN_UBX_RST_N, HIGH);

// FIXED — add these before the digitalWrite calls:
pinMode(PIN_UBX_SAFEBOOT, OUTPUT);
pinMode(PIN_UBX_RST_N, OUTPUT);
```

**Explanation:**  
`PIN_UBX_SAFEBOOT` (pin 3) and `PIN_UBX_RST_N` (pin 4) have no `pinMode()` call. Arduino/Teensy pins default to `INPUT` mode. Calling `digitalWrite()` on an `INPUT` pin does not drive the pin — it only controls the internal pull-up resistor. The GPS module's SAFEBOOT and RESET lines are never actually driven to the correct state, so the GPS may fail to start properly.

---

### F8 — `set_pd_inputs()` is immediately overridden by `set_output()` ⚠️ SHOW-STOPPER

**Location:** `loop()`, test loop body

```cpp
cy.set_output(output_mask, output_mask);  // sets ALL 64 pins as outputs
cy.set_pd_inputs(~output_mask);           // sets inputs... but called AFTER set_output!

// Wait, the actual order in the code is:
cy.set_output(output_mask, output_mask);   // line 1: all 64 pins → strong output
cy.set_pd_inputs(~output_mask);            // line 2: tries to set 63 pins as pull-down inputs
```

**Explanation:**  
`set_output()` writes `REG_PIN_DIRECTION = 0x00` (all outputs) for **all 8 ports** (all 64 pins). `set_pd_inputs()` then tries to reconfigure the other 63 pins as pull-down inputs — but it also writes `REG_PIN_DIRECTION = 0xFF` (all inputs) for all 8 ports, which *again* sets the driven pin back to an input. The two functions fight each other, and the net result is that pins are never in the right state for measurement. The order must be: set all others as pull-down inputs first, then set the driven pin as output.

---

### F9 — NMEA buffer overflow — no bounds check

**Location:** `loop()`, NMEA processing

```cpp
char nmea_buf[64];
int nmea_idx = 0;

// BUGGY — no guard:
nmea_buf[nmea_idx++] = UBX_SERIAL.read();

// FIXED:
if (nmea_idx < sizeof(nmea_buf) - 1) {
    nmea_buf[nmea_idx++] = UBX_SERIAL.read();
} else {
    nmea_idx = 0;  // reset on overflow
}
```

**Explanation:**  
GPS NMEA sentences are typically under 82 characters but can technically be longer. More importantly, any malformed or unexpected serial data could overflow `nmea_buf[64]`, corrupting the stack.

---

### F10 — Off-by-one out-of-bounds write in `process_nmea()`

**Location:** `process_nmea()`

```cpp
void process_nmea(char *buf, int len) {
    buf[len] = 0;  // null-terminate
    // If len == 64 (full buffer), buf[64] is out of bounds!
}
```

**Explanation:**  
If the buffer fills completely (`nmea_idx == 64`), `process_nmea` is called with `len = 64`, writing a null byte to `buf[64]` — one past the end of the 64-byte array. This is an out-of-bounds write.

---

### F11 — CRLF line endings cause `process_nmea` to fire twice per sentence

**Location:** `loop()`, NMEA processing

```cpp
if (nmea_buf[nmea_idx - 1] == '\n' || nmea_buf[nmea_idx - 1] == '\r') {
    process_nmea(nmea_buf, nmea_idx);
    nmea_idx = 0;
```

**Explanation:**  
NMEA sentences end with `\r\n`. The `||` condition triggers `process_nmea` on `\r`, resets `nmea_idx = 0`, then triggers it again on `\n` (with `len = 1`, containing only the newline character). The second call is mostly harmless but parses garbage input and generates a spurious "Failed to parse $GPRMC" message every sentence.

---

## 🔴 CY8C9560 Driver Bugs (`firmware/CY8C9560.cpp` / `.h`)

---

### D1 — `begin()` reset sequence leaves chip permanently in reset ⚠️ SHOW-STOPPER

**Location:** `CY8C9560::begin()`

```cpp
// BUGGY — ends with RESET_N LOW (asserted = chip in reset):
digitalWrite(CY_RST, HIGH);  // briefly deassert
delay(10);
digitalWrite(CY_RST, LOW);   // re-assert — chip stays in reset forever

// FIXED — pulse LOW to reset, then bring HIGH to operate:
digitalWrite(CY_RST, LOW);   // assert reset
delay(10);
digitalWrite(CY_RST, HIGH);  // deassert — chip now operational
delay(100);
```

**Explanation:**  
`RESET_N` on the CY8C9560A is **active-low**: 0V = in reset, 3.3V = normal operation. The current code ends the sequence with the pin held LOW, meaning the IC is permanently held in reset and never becomes operational. Every I2C transaction that follows will receive no response.

---

### D2 — `set_output()` turns all 64 I/O expander pins into outputs ⚠️ SHOW-STOPPER

**Location:** `CY8C9560::set_output()`

```cpp
void CY8C9560::set_output(uint64_t pins, uint64_t values) {
    write_registers(REG_OUTPUT_BASE, values, 8);
    for (int i = 0; i < 8; i++) {
        write_register(REG_PORT_SELECT, i);
        write_register(REG_PIN_DIRECTION, 0x00);  // ALL pins in port = output
        // 'pins' mask is only used for drive mode, NOT for direction!
        write_register(REG_PIN_DRIVE_MODE_BASE + DRIVE_MODE_STRONG, pins >> (i * 8));
    }
}
```

**Explanation:**  
`REG_PIN_DIRECTION` is hardcoded to `0x00` (all outputs) for every port, completely ignoring the `pins` mask parameter. When testing pin `i`, all 63 other harness pins are also driven as strong outputs. Any pins that are shorted together in the harness will experience **bus contention** between two strong-drive outputs, potentially damaging the PCB or the harness under test.

**Fix:** Use the `pins` mask to selectively set direction: `write_register(REG_PIN_DIRECTION, ~(uint8_t)(pins >> (i * 8)))` (0 = output, 1 = input per the CY8C9560A datasheet).

---

### D3 — Wrong device ID check in `begin()`

**Location:** `CY8C9560::begin()` and `read_id()`

```cpp
// BUGGY:
return read_id() == 0x06;

// FIXED (CY8C9560A family ID = 0x04 per datasheet):
return read_id() == 0x04;
```

**Explanation:**  
The `DEVICE_ID_STATUS` register bits [7:4] contain the device family ID. For the CY8C9560**A**, this value is `0x04`, not `0x06`. `begin()` will always return `false` even on a correctly functioning chip, and any initialization check relying on this return value will fail.

---

## 🔴 Schematic Bugs

---

### S1 — SiSS27DN is an N-channel MOSFET where a P-channel is required ⚠️ SHOW-STOPPER

**Component:** Q1 (`SiSS27DN`) in the reverse polarity protection circuit

**Explanation:**  
The design places Q1 as a high-side series switch on the +12V input for reverse polarity protection. This topology requires a **P-channel** MOSFET (gate pulled to source when powered correctly → FET conducts; reversed polarity → gate floats high → FET off). The **SiSS27DN is an N-channel device**. An N-channel FET in the high-side position with gate referenced to ground cannot be properly gated from the 12V rail without a charge pump. In practice:
- Normal polarity: gate ≈ 0V, source ≈ 12V → Vgs ≈ −12V → **FET off, board doesn't power on**
- Reversed polarity: no protection either

The part must be replaced with a P-channel MOSFET (e.g., SiSS78DN, Si2333DS, or similar 20V+ rated PMOS).

---

### S2 — RGB LED has no current-limiting resistors ⚠️ SHOW-STOPPER

**Component:** D3 (`ASMB-KTF0-0A306`)

**Explanation:**  
The ASMB-KTF0-0A306 is a common-anode RGB LED with its anode tied to +3.3V. The three cathodes (R, G, B) connect directly to Teensy GPIO pins 5, 6, and 7 with **no series resistors**. With a 3.3V supply and a forward voltage of ~2.0–3.3V depending on color, the current is limited only by the GPIO output impedance (~25–40Ω), resulting in 0–65mA per channel — far exceeding both the LED's rated current (~20mA typ) and the Teensy 4.1's GPIO source/sink rating (10mA per pin, 120mA total). This will damage the LED and/or the MCU.

**Fix:** Add series resistors (e.g., 68–100Ω for each channel at 3.3V).

---

### S3 — L7805 voltage regulator missing output bypass capacitor

**Component:** U1 (`L7805`)

**Explanation:**  
The L7805 datasheet requires a minimum 0.1µF capacitor on the output pin for stability. Input-side capacitance is covered by `C7` (10µF) and `C1` (1µF). However, there is **no output-side bypass capacitor** on the +5V rail. Without it, the L7805 can oscillate, producing a noisy or unstable 5V supply.

**Fix:** Add ≥100nF ceramic capacitor from the L7805 OUT pin to GND.

---

### S4 — GPS V_BCKP (backup power) pin unconnected

**Component:** U3 (`NEO-M8N`), pin 22

**Explanation:**  
The NEO-M8N pin 22 (`V_BCKP`) is a backup supply input that retains the satellite almanac and ephemeris data across power cycles, enabling fast warm/hot starts. If this pin is left unconnected or floating, **every power cycle is a cold start**, adding 30–60+ seconds to GPS acquisition time. For a tester that relies on GPS time for log timestamps, this significantly degrades usability.

**Fix:** Connect `V_BCKP` to a small always-on supply (coin cell, supercap, or a low-dropout from the main rail through a diode).

---

### S5 — GPS bias-T inductor L1 = 12nH is far too small for 1.575 GHz

**Component:** L1 (12nH) in the active antenna bias-T network

**Explanation:**  
The bias-T supplies DC power to the active GPS antenna through the RF coax. The inductor L1 serves as an RF choke, blocking GPS-frequency signals from entering the DC supply rail. At 1.575 GHz:

```
Z = 2π × 1.575 GHz × 12 nH ≈ 118 Ω
```

118Ω is far too low to effectively block RF. A proper bias-T choke should present thousands of ohms at the operating frequency (typically 1µH–10µH, giving 10–100 kΩ). With only 118Ω of impedance, GPS signal energy bleeds into the supply rail, degrading receiver sensitivity and potentially injecting noise back into other circuitry.

**Fix:** Replace L1 with a 1µH–10µH high-frequency ferrite chip inductor rated for 1.5+ GHz operation.

---

### S6 — Duplicate U1 schematic symbol (second L7805 instance)

**Component:** U1 (`L7805`) — appears twice in schematic

**Explanation:**  
The schematic contains two separate instances of U1 (L7805) — once in the main power section at the top of the sheet, and a second instance near the GPS/signal section at the bottom. This appears to be a hierarchical power symbol re-use error in KiCad that creates an ambiguous net assignment and could result in incorrect ERC/netlist output from the schematic tool.

---

## Summary Table

| ID | Area | Severity | Description |
|----|------|----------|-------------|
| F1 | Firmware | 🔴 Show-stopper | `1 << i` overflow — pins 32–39 never correctly driven |
| F2 | Firmware | 🟠 Functional | `1 << j` overflow in debug print loop |
| F3 | Firmware | 🔴 Show-stopper | OR pass/fail logic — any one match = pass |
| F4 | Firmware | 🔴 Show-stopper | Self-bits in matrix always masked → comparison never succeeds |
| F5 | Firmware | 🔴 Show-stopper | Bit ordering in `EXPECTED_CONNECTIONS` reversed vs. mask generation |
| F6 | Firmware | 🔴 Show-stopper | `cy.begin()` never called — I2C bus never initialized |
| F7 | Firmware | 🔴 Show-stopper | GPS SAFEBOOT/RST pins never set as OUTPUT |
| F8 | Firmware | 🔴 Show-stopper | `set_pd_inputs()` called in wrong order, immediately overridden |
| F9 | Firmware | 🟠 Reliability | NMEA buffer overflow — no bounds check |
| F10 | Firmware | 🟠 Reliability | Off-by-one OOB write in `process_nmea()` |
| F11 | Firmware | 🟡 Minor | CRLF causes `process_nmea` to fire twice per sentence |
| D1 | Driver | 🔴 Show-stopper | Reset sequence ends with chip held in reset |
| D2 | Driver | 🔴 Show-stopper | `set_output()` drives all 64 expander pins as outputs |
| D3 | Driver | 🟠 Functional | Wrong device ID check (0x06 vs correct 0x04) |
| S1 | Schematic | 🔴 Show-stopper | N-channel MOSFET Q1 used where P-channel required |
| S2 | Schematic | 🔴 Show-stopper | LED cathodes wired directly to GPIO — no current-limiting resistors |
| S3 | Schematic | 🟠 Functional | L7805 missing output bypass capacitor |
| S4 | Schematic | 🟠 Functional | GPS V_BCKP pin unconnected — always cold start |
| S5 | Schematic | 🟠 Functional | Bias-T choke L1 = 12nH, far too small for 1.575 GHz |
| S6 | Schematic | 🟡 Minor | Duplicate U1 symbol in schematic |

**Total: 20 bugs** — 11 show-stoppers, 7 functional/reliability issues, 2 minor
