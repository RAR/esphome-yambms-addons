# Charger Proxy RS485: Solar Assistant Support — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `charger_baud_rate` and `charger_charge_control` variables to `charger_proxy_rs485.yaml` so it can serve Solar Assistant (115200 baud, no charge spoofing) while preserving current charger behavior by default.

**Architecture:** Two new optional vars with backward-compatible defaults. Jinja2 `<% if %>` blocks conditionally include full charge-control entities or internal-only stubs. Lambda logic uses compile-time boolean to skip spoofing when charge control is disabled.

**Tech Stack:** ESPHome YAML, Jinja2 (ESPHome custom delimiters: `<% %>`), C++ lambdas

---

### Task 1: Add default substitutions and update baud rate

**Files:**
- Modify: `packages/charger/charger_proxy_rs485.yaml:1-31` (header comments + substitutions)
- Modify: `packages/charger/charger_proxy_rs485.yaml:149` (baud_rate)

- [ ] **Step 1: Update header comments to document new vars**

Replace lines 1-31 with:

```yaml
# Charger RS485 Proxy - Pylontech protocol responder
# GitHub: https://github.com/RAR/esphome-yambms-addons
#
# Used as a remote package with vars:
#   packages:
#     charger_a:
#       url: https://github.com/RAR/esphome-yambms-addons
#       ref: main
#       files:
#         - path: 'packages/charger/charger_proxy_rs485.yaml'
#           vars:
#             charger_id: 'charger_a'
#             charger_name: 'Charger A'
#             charger_uart_id: 'uart_charger_a'
#             charger_tx_pin: '17'
#             charger_rx_pin: '18'
#             charger_flow_control_pin: '21'
#             charger_baud_rate: '9600'          # optional, default 9600
#             charger_charge_control: 'true'     # optional, default true
#
# Required vars:
#   charger_id:               Unique ID (e.g., 'charger_a')
#   charger_name:             Display name (e.g., 'Charger A')
#   charger_uart_id:          UART ID (e.g., 'uart_charger_a')
#   charger_tx_pin:           UART TX GPIO pin
#   charger_rx_pin:           UART RX GPIO pin
#   charger_flow_control_pin: DE/RE direction control GPIO pin
#
# Optional vars:
#   charger_baud_rate:        UART baud rate (default: '9600')
#   charger_charge_control:   Enable charge control entities and spoofing (default: 'true')
#                             Set to 'false' for monitoring-only (e.g., Solar Assistant)
#
# Also requires ${proxy_id} from top-level substitutions
# (references shared BLE client sensors).
#
# Full Pylontech RS485 protocol emulator (0x60-0x63).
# Responds to charger queries as if it were a Pylontech battery.
```

- [ ] **Step 2: Add default substitutions block**

Add after the header comments, before the `globals:` section:

```yaml
substitutions:
  charger_baud_rate: '9600'
  charger_charge_control: 'true'
```

- [ ] **Step 3: Replace hardcoded baud rate**

Change line 149 from:

```yaml
    baud_rate: 9600
```

to:

```yaml
    baud_rate: ${charger_baud_rate}
```

- [ ] **Step 4: Commit**

```bash
git add packages/charger/charger_proxy_rs485.yaml
git commit -m "feat: add charger_baud_rate and charger_charge_control substitutions"
```

---

### Task 2: Wrap charge-control entities in Jinja2 conditionals

**Files:**
- Modify: `packages/charger/charger_proxy_rs485.yaml:39-119` (switch, number, sensor, text_sensor entities)

The charge-control entities need two versions: full (when `charger_charge_control == 'true'`) and internal stubs (when `'false'`). Stubs must exist so lambda ID references compile.

- [ ] **Step 1: Wrap the Charge Enable switch (lines 39-46)**

Replace:

```yaml
switch:
  - platform: template
    name: "${charger_name} Charge Enable"
    id: ${charger_id}_charge_enable
    icon: "mdi:battery-charging"
    optimistic: true
    restore_mode: RESTORE_DEFAULT_ON
    entity_category: config
```

with:

```yaml
<% if charger_charge_control == 'true' %>
switch:
  - platform: template
    name: "${charger_name} Charge Enable"
    id: ${charger_id}_charge_enable
    icon: "mdi:battery-charging"
    optimistic: true
    restore_mode: RESTORE_DEFAULT_ON
    entity_category: config
<% else %>
switch:
  - platform: template
    id: ${charger_id}_charge_enable
    internal: true
    optimistic: true
    restore_mode: ALWAYS_ON
<% endif %>
```

