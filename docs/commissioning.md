# Commissioning & troubleshooting

> 🇧🇷 Versão em português: [comissionamento.md](comissionamento.md)

## 1. First pairing

After flashing, open the logs (`esphome logs device.yaml`). On boot the component prints the onboarding payload:

```
[I][matter]: ═════════════════════════════════════════════
[I][matter]:   Matter onboarding — commission via HA:
[I][matter]:     Manual pairing code:  34970112332
[I][matter]:     QR code payload:      MT:Y.K9042C00KA0648G00
[I][matter]:     Setup passcode:       20202021 (raw)
[I][matter]:     Discriminator:        3840
[I][matter]:     Vendor / Product:     0xFFF1 / 0x8000
[I][matter]: ═════════════════════════════════════════════
```

- **QR code**: paste the `MT:...` string into any QR generator (e.g. `https://project-chip.github.io/connectedhomeip/qrcode.html?data=MT:...`) and scan it with the controller app.
- **Manual code**: type the 11-digit code in the controller ("Add device → More options → Enter code").

### Wi-Fi with credentials in YAML / Ethernet

The device is already on the network. The controller discovers it via mDNS (on-network commissioning). The phone/hub must be on the **same L2 network with IPv6 enabled**.

### Wi-Fi with `ble_commissioning: true`

1. Flash a firmware **without** `wifi.ssid`/`networks` (only `ap:` for recovery).
2. Add the device in the controller app — the phone finds it over BLE.
3. The controller sends the Wi-Fi credentials; the device joins the network and finishes commissioning over IP.
4. The credentials are copied into ESPHome preferences, so after a reboot ESPHome's `wifi:` reconnects by itself.

## 2. Adding more controllers (multi-admin)

After the first pairing the original `setup_code` **no longer works** (Matter spec). To add a second ecosystem (e.g. Home Assistant + Apple Home):

- From the first controller: "Share device / Turn on pairing mode", **or**
- Press the `button: platform: matter` with `action: open_commissioning_window`. A new code is printed in the log:

```
[I][matter]:   Enhanced Commissioning Window open for 300s:
[I][matter]:     Manual pairing code:  ...
[I][matter]:     QR code payload:      MT:...
```

Up to 10 fabrics with PSRAM, 6 without.

## 3. Reset

- `button: platform: matter` with `action: factory_reset` — removes every fabric and reboots.
- Erasing the flash (`esptool.py erase_flash` and re-flashing) also clears the fabrics.

Removing the device from a controller only deletes that controller's fabric.

## 4. Controller notes

| Controller | Notes |
|---|---|
| Home Assistant | Requires the *Matter Server* add-on/container with IPv6 on the host network. |
| Apple Home | Needs a home hub (HomePod / Apple TV). Shows a "non-certified accessory" warning — accept it. |
| Google Home | Test VID/PID must be registered in the Google Home Developer Console for the account. |
| SmartThings | Enable developer mode to accept test devices. |
| eWeLink Cube / NSPanel Pro | Prefer `topology: composed` if bridged endpoints show as offline. |
| Alexa | Accepts test VIDs on most devices. |

## 5. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `The matter component requires IPv6` | Add `network: enable_ipv6: true`. |
| `requires framework 'esp-idf'` | Set `esp32.framework.type: esp-idf`. |
| Controller does not find the device | Phone/hub on another VLAN/SSID, IPv6 or mDNS blocked by the router (disable "multicast filtering"/"IGMP snooping" or enable mDNS reflector). |
| `skipping sensor '...' — no unit/device_class mapping` | The sensor's unit is not supported by Matter (see mapping table). Mark it `internal: true` to silence. |
| Boot loop / `bad_alloc` with many entities | Use a board with PSRAM and add `psram:`. |
| BLE commissioning fails at the Wi-Fi step | Use 2.4 GHz WPA2; avoid hidden SSIDs; add PSRAM on classic ESP32. |
| BLE component conflict error | Remove `esp32_ble_tracker`, `bluetooth_proxy`, `esp32_improv`, … when `ble_commissioning: true`. |
| Entities show as "Endpoint N" | Controller ignores FixedLabel in `composed` topology — use `bridge`. |
| Endpoint IDs changed after editing the YAML | Endpoints are assigned in scan order (switch, binary_sensor, sensor, cover, fan, lock, valve, select, button, light, climate). Adding/removing entities may shift IDs; some controllers then need the device to be re-added. |

First compile downloads esp-matter (~1 GB including connectedhomeip) and takes a long time; later builds are incremental.
