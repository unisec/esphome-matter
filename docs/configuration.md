# Configuration reference

> 🇧🇷 Versão em português: [configuracao.md](configuracao.md)

## `matter:` component

```yaml
matter:
  vendor_id: 0xFFF1
  product_id: 0x8000
  setup_code: "20202021"
  discriminator: 3840
  # optional
  vendor_name: "Unicontrol"
  product_name: "Matter Relay"
  node_label: "Living Room Relay"
  hardware_version_string: "rev-a"
  software_version_string: "1.0.0"
  ble_commissioning: false
  topology: bridge
```

| Option | Type | Required | Default | Description |
|---|---|---|---|---|
| `id` | ID | no | auto | ID of the `MatterComponent` instance. |
| `vendor_id` | hex uint16 | **yes** | — | Matter Vendor ID. Use `0xFFF1`–`0xFFF4` (CSA test VIDs) during development. |
| `product_id` | hex uint16 | **yes** | — | Matter Product ID. Test range `0x8000`–`0x801F`. |
| `setup_code` | string (8 digits) | **yes** | — | Setup passcode (PIN). Must be `00000001`–`99999998` and **not** one of the reserved values (`00000000`, `11111111` … `99999999`, `12345678`, `87654321`). Quote it so leading zeros are preserved. |
| `discriminator` | int `0`–`4095` | **yes** | — | 12-bit discriminator used to find the device during commissioning. Use a different value per device on the same network. |
| `vendor_name` | string ≤ 32 | no | `""` | `BasicInformation.VendorName`. Empty keeps the SDK default. |
| `product_name` | string ≤ 32 | no | `""` | `BasicInformation.ProductName`. |
| `node_label` | string ≤ 32 | no | `""` | `BasicInformation.NodeLabel` (user-visible device name). |
| `hardware_version_string` | string ≤ 64 | no | `""` | `BasicInformation.HardwareVersionString`. |
| `software_version_string` | string ≤ 64 | no | `""` | `BasicInformation.SoftwareVersionString`. |
| `ble_commissioning` | bool | no | `false` | Enables Matter BLE commissioning (the controller sends Wi-Fi credentials over BLE). Requires `wifi:`. |
| `topology` | `bridge` \| `composed` | no | `bridge` | Endpoint layout — see [Topology](#topology). |

### Requirements checked at config time

The component refuses to build when:

- the framework is not `esp-idf`;
- the chip variant is not one of `esp32`, `esp32s3`, `esp32c3`, `esp32c6`, `esp32h2`;
- `network: enable_ipv6: true` is missing (Matter is IPv6-only);
- neither `wifi:` nor `ethernet:` is configured;
- `ble_commissioning: true` without `wifi:`;
- `ble_commissioning: true` together with `esp32_ble`, `esp32_ble_tracker`, `esp32_ble_beacon`, `bluetooth_proxy`, `improv_serial` or `esp32_improv`;
- the number of exposed entities needs more than 255 Matter endpoints.

### Topology

| Value | Layout | When to use |
|---|---|---|
| `bridge` (default) | `root (0)` → `Aggregator (1)` → one *Bridged Node* per entity, each with `BridgedDeviceBasicInformation.NodeLabel` = entity name. | Many independent entities. Each one shows as a separate accessory in Apple Home / Google Home / SmartThings / Home Assistant. |
| `composed` | `root (0)` → entity endpoints directly (Matter composed device). | A single product with several functions (e.g. 2-gang switch) or controllers that misbehave with bridges (eWeLink Cube / NSPanel Pro). Some controllers show generic names because only `FixedLabel` carries the entity name. |

Every endpoint also gets a `FixedLabel` (`name` = entity name, truncated to 16 bytes) and a writable `UserLabel` cluster persisted in NVS.

## Entity → Matter mapping

Every **non-internal** entity is exposed automatically. Use `internal: true` to hide an entity from Matter (it also hides it from Home Assistant's native API).

| ESPHome domain | Matter device type | Notes |
|---|---|---|
| `switch` | On/Off Plug-in Unit (0x010A) | Bidirectional on/off. |
| `light` (binary) | On/Off Light (0x0100) | |
| `light` (brightness) | Dimmable Light (0x0101) | OnOff + LevelControl. |
| `light` (color temperature) | Color Temperature Light (0x010C) | + ColorControl (mireds). |
| `light` (RGB) | Extended Color Light (0x010D) | |
| `binary_sensor` | Contact Sensor (0x0015) | Default for any `device_class`. |
| `binary_sensor` with `device_class: motion` / `occupancy` / `presence` | Occupancy Sensor (0x0107) | |
| `sensor` `°C` / `device_class: temperature` | Temperature Sensor (0x0302) | |
| `sensor` `%` + `device_class: humidity` | Humidity Sensor (0x0307) | `%` alone is ambiguous (battery) and is skipped. |
| `sensor` `hPa` / `kPa` / `device_class: pressure`/`atmospheric_pressure` | Pressure Sensor (0x0305) | |
| `sensor` `lx` / `device_class: illuminance` | Light Sensor (0x0106) | |
| `sensor` `m³/h` | Flow Sensor (0x0306) | |
| other `sensor`s | — | Skipped with a warning in the log. |
| `cover` | Window Covering (0x0202) | Lift + position-aware lift (0–100 %). |
| `fan` | Fan (0x002B) | On/off, percent; MultiSpeed when `speed_count` is set. |
| `lock` | Door Lock (0x000A) | Lock / Unlock commands. |
| `valve` | Water Valve (0x0042) | Open / close. |
| `select` | Mode Select (0x0027) | Each option becomes a mode. |
| `climate` | Thermostat (0x0301) | Needs at least HEAT or COOL. AUTO requires two-point target temperature. |
| `button` | Generic Switch (0x000F) | Emits `InitialPress` + `ShortRelease` events — use as automation trigger. |
| `button` `platform: matter` | — | **Not** exposed; local management action (see below). |

## `button: platform: matter`

Buttons that run Matter management actions **on the device itself**. They are visible in ESPHome / Home Assistant / web_server but never exposed to the Matter fabric.

```yaml
button:
  - platform: matter
    name: "Open Matter Pairing Window"
    action: open_commissioning_window
    timeout: 5min
  - platform: matter
    name: "Matter Factory Reset"
    action: factory_reset
```

| Option | Type | Required | Description |
|---|---|---|---|
| `action` | `open_commissioning_window` \| `factory_reset` | **yes** | Action executed on press. |
| `timeout` | time `180s`–`900s` | no (default `300s`) | Only for `open_commissioning_window`. |
| all [button](https://esphome.io/components/button/) options | | | `name`, `icon`, `entity_category`, … |

- **`open_commissioning_window`** — opens an *Enhanced Commissioning Window* with a **new random passcode and discriminator** so a second controller can be added (multi-admin). The new manual code and QR payload are printed in the log.
- **`factory_reset`** — removes all Matter fabrics and reboots. The device returns to the initial pairing state.

### Calling from a lambda

```yaml
on_press:
  - lambda: |-
      esphome::matter::MatterComponent::instance()->open_commissioning_window(300);
```

```yaml
  - lambda: |-
      esphome::matter::MatterComponent::instance()->factory_reset();
```

## Automatic SDK configuration

The component sets the required ESP-IDF / CHIP `sdkconfig` options for you, including:

- `CONFIG_ESP_MATTER_MAX_DYNAMIC_ENDPOINT_COUNT` — computed from the number of entities (+2 root/aggregator, +4 headroom, min 16). Override it in `esp32.framework.sdkconfig_options` and the component will respect your value.
- Wi-Fi / Ethernet commissioning drivers according to the configured transport.
- NimBLE + CHIPoBLE only when `ble_commissioning: true`.
- `CONFIG_MAX_FABRICS` = 10 with PSRAM, 6 without.
- When `psram:` is present: `SPIRAM_USE_MALLOC` (CHIP allocations spill to PSRAM) and `CHIP_SYSTEM_CONFIG_POOL_USE_HEAP`.
- TCP transport, larger MRP retry intervals, larger event queue — tuned for bridges with many endpoints and several controllers.
- IPv4 kept enabled inside CHIP (some hubs only reach the device over IPv4).
