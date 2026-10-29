# Benfu BLE Charger Protocol

This document describes observed Bluetooth Low Energy interoperability behavior for Benfu chargers exposing the GATT profile below. It is an independent protocol description based on compatible implementations, test vectors, vendor-application data handling, and selected device readbacks. It is not an official specification and does not imply vendor endorsement.

## Scope

This reference covers realtime telemetry (ff11), basic settings (ff12 read, ff22 write), advanced settings (ff14 read/write), and third-stage configuration import (ff40). Other observed characteristics are listed for discovery only. Hardware and firmware revisions may vary. At least two advanced-settings layouts have been observed.

## GATT

The observed service UUIDs are 0000ff01-0000-1000-8000-00805f9b34fb and 0000ff02-0000-1000-8000-00805f9b34fb. Characteristic UUIDs use the Bluetooth base UUID format.

| Short UUID | Expanded characteristic UUID | Role | Observed access |
|---|---|---|---|
| ff11 | 0000ff11-0000-1000-8000-00805f9b34fb | Realtime telemetry | Notify |
| ff12 | 0000ff12-0000-1000-8000-00805f9b34fb | Basic settings readback | Read |
| ff13 | 0000ff13-0000-1000-8000-00805f9b34fb | Calibration | Incompletely documented |
| ff14 | 0000ff14-0000-1000-8000-00805f9b34fb | Advanced settings | Read; write with response |
| ff18 | 0000ff18-0000-1000-8000-00805f9b34fb | Lock control | Incompletely documented |
| ff22 | 0000ff22-0000-1000-8000-00805f9b34fb | Basic settings write | Write with response |
| ff23 | 0000ff23-0000-1000-8000-00805f9b34fb | Mainboard data/ID path | Incompletely documented |
| ff25 | 0000ff25-0000-1000-8000-00805f9b34fb | Mainboard ID read path | Incompletely documented |
| ff40 | 0000ff40-0000-1000-8000-00805f9b34fb | Configuration import | Write with response; no application acknowledgement observed |

Discover actual services and characteristic properties. Subscribe specifically to ff11 for telemetry; the charger pushes updates and does not require polling.

## Encoding and CRC

Unless noted otherwise, multi-byte fields are big-endian. Values in hundredths use integer value equal to the physical value multiplied by 100. CRC-bearing frames use CRC-16/CMS as observed in compatible implementations: polynomial 0x8005, initial value 0xFFFF, MSB first, no reflection, no final XOR. CRC covers the body only and is appended as two big-endian bytes. In the import body, selected fields are little-endian even though the CRC remains big-endian.

Reject frames with an unexpected length or invalid CRC. Do not treat missing data as zero.

## Realtime notification: ff11

Observed notifications are at least 20 bytes. Offsets are from the first notification byte. No CRC is observed in this frame.

| Offset | Width | Field | Decoding |
|---:|---:|---|---|
| 0 | 1 | Battery mode | 0 lead-acid; 1 lithium; 17 always-on; 240 open-loop; 241 aging; otherwise unknown |
| 1 | 2 | Battery voltage | u16 BE / 100 V |
| 3 | 2 | Maximum current setting | u16 BE / 100 A |
| 5 | 1 | Work status | 0 no battery; 1 charging; 2 complete |
| 6 | 1 | Charge segment count | Unsigned raw value |
| 7 | 1 | Charge stage | Unsigned raw value; enumeration not established |
| 8 | 2 | Fault status | Bit field; byte order selected by compatibility heuristic |
| 10 | 2 | Output current | u16 BE / 100 A |
| 12 | 2 | Output voltage | u16 BE / 100 V |
| 14 | 2 | Charger temperature | signed i16 degrees C |
| 16 | 2 | Heatsink/radiator temperature | signed i16 degrees C |
| 18 | 2 | Charged capacity | u16 BE / 100 Ah |
| 20 | 1 | Battery level | Optional percent byte |
| 21 | 2 | Bus voltage | Optional u16 BE / 100 V |
| 23 | 1 | Lock code | Optional; A1 observed locked, A8 observed unlocked |

Observed fault bits: 4000 hardware overcurrent, 2000 software overcurrent, 1000 output undervoltage, 0800 timeout protection, 0400 overtemperature, 0200 input overvoltage, 0100 input undervoltage, and 0001 user shutdown (hexadecimal). The compatibility decoder checks both byte orders against reserved mask 80FE. If exactly one interpretation avoids the reserved bits, that interpretation is selected; otherwise it defaults to big-endian. This is a heuristic, not a definitive wire-format guarantee.

## Basic settings: ff12 readback, ff22 write

### ff12 readback

Two observed frames exist: 19 bytes (17-byte body plus CRC) and 21 bytes (19-byte body plus CRC).