- [ ] **Step 2: Wrap Sent CVL and Sent CCL sensors (lines 73-84)**

Replace:

```yaml
  - platform: template
    id: ${charger_id}_sent_cvl
    name: "${charger_name} Sent CVL"
    unit_of_measurement: "V"
    accuracy_decimals: 1
    icon: "mdi:flash"
  - platform: template
    id: ${charger_id}_sent_ccl
    name: "${charger_name} Sent CCL"
    unit_of_measurement: "A"
    accuracy_decimals: 1
    icon: "mdi:current-dc"
```

with:

```yaml
<% if charger_charge_control == 'true' %>
  - platform: template
    id: ${charger_id}_sent_cvl
    name: "${charger_name} Sent CVL"
    unit_of_measurement: "V"
    accuracy_decimals: 1
    icon: "mdi:flash"
  - platform: template
    id: ${charger_id}_sent_ccl
    name: "${charger_name} Sent CCL"
    unit_of_measurement: "A"
    accuracy_decimals: 1
    icon: "mdi:current-dc"
<% else %>
  - platform: template
    id: ${charger_id}_sent_cvl
    internal: true
  - platform: template
    id: ${charger_id}_sent_ccl
    internal: true
<% endif %>
```

- [ ] **Step 3: Wrap Charge Mode text sensor (lines 86-91)**

Replace:

```yaml
text_sensor:
  - platform: template
    id: ${charger_id}_charge_mode
    name: "${charger_name} Charge Mode"
    icon: "mdi:information-outline"
    entity_category: diagnostic
```

with:

```yaml
<% if charger_charge_control == 'true' %>
text_sensor:
  - platform: template
    id: ${charger_id}_charge_mode
    name: "${charger_name} Charge Mode"
    icon: "mdi:information-outline"
    entity_category: diagnostic
<% else %>
text_sensor:
  - platform: template
    id: ${charger_id}_charge_mode
    internal: true
<% endif %>
```

- [ ] **Step 4: Wrap CVL Override and CCL Override number entities (lines 93-119)**

Replace:

```yaml
number:
  - platform: template
    name: "${charger_name} CVL Override"
    id: ${charger_id}_cvl_override
    icon: "mdi:flash"
    unit_of_measurement: "V"
    min_value: 0
    max_value: 58
    step: 0.1
    initial_value: 0
    restore_value: true
    optimistic: true
    entity_category: config
    mode: box
  - platform: template
    name: "${charger_name} CCL Override"
    id: ${charger_id}_ccl_override
    icon: "mdi:current-dc"
    unit_of_measurement: "A"
    min_value: 0
    max_value: 300
    step: 1
    initial_value: 0
    restore_value: true
    optimistic: true
    entity_category: config
    mode: box
```

with:

```yaml
<% if charger_charge_control == 'true' %>
number:
  - platform: template
    name: "${charger_name} CVL Override"
    id: ${charger_id}_cvl_override
    icon: "mdi:flash"
    unit_of_measurement: "V"
    min_value: 0
    max_value: 58
    step: 0.1
    initial_value: 0
    restore_value: true
    optimistic: true
    entity_category: config
    mode: box
  - platform: template
    name: "${charger_name} CCL Override"
    id: ${charger_id}_ccl_override
    icon: "mdi:current-dc"
    unit_of_measurement: "A"
    min_value: 0
    max_value: 300
    step: 1
    initial_value: 0
    restore_value: true
    optimistic: true
    entity_category: config
    mode: box
<% else %>
number:
  - platform: template
    id: ${charger_id}_cvl_override
    internal: true
    min_value: 0
    max_value: 58
    step: 0.1
    initial_value: 0
    optimistic: true
  - platform: template
    id: ${charger_id}_ccl_override
    internal: true
    min_value: 0
    max_value: 300
    step: 1
    initial_value: 0
    optimistic: true
<% endif %>
```

- [ ] **Step 5: Commit**

```bash
git add packages/charger/charger_proxy_rs485.yaml
git commit -m "feat: conditionally hide charge control entities via Jinja2"
```

---

### Task 3: Update lambda logic to skip spoofing when charge control is disabled

