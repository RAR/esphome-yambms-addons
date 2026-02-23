# Ecoworthy EC100 Charger Proxy

Control Ecoworthy EC100 48V 100A charger(s) via Pylontech RS485 protocol spoofing,
using an ESP32 as a BLE-to-RS485 bridge connected to YamBMS.

## Architecture

```
YamBMS (Tab5)  ──BLE──►  ESP32 Proxy  ──RS485──►  Ecoworthy EC100 Charger
                          (Waveshare)
```

The proxy receives battery data from YamBMS over BLE (packed characteristics),
then responds to the charger's Pylontech RS485 queries as if it were a battery.

## Hardware

### ESP32 Board: Waveshare ESP32-S3-RS485-CAN

- [Product page](https://www.waveshare.com/esp32-s3-rs485-can.htm)
- ESP32-S3, 16MB Flash, 8MB PSRAM
- **Onboard RS485 transceiver** with explicit DE/RE direction control

| Function         | GPIO |
|------------------|------|
| RS485 TX         | 17   |
| RS485 RX         | 18   |
| RS485 DE/RE      | 21   |

### Charger: Ecoworthy EC100 (2026 model, 48V 100A)

- RS485 port via RJ45 connector (battery side)
- Acts as **RS485 master** — it sends Pylontech queries, expects battery responses
- Uses 9600 baud, ASCII Pylontech protocol

#### RJ45 Pinout (battery-side port)

| Pin  | Function     |
|------|-------------|
| 1, 8 | B- (Data-)  |
| 2, 7 | A+ (Data+)  |
| 3, 6 | GND         |
| 4, 5 | NC          |

**Note:** The supplied charger cable only connects pins 1,2,7,8 — no ground on 3,6.
A shared ground between the charger and ESP32 may be needed if communication is
unreliable.

### Wiring

Connect the charger's RS485 A+/B- to the Waveshare board's RS485 A/B screw terminals.
The charger can be powered on AC only, DC only, or both — it sends RS485 queries
as soon as AC power is connected.

## Software

### Files

| File | Purpose |
|------|---------|
| `packages/charger/charger_proxy_rs485.yaml` | RS485 Pylontech protocol responder (remote package, per-charger `vars:`) |
| `YamBMS_Charger_Proxy_Example.yaml` | Example device config — board, WiFi, BLE client, charger packages |

### Dependencies

- **esphome-yambms-addons** `packages/inverter/inverter_proxy_ble_client.yaml` — BLE client
  that connects to the YamBMS BLE server and unpacks battery data into sensors.

### Pylontech Protocol (0x60–0x63)

The charger cycles through four queries every ~2 seconds:

| CID2 | Name | Our Response |
|------|------|-------------|
| 0x60 | Battery Pack Info | Device name "US2000C", manufacturer "Pylontech" |
| 0x61 | Analog Data | Voltage, current, SOC, SOH, cell voltages, temperatures |
| 0x62 | Alarm Info | Alarm/warning bitmasks from YamBMS |
| 0x63 | Charge/Discharge Mgmt | CVL, CCL, DVL, DCL, status flags |

Frame format: `~{HEADER}{LENGTH}{INFO}{CHECKSUM}\r` (ASCII hex)

- Address: `0x02` (battery response)
- 9600 baud, 8N1

## Home Assistant Entities

### Per Charger (e.g., "Charger A")

| Entity | Type | Description |
|--------|------|-------------|
| Charge Enable | Switch | **Main control.** ON = normal operation, OFF = spoof SOC=100%, CCL=0A (charger stops) |
| RS485 Protocol | Select | "Pylon" or "Disabled" |
| Charge Mode | Text Sensor | Current state: "Charging Allowed", "Charging Blocked", "BLE Disconnected", "Disabled" |
| RS485 Status | Binary Sensor | ON if charger queries received within last 30s |
| RS485 Heartbeat | Sensor | Seconds since last charger query |
| Sent CVL | Sensor | Actual charge voltage limit being sent to charger (V) |
| Sent CCL | Sensor | Actual charge current limit being sent to charger (A) |
| CVL Override | Number | Override CVL sent to charger (0 = use BMS value) |
| CCL Override | Number | Override CCL sent to charger (0 = use BMS value) |

### Charge Blocking Logic

When **Charge Enable = OFF**:
1. SOC is reported as **100%** (0x61 response)
2. CCL is set to **0A** (0x63 response)
3. CVL is set to **current pack voltage** (prevents voltage jump)
4. Charge status bit cleared

This reliably stops the EC100 from charging.

### CVL/CCL Overrides

**Note:** Testing confirmed the EC100 **ignores** CVL and CCL values for rate limiting.
It uses its own internal charging profile and only reads SOC to decide start/stop.
The overrides are available but have no practical effect on this charger model.

## Multi-Charger Support

`charger_proxy_rs485.yaml` is designed for multiple chargers via remote package `vars:`:

```yaml
packages:
  # Charger A - onboard RS485
  charger_a:
    url: https://github.com/RAR/esphome-yambms-addons
    ref: main
    refresh: 0s
    files:
      - path: 'packages/charger/charger_proxy_rs485.yaml'
        vars:
          charger_id: 'charger_a'
          charger_name: 'Charger A'
          charger_uart_id: 'uart_charger_a'
          charger_tx_pin: '17'
          charger_rx_pin: '18'
          charger_flow_control_pin: '21'

  # Charger B - external RS485 adapter
  charger_b:
    url: https://github.com/RAR/esphome-yambms-addons
    ref: main
    refresh: 0s
    files:
      - path: 'packages/charger/charger_proxy_rs485.yaml'
        vars:
          charger_id: 'charger_b'
          charger_name: 'Charger B'
          charger_uart_id: 'uart_charger_b'
          charger_tx_pin: '5'
          charger_rx_pin: '6'
          charger_flow_control_pin: '7'
```

Each charger gets its own UART, own HA entities, and independent charge control.
The Waveshare board has additional UART pins on the SH1.0 connector (GPIO1/2)
and pin header (GPIO5/6) for a second RS485 adapter.

## Critical Lessons Learned

### 1. RS485 Transceivers Need Explicit DE/RE Control

The M5Stack RS485 Unit (U034) uses "auto direction" where DE/RE is derived from
the TX line. Since UART TX idles HIGH, the driver stays permanently enabled,
overpowering incoming signals. **You will never receive data with this unit.**

**Solution:** Use a board/adapter with an explicit DE/RE pin (like the Waveshare
ESP32-S3-RS485-CAN with GPIO21) and configure ESPHome's `flow_control_pin`.

### 2. UART Debug Needs `dummy_receiver: true`

ESPHome's UART debug component only monitors bytes consumed by other components.
If nothing else reads the UART (our case — the debug sequence IS the reader),
you must set `dummy_receiver: true` or the debug lambda never fires.

### 3. The EC100 is the RS485 Master

The charger sends queries, the battery responds. This is the opposite of most
RS485 setups where the controller is the master. The ESP32 must be purely
reactive — listen for queries and respond to each one.

### 4. Remote Package `vars:` Work Great for Per-Instance Config

ESPHome's remote package `vars:` feature allows each charger instance to have
its own IDs, names, and pin assignments without any local files. Each package
block can pass different `vars:` to the same remote YAML, creating fully
independent charger instances.

## Configuration

1. Set `yambms_server_mac` to your YamBMS BLE server's MAC address
2. Update `secrets.yaml` with WiFi credentials
3. Flash with ESPHome
4. Verify BLE connects (check for sensor values in logs)
5. Connect RS485 to charger — should see `Pylon request CID2=60/61/62/63` in logs
6. Charger display should show SOC
7. Toggle "Charge Enable" switch to verify start/stop control
