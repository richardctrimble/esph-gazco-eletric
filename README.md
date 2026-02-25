# Gazco Electric Fireplace ESPHome Controller

[![GitHub release](https://img.shields.io/github/release/richardctrimble/esph-gazco-electric.svg)](https://github.com/richardctrimble/esph-gazco-electric/releases/)

ESP32/ESPHome implementation for controlling a Gazco electric fireplace via 433MHz OOK radio, replacing the original remote control.

## Hardware

- **TTGO LoRa32 v2.1** (ESP32 + SX1276/SX1278 built-in)
- **433MHz antenna**
- **`calibril.ttf`** font file required in the ESPHome config directory (for the OLED display)

## Fireplace Controls

| Control | Options | Notes |
|---------|---------|-------|
| LED Display | On / Off | Independent of heater |
| Heater | Off / Low / High | Independent of LED display |
| Flame Colour | 1–3 | |
| Flame Brightness | Off, 1–5 | |
| Fuel Bed Colour | White, Red, Amber, Yellow, Lime, Green, Cyan, Sky Blue, Blue, Lavender, Magenta, Bright Pink, Deep Pink, Cycle | Cycle = auto-rotate through colours |
| Fuel Bed Brightness | Off, 1–5 | |

## RF Protocol

### Overview

- **Frequency**: 433.92 MHz
- **Modulation**: OOK (On-Off Keying)
- **Encoding**: Mark-width PWM (data encoded in mark duration, spaces are constant)
- **Original Remote Hardware**: STM8L151 MCU + A7302 RF transceiver (AMICCOM)
- **Security**: Fixed codes (not rolling code)

### Timing

| Element | Mark Width | Space Width |
|---------|-----------|-------------|
| Preamble | ~480 µs | ~470 µs |
| Data '0' | ~280 µs | ~470 µs |
| Data '1' | ~670 µs | ~470 µs |

- 10 preamble pulses before each packet
- ~10 ms inter-packet gap
- 6 repeats per transmission

### Packet Structure (64 bits)

```
[Preamble ×10] [Unit ID 16b] [B0] [B1] [B2] [B3] [B4 CHK] [B5 ~CHK]
```

| Field | Bits | Description |
|-------|------|-------------|
| Preamble | 10 marks | Wake-up / sync |
| Unit ID | 16 | Remote's unique address (e.g. 0x0FD0) |
| B0 | 8 | Command + Flame Colour + Flame Brightness |
| B1 | 8 | Fuel Bed Colour (offset) |
| B2 | 8 | Fuel Bed Brightness + Heater Level |
| B3 | 8 | Reserved (always 0x00) |
| B4 | 8 | Checksum |
| B5 | 8 | Bitwise complement of checksum |

### Byte Encoding

**B0** = `((Cmd × 8 + FC) << 3) | FB`
- **Cmd** (2-bit field): bit 0 = LED on/off, bit 1 = Heater enabled
  - `0` = Both off
  - `1` = LED only
  - `2` = Heater only (no LED)
  - `3` = Both LED and Heater
- **FC**: Flame Colour (1–3)
- **FB**: Flame Brightness (0 = Off, 1–5)

**B1** = `UC + 16`
- **UC**: Fuel Bed Colour (1–13, 14 = Cycle)

**B2** = `(UB << 2) | HL`
- **UB**: Fuel Bed Brightness (0 = Off, 1–5) — upper 6 bits
- **HL**: Heater Level (0 = Off, 1 = Low, 2 = High) — lower 2 bits

**B3** = `0x00` (reserved)

### Checksum

```
B4 = (ID_hi + ID_lo + B0 + B1 + B2 + B3 + 0x55) & 0xFF
B5 = ~B4 & 0xFF
```

The checksum seed `0x55` was determined empirically. B5 is the bitwise complement of B4, providing error detection.

### Example Packets

**LED On, Heater Low, FC=2, FB=3, UC=10, UB=4:**
```
Cmd = 3 (LED|Heater), B0 = ((3×8+2)<<3)|3 = 0xD3
B1 = 10+16 = 0x1A, B2 = (4<<2)|1 = 0x11, B3 = 0x00
Checksum = (0x0F+0xD0+0xD3+0x1A+0x11+0x00+0x55) & 0xFF
```

**All Off:**
```
Cmd = 0, B0 = ((0×8+FC)<<3)|0, B2 lower 2 = 0
```

## Receiver

The decoder uses a "first-short-mark" strategy for preamble detection:

1. Scan marks from the start — marks ≥ 320 µs are preamble
2. First mark < 320 µs signals end of preamble, start of data
3. Data marks: < 450 µs = `0`, ≥ 450 µs = `1`
4. Extract 64 data bits, verify checksum

Mark-width thresholds from real remote captures:
- Preamble marks: 333–507 µs
- Data '0' marks: 120–307 µs
- Data '1' marks: 560–707 µs

### Debug Level

Controls receiver output verbosity:

- **Valid Only**: checksum-verified packets only
- **All Packets**: all decoded packets regardless of checksum
- **Verbose**: raw pulse data, timing pairs, and binary dumps

## Unit ID Configuration

The Unit ID identifies your fireplace's remote. To learn it:

1. Set **Debug Level** to "All Packets" or "Verbose"
2. Press a button on the real remote near the antenna
3. Read the ID from the log output (e.g. `ID:0x0FD0`)
4. Enter the 4-character hex string into the **Gazco - Unit ID** text input

The default is `0FD0`. The value persists across reboots.

## Setup

1. Flash `minibuddy-gazco.yaml` to TTGO LoRa32 v2.1
2. Set desired fire settings using the select/switch controls
3. Press "Gazco - Send Command" to transmit

## Safety

- This system controls heating equipment — test thoroughly before unsupervised use
- Keep original remote as backup
- Comply with local RF transmission regulations (433 MHz ISM band)

## Licence

Released under the [MIT Licence](LICENSE).

## References

- [ESPHome SX127x Component](https://esphome.io/components/sx127x.html)
- A7302 RF Transceiver (AMICCOM) — original remote RF chip
- STM8L151 — original remote MCU