**Files:**
- Modify: `packages/charger/charger_proxy_rs485.yaml` — interval lambda (~line 124-143) and UART debug lambda (~line 207-350)

- [ ] **Step 1: Update interval charge mode logic (lines 124-143)**

Replace the interval lambda:

```yaml
  - interval: 1s
    then:
      - lambda: |-
          uint32_t now = millis();
          if (id(${charger_id}_rs485_last_request) > 0) {
            uint32_t elapsed = (now - id(${charger_id}_rs485_last_request)) / 1000;
            id(${charger_id}_rs485_heartbeat).publish_state(elapsed);
            id(${charger_id}_rs485_status).publish_state(elapsed <= 30);
          }
          std::string mode;
          if (id(${charger_id}_rs485_protocol).active_index() == 0) {
            mode = "Disabled";
          } else if (!id(${proxy_id}_ble_connected).state) {
            mode = "BLE Disconnected";
          } else if (id(${charger_id}_charge_enable).state) {
            mode = "Charging Allowed";
          } else {
            mode = "Charging Blocked";
          }
          if (!id(${charger_id}_charge_mode).has_state() || id(${charger_id}_charge_mode).state != mode) {
            id(${charger_id}_charge_mode).publish_state(mode);
          }
```

with:

```yaml
  - interval: 1s
    then:
      - lambda: |-
          uint32_t now = millis();
          if (id(${charger_id}_rs485_last_request) > 0) {
            uint32_t elapsed = (now - id(${charger_id}_rs485_last_request)) / 1000;
            id(${charger_id}_rs485_heartbeat).publish_state(elapsed);
            id(${charger_id}_rs485_status).publish_state(elapsed <= 30);
          }
          std::string mode;
          if (id(${charger_id}_rs485_protocol).active_index() == 0) {
            mode = "Disabled";
          } else if (!id(${proxy_id}_ble_connected).state) {
            mode = "BLE Disconnected";
          } else if (!${charger_charge_control}) {
            mode = "Monitoring";
          } else if (id(${charger_id}_charge_enable).state) {
            mode = "Charging Allowed";
          } else {
            mode = "Charging Blocked";
          }
          if (!id(${charger_id}_charge_mode).has_state() || id(${charger_id}_charge_mode).state != mode) {
            id(${charger_id}_charge_mode).publish_state(mode);
          }
```

- [ ] **Step 2: Update charge_ok derivation in UART lambda (line 207)**

Replace:

```cpp
            bool charge_ok = id(${charger_id}_charge_enable).state;
```

with:

```cpp
            bool charge_ok = ${charger_charge_control} ? id(${charger_id}_charge_enable).state : true;
```

- [ ] **Step 3: Update 0x61 SOC spoofing (line 245)**

Replace:

```cpp
              float reported_soc = charge_ok ? real_soc : 100.0f;
```

No change needed — this already uses `charge_ok`, which is now always `true` when charge control is disabled. The line stays as-is.

- [ ] **Step 4: Wrap CVL/CCL override logic in 0x63 handler (lines 320-323)**

Replace:

```cpp
              // Apply CVL/CCL overrides (0 = use BMS value)
              float cvl_ovr = id(${charger_id}_cvl_override).state;
              float ccl_ovr = id(${charger_id}_ccl_override).state;
              if (!std::isnan(cvl_ovr) && cvl_ovr > 0) reported_cvl = cvl_ovr;
              if (!std::isnan(ccl_ovr) && ccl_ovr > 0) reported_ccl = ccl_ovr;
```

with:

```cpp
              // Apply CVL/CCL overrides (0 = use BMS value)
              float cvl_ovr = 0;
              float ccl_ovr = 0;
              if (${charger_charge_control}) {
                cvl_ovr = id(${charger_id}_cvl_override).state;
                ccl_ovr = id(${charger_id}_ccl_override).state;
                if (!std::isnan(cvl_ovr) && cvl_ovr > 0) reported_cvl = cvl_ovr;
                if (!std::isnan(ccl_ovr) && ccl_ovr > 0) reported_ccl = ccl_ovr;
              }
```

- [ ] **Step 5: Update 0x63 sent CVL/CCL publish (lines 341-342)**

Replace:

```cpp
              id(${charger_id}_sent_cvl).publish_state(reported_cvl);
              id(${charger_id}_sent_ccl).publish_state(reported_ccl);
```

with:

