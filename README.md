# esphome-yambms-addons

Non-mainline addon packages for [YamBMS](https://github.com/Sleeper85/esphome-yambms).

These packages extend YamBMS with experimental BLE-based transport features that are not part of the upstream project.

## Contents

### BLE Client/Server Transport

BLE-based multi-node communication as an alternative to modbus RS485. A BLE server node reads BMS data locally and advertises it over BLE GATT. A BLE client node discovers and combines data from multiple servers.

| Package | Description |
|---------|-------------|
| `packages/bms/bms_ble_server.yaml` | Core BLE GATT server for BMS data |
| `packages/bms/bms_combine_ble_client.yaml` | BLE client that receives BMS data |
| `packages/bms/bms_BLE_*` | BMS-specific BLE server wrappers (EG4, Ecoworthy, JBD, JK) |
| `packages/board/board_options_itf_bluetooth.yaml` | Bluetooth interface options |
| `packages/board/board_options_display_128x128_server.yaml` | 128x128 display layout for server nodes |

### BLE Inverter Proxy

Proxies inverter CAN bus or RS485 communication over BLE, allowing the inverter-connected node to be physically separate from the BMS-reading nodes.

| Package | Description |
|---------|-------------|
| `packages/inverter/inverter_proxy_ble_client.yaml` | BLE client for inverter proxy |
| `packages/inverter/inverter_proxy_canbus.yaml` | CAN bus inverter proxy |
| `packages/inverter/inverter_proxy_rs485.yaml` | RS485 inverter proxy |
| `packages/yambms/yambms_ble_server.yaml` | YamBMS BLE server integration |
| `packages/shunt/shunt_combine_Victron_SmartShunt_BLE_connect.yaml` | Victron SmartShunt BLE direct connect |

### Charger Proxy

Controls chargers that speak the Pylontech RS485 battery protocol by spoofing battery responses. The proxy receives real battery data from YamBMS over BLE and responds to charger RS485 queries (0x60–0x63) as if it were a Pylontech battery. Supports multiple chargers on separate UARTs with independent charge control.

See [CHARGER_PROXY_README.md](CHARGER_PROXY_README.md) for full documentation, hardware details, and protocol notes.

| Package | Description |
|---------|-------------|
| `packages/charger/charger_proxy_rs485.yaml` | RS485 Pylontech protocol responder (per-charger, used with `vars:`) |

### Examples

| File | Description |
|------|-------------|
| `YamBMS_BLE_Server_Example.yaml` | BLE server node example |
| `YamBMS_BLE_Proxy_CAN_Example.yaml` | BLE inverter proxy with CAN bus |
| `YamBMS_BLE_Proxy_RS485_Example.yaml` | BLE inverter proxy with RS485 |
| `YamBMS_Charger_Proxy_Example.yaml` | BLE charger proxy with RS485 (Ecoworthy EC100) |

## Usage

These packages can be used as remote packages from this repo, or copied locally.

```yaml
packages:
  ble:
    url: https://github.com/RAR/esphome-yambms-addons
    ref: main
    files:
      - path: 'packages/bms/bms_ble_server.yaml'
```

## License

GPLv3 — see [LICENSE](LICENSE).
