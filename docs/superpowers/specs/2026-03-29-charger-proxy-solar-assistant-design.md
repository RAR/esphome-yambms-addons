# Charger Proxy RS485: Solar Assistant Support

## Problem

The existing `charger_proxy_rs485.yaml` package is hardcoded to 9600 baud and includes charge control spoofing logic (fake SOC, zeroed CVL/CCL) designed for controlling chargers like the Ecoworthy EC100. Solar Assistant expects Pylontech RS485 at 115200 baud and needs accurate, unspoofed battery data for monitoring.

## Solution

Add two optional variables to the existing `charger_proxy_rs485.yaml` package. No new files. Backward-compatible defaults preserve current behavior.

### New Variables

| Variable | Default | Purpose |
|---|---|---|
| `charger_baud_rate` | `'9600'` | UART baud rate. Set to `'115200'` for Solar Assistant. |
| `charger_charge_control` | `'true'` | When `'false'`: hides charge control entities, disables all spoofing logic. |

### Conditional Entity Inclusion

ESPHome's Jinja2 preprocessor (custom delimiters: `<% %>`) processes `vars` before YAML parsing. Block-level conditionals control which entities are included.

When `charger_charge_control == 'true'` (default, current behavior): all entities included as-is.

When `charger_charge_control == 'false'`: charge-control entities are replaced with internal-only stubs that still provide valid IDs for lambda references but are hidden from Home Assistant.

**Entities conditionally hidden:**

| Entity | Type | Stub Behavior |
|---|---|---|
| Charge Enable | switch | `internal: true`, `restore_mode: ALWAYS_ON` |
| CVL Override | number | `internal: true`, value ignored |
| CCL Override | number | `internal: true`, value ignored |
| Charge Mode | text_sensor | `internal: true` |
| Sent CVL | sensor | `internal: true` |
| Sent CCL | sensor | `internal: true` |

**Entities always visible (useful for diagnostics in all modes):**

| Entity | Type |
|---|---|
| RS485 Protocol | select |
| RS485 Status | binary_sensor |
| RS485 Heartbeat | sensor |

### Lambda Logic Changes

A compile-time boolean controls spoofing at zero runtime cost (dead code elimination):

```cpp
bool charge_ok = ${charger_charge_control} ? id(${charger_id}_charge_enable).state : true;
```

When `charger_charge_control` is `false`, `charge_ok` is always `true`, so:

- **0x61 (Analog Data):** `reported_soc = real_soc` (no 100% spoof)
- **0x63 (Charge Management):** `reported_cvl = real_cvl`, `reported_ccl = real_ccl` (no zeroing), CVL/CCL overrides skipped, status byte reflects real BMS flags

### UART Baud Rate

Simple substitution replacement:

```yaml
baud_rate: ${charger_baud_rate}
```

### Interval Logic

The charge mode text sensor update in the 1s interval is wrapped in a Jinja2 conditional or uses the compile-time boolean to publish "Monitoring" instead of "Charging Allowed/Blocked".

### Example Usage (Solar Assistant)

```yaml
packages:
  ble_client:
    url: https://github.com/RAR/esphome-yambms-addons
    ref: main
    refresh: 0s
    files:
      - path: 'packages/inverter/inverter_proxy_ble_client.yaml'

  solar_assistant:
    url: https://github.com/RAR/esphome-yambms-addons
    ref: main
    refresh: 0s
    files:
      - path: 'packages/charger/charger_proxy_rs485.yaml'
        vars:
          charger_id: 'solar_assistant'
          charger_name: 'Solar Assistant'
          charger_uart_id: 'uart_charger_a'
          charger_tx_pin: '7'
          charger_rx_pin: '8'
          charger_flow_control_pin: '21'
          charger_baud_rate: '115200'
          charger_charge_control: 'false'
```

### Example Usage (Existing Charger, Unchanged)

```yaml
packages:
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
```

No vars needed for `charger_baud_rate` or `charger_charge_control` -- defaults match current behavior exactly.

## Scope

Single file modified: `packages/charger/charger_proxy_rs485.yaml`

Changes:
1. Add default substitutions for `charger_baud_rate` and `charger_charge_control`
2. Replace hardcoded `baud_rate: 9600` with `baud_rate: ${charger_baud_rate}`
3. Wrap charge-control entities in `<% if charger_charge_control == 'true' %>` / `<% else %>` blocks with internal stubs
4. Update lambda `charge_ok` derivation to use compile-time conditional
5. Wrap CVL/CCL override logic in 0x63 handler with charge_control check
6. Update interval charge mode logic
7. Update header comments with new vars documentation

## Out of Scope

- New package files
- Changes to BLE client, inverter proxy, or CAN proxy packages
- Solar Assistant-specific protocol extensions (standard Pylon is sufficient)
