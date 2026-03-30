# JK BMS Protocol Addition to Charger Proxy RS485

## Problem

Solar Assistant configured for JK BMS sends JK's proprietary protocol over RS485, not standard Modbus RTU. The proxy only speaks Pylontech V2 (CID2 0x60-0x63). SA sends two requests back-to-back:

1. **NW protocol** (21 bytes): `4E 57` header, command 0x06 (read all)
2. **Old RS485 trigger** (11 bytes): Modbus write `XX 10 16 20` triggering a Type 02 cell data response

The expected response is a 308-byte frame: `55 AA EB 90` header + 304 bytes of Type 02 payload containing all BMS data in JK's proprietary little-endian packed format.

## Solution

Add "JK" as a third option in the RS485 Protocol selector alongside "Pylon" and "Disabled".

### Detection

Match either:
- NW request: first two bytes `0x4E 0x57` with command byte (offset 8) = `0x06`
- Modbus write trigger: bytes pattern `XX 10 16 20` (any address, FC 0x10, register 0x1620)

### Response Frame (308 bytes)

Header: `55 AA EB 90` (4 bytes)
Payload: 304 bytes, record Type 0x02, all remaining bytes zero-initialized then filled:

| Payload Offset | Size | Field | Type | Source |
|---|---|---|---|---|
| 0 | 1 | Record type | u8 | `0x02` constant |
| 1 | 1 | Frame counter | u8 | Incrementing counter |
| 2-33 | 32 | Cell voltages 1-16 | u16 LE × 16 | Fake: cells 2-15 at avg voltage, cell 1 at min, cell 16 at max |
| 146-149 | 4 | Total voltage | u32 LE | `voltage * 1000` (mV) |
| 154-157 | 4 | Current | i32 LE | `current * 1000` (mA) |
| 158-159 | 2 | Temp sensor 1 | i16 LE | `min_temperature * 10` (0.1°C) |
| 160-161 | 2 | Temp sensor 2 | i16 LE | `max_temperature * 10` (0.1°C) |
| 162-164 | 3 | Alarms bitmask | 24-bit LE | Mapped from YamBMS error bitmasks |
| 168 | 1 | Balance status | u8 | `equalizing` ? 1 : 0 |
| 169 | 1 | SOC | u8 | `soc` (0-100%) |
| 170-173 | 4 | Capacity remaining | u32 LE | `capacity_remaining * 1000` (mAh) |
| 174-177 | 4 | Nominal capacity | u32 LE | `capacity * 1000` (mAh) |
| 178-181 | 4 | Cycle count | u32 LE | `charging_cycles` |
| 186-187 | 2 | SOH | u16 LE | `soh * 256` |
| 194 | 1 | Charge MOSFET | u8 | `charge_enabled` ? 1 : 0 |
| 195 | 1 | Discharge MOSFET | u8 | `discharge_enabled` ? 1 : 0 |

### Cell Voltage Faking Strategy

YamBMS provides: min_cell_voltage, max_cell_voltage, delta_cell_voltage, cell_count, total_voltage.

- Calculate average: `avg = total_voltage / cell_count` (in mV)
- Cell 1 = min_cell_voltage
- Cell 16 (or cell_count) = max_cell_voltage
- Cells 2 through 15 = avg voltage
- Report `cell_count` cells (default 16 if unavailable)

### Protocol Selector Change

Current options: `["Disabled", "Pylon"]`
New options: `["Disabled", "Pylon", "JK"]`

The UART debug lambda already parses incoming bytes. Add JK detection before the Pylon `0x7E` check:
- If `bytes[0] == 0x4E && bytes[1] == 0x57` → JK NW protocol
- If `bytes.size() >= 4 && bytes[1] == 0x10 && bytes[2] == 0x16` → JK Modbus trigger
- If `bytes[0] == 0x7E` → Pylon (existing)

### Charge Control Interaction

None. JK protocol is read-only monitoring. The `charger_charge_control` flag and spoofing logic only apply to Pylon responses. JK always reports real data.

## Scope

Single file modified: `packages/charger/charger_proxy_rs485.yaml`

Changes:
1. Add "JK" to RS485 Protocol selector options
2. Add JK frame counter global
3. Add JK detection and response builder in UART lambda
4. Update documentation