```cpp
              if (${charger_charge_control}) {
                id(${charger_id}_sent_cvl).publish_state(reported_cvl);
                id(${charger_id}_sent_ccl).publish_state(reported_ccl);
              }
```

- [ ] **Step 6: Commit**

```bash
git add packages/charger/charger_proxy_rs485.yaml
git commit -m "feat: skip charge spoofing when charger_charge_control is false"
```

---

### Task 4: Update documentation

**Files:**
- Modify: `README.md`
- Modify: `CHARGER_PROXY_README.md`

- [ ] **Step 1: Add Solar Assistant section to CHARGER_PROXY_README.md**

Add a new section after the existing charger configuration documentation:

```markdown
## Solar Assistant / Monitoring-Only Mode

To use this package for monitoring (e.g., with Solar Assistant), set the optional
variables to disable charge control and use the correct baud rate:

```yaml
packages:
  solar_assistant:
    url: https://github.com/RAR/esphome-yambms-addons
    ref: main
    refresh: 0s
    files:
      - path: 'packages/charger/charger_proxy_rs485.yaml'
        vars:
          charger_id: 'solar_assistant'
          charger_name: 'Solar Assistant'
          charger_uart_id: 'uart_solar_assistant'
          charger_tx_pin: '7'
          charger_rx_pin: '8'
          charger_flow_control_pin: '21'
          charger_baud_rate: '115200'
          charger_charge_control: 'false'
```

When `charger_charge_control` is `'false'`:
- Charge Enable switch, CVL/CCL overrides, Charge Mode, and Sent CVL/CCL entities are hidden
- Battery data is always reported accurately (no SOC/CVL/CCL spoofing)
- RS485 Protocol selector, RS485 Status, and RS485 Heartbeat remain visible for diagnostics
```

- [ ] **Step 2: Update README.md to mention Solar Assistant support**

Add a brief note in the charger proxy section mentioning Solar Assistant compatibility with a link to `CHARGER_PROXY_README.md`.

- [ ] **Step 3: Commit**

```bash
git add README.md CHARGER_PROXY_README.md
git commit -m "docs: add Solar Assistant usage to charger proxy documentation"
```

---

### Task 5: Verify with ESPHome compile

- [ ] **Step 1: Create a minimal test config**

Create a temporary test file `test_solar_assistant.yaml` in the repo root (do not commit):

```yaml
substitutions:
  proxy_id: 'test_proxy'
  yambms_server_mac: '00:00:00:00:00:00'

packages:
  ble_client:
    files:
      - path: 'packages/inverter/inverter_proxy_ble_client.yaml'

  solar_assistant:
    files:
      - path: 'packages/charger/charger_proxy_rs485.yaml'
        vars:
          charger_id: 'solar_assistant'
          charger_name: 'Solar Assistant'
          charger_uart_id: 'uart_solar_assistant'
          charger_tx_pin: '7'
          charger_rx_pin: '8'
          charger_flow_control_pin: '21'
          charger_baud_rate: '115200'
          charger_charge_control: 'false'

esphome:
  name: test-solar-assistant

esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: esp-idf
```

- [ ] **Step 2: Run ESPHome compile check**

```bash
esphome compile test_solar_assistant.yaml
```

Expected: Compiles without errors. Jinja2 conditionals resolve correctly, internal stubs provide valid IDs, lambda compile-time booleans eliminate dead branches.

- [ ] **Step 3: Also test with default vars (backward compatibility)**

Create `test_charger_default.yaml`:

```yaml
substitutions:
  proxy_id: 'test_proxy'
  yambms_server_mac: '00:00:00:00:00:00'

packages:
  ble_client:
    files:
      - path: 'packages/inverter/inverter_proxy_ble_client.yaml'

  charger_a:
    files:
      - path: 'packages/charger/charger_proxy_rs485.yaml'
        vars:
          charger_id: 'charger_a'
          charger_name: 'Charger A'
          charger_uart_id: 'uart_charger_a'
          charger_tx_pin: '17'
          charger_rx_pin: '18'
          charger_flow_control_pin: '21'

esphome:
  name: test-charger-default

esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: esp-idf
```

```bash
esphome compile test_charger_default.yaml
```

Expected: Compiles without errors. Default behavior unchanged (9600 baud, all charge control entities visible).

- [ ] **Step 4: Clean up test files**

```bash
rm test_solar_assistant.yaml test_charger_default.yaml
```