| Offset | Width | Field | Decoding |
|---:|---:|---|---|
| 0 | 1 | Auxiliary flags | bit 0 force activation; bit 1 balance mode |
| 1 | 2 | Target voltage | u16 BE / 100 V |
| 3 | 2 | Target current | u16 BE / 100 A |
| 5 | 2 | Voltage calibration value | u16; units/semantics unknown |
| 7 | 2 | Equalization voltage | u16 BE / 100 V |
| 9 | 2 | Equalization current | u16 BE / 100 A |
| 11 | 2 | Maximum voltage | u16 BE / 100 V |
| 13 | 2 | Maximum current | u16 BE / 100 A |
| 15 | 2 or 4 | Maximum power | u16 in 19-byte form; u32 in 21-byte form, watts |
| 17 or 19 | 2 | CRC-16 | BE; covers prior 17 or 19 bytes |

### ff22 write

The observed write frame is 13 bytes: an 11-byte body plus CRC.

| Offset | Width | Field | Encoding |
|---:|---:|---|---|
| 0 | 2 | Target voltage | u16 BE, 0.01 V |
| 2 | 2 | Target current | u16 BE, 0.01 A |
| 4 | 1 | Auxiliary flags | bit 0 force activation; bit 1 balance mode |
| 5 | 2 | Equalization voltage | u16 BE, 0.01 V |
| 7 | 2 | Equalization current | u16 BE, 0.01 A |
| 9 | 1 | Reserved/observed zero | Send zero unless device-specific specification says otherwise |
| 10 | 1 | Reserved/observed zero | Send zero unless device-specific specification says otherwise |
| 11 | 2 | CRC-16 | BE; covers bytes 0–10 |

The frame replaces the fields it represents. Read first and explicitly preserve values that should remain. The frame does not contain calibration or maximum limits. Use write with response; an unacknowledged ATT write has been observed to disconnect a charger.

## Advanced settings: ff14

Read and write use the same characteristic. Two observed formats are 21 bytes total (19-byte body, older layout) and 23 bytes total (21-byte body, including maximum-voltage field). Readback CRC has been accepted in either byte order; new frames append the CRC big-endian.

| Offset | Width | Field | Encoding |
|---:|---:|---|---|
| 0 | 1 | Fan setting | Encoded value is 100 minus percent; percent 0–100 |
| 1 | 1 | Timed charging | 0 off, 1 on |
| 2 | 4 | Timestamp | u32 BE seconds; preserve the readback value |
| 6 | 4 | Start timestamp | u32 BE Unix seconds |
| 10 | 4 | Stop timestamp | u32 BE Unix seconds |
| 14 | 1 | Constant-power enabled | 0 off, 1 on |
| 15 | 2 | Constant-power value | u16 BE watts |
| 17 | 2 | Constant-power current limit | u16 BE, 0.01 A |
| 19 | 2 | Maximum/full-charge voltage | u16 BE, 0.01 V; only in 23-byte format |
| 19 or 21 | 2 | CRC-16 | Covers prior 19 or 21 bytes |

This is a whole-frame write. Read current settings first, preserve fields not intentionally changed, then recalculate CRC. Defaulting unread fields may clear settings. Timezone conventions for scheduled timestamps are not established.

## Third-stage import: ff40

The observed configuration body is 232 bytes followed by a two-byte big-endian CRC (234 bytes total). Only the offsets below are documented; do not guess the meaning of other bytes.

| Offset | Width | Field | Encoding |
|---:|---:|---|---|
| 0 | 1 | Battery mode | 0 lead-acid; 1 lithium |
| 28 | 2 | Third-stage voltage | u16 LE, 0.01 V |
| 44 | 2 | Third-stage current | u16 LE, 0.01 A |
| 224 | 1 | Auxiliary flags | bit 0 force activation; bit 1 balance mode |
| 226 | 2 | Equalization voltage | u16 LE, 0.01 V |
| 228 | 2 | Equalization current | u16 LE, 0.01 A |
| 232 | 2 | CRC-16 | BE; covers body bytes 0–231 |

The observed transport sends twelve write-with-response chunks no larger than 20 bytes, with a 14-byte final chunk. This is specific to the observed device/stack path. Do not apply this chunking to the shorter ff22 or ff14 frames.

No application-level acknowledgement has been observed for ff40, and third-stage fields cannot be read back through ff12 or ff14. Successful ATT writes prove transport delivery only, not that the charger accepted or applied the configuration.

## Safety and limitations

These characteristics can change charging behavior. Incorrect battery chemistry, voltage, current, balancing, or timing values can damage equipment or create a fire hazard.

- Prefer read-only discovery and telemetry. Before any write, identify the charger and battery chemistry and obtain a fresh settings readback.
- Stay within charger and battery manufacturer limits. Do not bypass charger or BMS protections.
- Stop on ATT errors, disconnects, invalid CRC, unexpected lengths, or a change in target identity.
- Preserve every field not intentionally changed in full-frame updates.
- Verify by readback where supported. Never report ff40 import success based solely on ATT writes.
- Keep unknown enum values and reserved bytes unchanged.

The layouts are supported by independent encoder/decoder implementations, offline vectors, compatible-application data handling, and selected device readbacks. Third-stage import is checked for structure, bounds, length, and CRC, but has no observed application-level acknowledgement. This is not a complete vendor specification and does not establish all firmware variants, security requirements, fault codes, calibration semantics, import-body schema, or timezone behavior.

No vendor source code or application package is reproduced. This document does not assign a license to source code or vendor material.
