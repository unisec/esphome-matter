# ESPHome Matter

**[English](#english) · [Português (BR)](#português-br)**

---

## English

External [ESPHome](https://esphome.io) component that turns an ESP32 into a native **[Matter](https://csa-iot.org/all-solutions/matter/)** device using Espressif's [esp-matter](https://github.com/espressif/esp-matter) SDK (v1.6.0).

Your existing ESPHome entities (switches, lights, sensors, covers, …) are **exposed automatically** as Matter endpoints — just add a `matter:` block. The device can be controlled locally by Apple Home, Google Home, Amazon Alexa, Home Assistant, SmartThings and any other Matter controller, while still working with the ESPHome native API, OTA, web server, etc.

### Features

- Automatic mapping of ESPHome entities to Matter device types (11 domains).
- **Bridge** (one accessory per entity) or **composed** (single device) topology.
- Transport over **Wi-Fi** or **Ethernet** (internal EMAC or SPI such as W5500).
- Optional Matter **BLE commissioning** (device ships without Wi-Fi credentials).
- **Multi-admin**: open a new pairing window from a button or lambda.
- Factory reset of Matter fabrics.
- Customizable Basic Information (vendor/product name, node label, versions).
- Writable `UserLabel` persisted in NVS + `FixedLabel` with the entity name.
- SDK `sdkconfig` tuned automatically (endpoint count, PSRAM, BLE, TCP, MRP).

### Requirements

| Item | Requirement |
|---|---|
| ESPHome | 2026.3 or newer (tested with 2026.10) |
| Framework | `esp-idf` (Arduino is not supported) |
| Chip | ESP32, ESP32-S3, ESP32-C3, ESP32-C6, ESP32-H2 |
| Flash | 8 MB recommended (4 MB is very tight) |
| PSRAM | Recommended for bridges with many entities and for BLE commissioning |
| Network | IPv6 enabled (`network: enable_ipv6: true`) and mDNS allowed on the LAN |

### Supported entities

| ESPHome | Matter device type |
|---|---|
| `switch` | On/Off Plug-in Unit |
| `light` | On/Off, Dimmable, Color Temperature or Extended Color Light (auto-detected) |
| `binary_sensor` | Contact Sensor · Occupancy Sensor (`motion`/`occupancy`/`presence`) |
| `sensor` | Temperature (°C) · Humidity (%, `humidity`) · Pressure (hPa/kPa) · Illuminance (lx) · Flow (m³/h) |
| `cover` | Window Covering (position) |
| `fan` | Fan (on/off, percent, multi-speed) |
| `lock` | Door Lock |
| `valve` | Water Valve |
| `select` | Mode Select |
| `climate` | Thermostat (heat / cool / auto) |
| `button` | Generic Switch (press events) |

Entities with `internal: true` are **not** exposed. Full details: [docs/configuration.md](docs/configuration.md).

### Quick start

1. Add the component and the required blocks to your YAML:

```yaml
esp32:
  board: esp32-c6-devkitc-1
  variant: esp32c6
  flash_size: 8MB
  framework:
    type: esp-idf            # required

external_components:
  - source: github://unisec/esphome-matter@main
    components: [matter]

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

network:
  enable_ipv6: true          # required — Matter is IPv6-only

matter:
  vendor_id: 0xFFF1          # CSA test vendor
  product_id: 0x8000
  setup_code: "20202021"     # 8-digit pairing PIN
  discriminator: 3840        # 0..4095, unique per device on the LAN
  vendor_name: "Unicontrol"
  product_name: "Matter Relay"

switch:
  - platform: gpio
    pin: GPIO18
    name: "Relay"            # → exposed as a Matter plug

button:
  - platform: matter         # management button (not exposed to Matter)
    name: "Open Matter Pairing Window"
    action: open_commissioning_window
```

2. Build and flash: `esphome run device.yaml` (the first build downloads esp-matter and takes a while).
3. Read the QR code payload / manual pairing code from the log and add the device in your Matter controller.

See [docs/commissioning.md](docs/commissioning.md) for pairing, multi-admin, reset and troubleshooting.

### `matter:` options

| Option | Required | Default | Description |
|---|---|---|---|
| `vendor_id` | yes | — | Matter Vendor ID (hex). |
| `product_id` | yes | — | Matter Product ID (hex). |
| `setup_code` | yes | — | 8-digit passcode, reserved values rejected. |
| `discriminator` | yes | — | 12-bit discriminator (0–4095). |
| `vendor_name` / `product_name` / `node_label` | no | `""` | Basic Information strings (≤ 32 chars). |
| `hardware_version_string` / `software_version_string` | no | `""` | Basic Information strings (≤ 64 chars). |
| `ble_commissioning` | no | `false` | Commission over BLE (Wi-Fi only). |
| `topology` | no | `bridge` | `bridge` or `composed`. |

### `button: platform: matter`

| Option | Required | Description |
|---|---|---|
| `action` | yes | `open_commissioning_window` or `factory_reset`. |
| `timeout` | no | Pairing window duration, 180 s – 900 s (default 300 s). |

### Examples

| File | Description |
|---|---|
| [examples/wifi-basic.yaml](examples/wifi-basic.yaml) | ESP32-C6 over Wi-Fi: relay, contact sensor, temperature, management buttons. |
| [examples/wifi-ble-commissioning.yaml](examples/wifi-ble-commissioning.yaml) | ESP32-S3 + PSRAM, RGB light, BLE commissioning (no Wi-Fi credentials in YAML). |
| [examples/ethernet-bridge.yaml](examples/ethernet-bridge.yaml) | ESP32-S3 + W5500 bridge exposing every supported entity type. |
| [examples/composed-device.yaml](examples/composed-device.yaml) | ESP32-C3 2-gang switch using `topology: composed`. |

Copy `examples/secrets.yaml.example` to `secrets.yaml` before compiling the examples.

### Repository layout

```
components/matter/          ESPHome external component
  __init__.py               schema, validation, sdkconfig generation
  button/                   `button: platform: matter`
  matter_component.*        root node, aggregator, scanning, dispatch
  matter_*_endpoint.*       one wrapper per ESPHome domain
  matter_ethernet_stub.cpp  lets ESPHome's ethernet: own the PHY
  external_platform/        CHIP external platform manifest
docs/                       configuration & commissioning guides (EN / PT-BR)
examples/                   ready-to-use YAML examples
```

### Limitations

- Not Matter-certified: uses test VIDs and test DAC certificates — for development, DIY and internal use. Commercial products need CSA certification and their own attestation credentials.
- No Thread transport (Wi-Fi / Ethernet only), even on ESP32-C6/H2.
- No Matter OTA (use ESPHome OTA).
- Endpoint IDs follow entity order; reshuffling entities may require re-adding the device in some controllers.

### License

No license has been defined for this repository yet. esp-matter and connectedhomeip are distributed under the Apache 2.0 license.

---

## Português (BR)

Componente externo do [ESPHome](https://esphome.io) que transforma um ESP32 em um dispositivo **[Matter](https://csa-iot.org/all-solutions/matter/)** nativo, usando o SDK [esp-matter](https://github.com/espressif/esp-matter) (v1.6.0) da Espressif.

As entidades que você já tem no ESPHome (switches, luzes, sensores, cortinas, …) são **expostas automaticamente** como endpoints Matter — basta adicionar o bloco `matter:`. O dispositivo pode ser controlado localmente pelo Apple Home, Google Home, Amazon Alexa, Home Assistant, SmartThings e qualquer outro controlador Matter, continuando a funcionar com a API nativa do ESPHome, OTA, web server etc.

### Funcionalidades

- Mapeamento automático das entidades ESPHome para tipos de dispositivo Matter (11 domínios).
- Topologia **bridge** (um acessório por entidade) ou **composed** (dispositivo único).
- Transporte via **Wi-Fi** ou **Ethernet** (EMAC interno ou SPI como W5500).
- **Comissionamento BLE** opcional (o dispositivo sai de fábrica sem credenciais Wi-Fi).
- **Multi-admin**: abre uma nova janela de pareamento por botão ou lambda.
- Reset de fábrica das fabrics Matter.
- Basic Information personalizável (fabricante/produto, node label, versões).
- `UserLabel` gravável persistido na NVS + `FixedLabel` com o nome da entidade.
- `sdkconfig` do SDK ajustado automaticamente (quantidade de endpoints, PSRAM, BLE, TCP, MRP).

### Requisitos

| Item | Requisito |
|---|---|
| ESPHome | 2026.3 ou mais recente (testado com 2026.10) |
| Framework | `esp-idf` (Arduino não é suportado) |
| Chip | ESP32, ESP32-S3, ESP32-C3, ESP32-C6, ESP32-H2 |
| Flash | 8 MB recomendado (4 MB fica muito apertado) |
| PSRAM | Recomendada para bridges com muitas entidades e para comissionamento BLE |
| Rede | IPv6 habilitado (`network: enable_ipv6: true`) e mDNS liberado na LAN |

### Entidades suportadas

| ESPHome | Tipo de dispositivo Matter |
|---|---|
| `switch` | On/Off Plug-in Unit (tomada) |
| `light` | On/Off, Dimmable, Color Temperature ou Extended Color Light (detecção automática) |
| `binary_sensor` | Contact Sensor · Occupancy Sensor (`motion`/`occupancy`/`presence`) |
| `sensor` | Temperatura (°C) · Umidade (%, `humidity`) · Pressão (hPa/kPa) · Luminosidade (lx) · Vazão (m³/h) |
| `cover` | Window Covering (cortina/persiana com posição) |
| `fan` | Fan (liga/desliga, percentual, multivelocidade) |
| `lock` | Door Lock (fechadura) |
| `valve` | Water Valve (válvula) |
| `select` | Mode Select |
| `climate` | Thermostat (aquecer / resfriar / auto) |
| `button` | Generic Switch (eventos de clique) |

Entidades com `internal: true` **não** são expostas. Detalhes completos: [docs/configuracao.md](docs/configuracao.md).

### Início rápido

1. Adicione o componente e os blocos obrigatórios no seu YAML:

```yaml
esp32:
  board: esp32-c6-devkitc-1
  variant: esp32c6
  flash_size: 8MB
  framework:
    type: esp-idf            # obrigatório

external_components:
  - source: github://unisec/esphome-matter@main
    components: [matter]

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

network:
  enable_ipv6: true          # obrigatório — Matter usa apenas IPv6

matter:
  vendor_id: 0xFFF1          # vendor de teste da CSA
  product_id: 0x8000
  setup_code: "20202021"     # PIN de pareamento com 8 dígitos
  discriminator: 3840        # 0..4095, único por dispositivo na rede
  vendor_name: "Unicontrol"
  product_name: "Matter Relay"

switch:
  - platform: gpio
    pin: GPIO18
    name: "Relé"             # → exposto como tomada Matter

button:
  - platform: matter         # botão de gerenciamento (não exposto no Matter)
    name: "Abrir Janela de Pareamento Matter"
    action: open_commissioning_window
```

2. Compile e grave: `esphome run dispositivo.yaml` (a primeira compilação baixa o esp-matter e demora).
3. Pegue o QR code / código manual de pareamento no log e adicione o dispositivo no seu controlador Matter.

Veja [docs/comissionamento.md](docs/comissionamento.md) para pareamento, multi-admin, reset e solução de problemas.

### Opções de `matter:`

| Opção | Obrigatório | Padrão | Descrição |
|---|---|---|---|
| `vendor_id` | sim | — | Vendor ID Matter (hex). |
| `product_id` | sim | — | Product ID Matter (hex). |
| `setup_code` | sim | — | Passcode de 8 dígitos; valores reservados são rejeitados. |
| `discriminator` | sim | — | Discriminador de 12 bits (0–4095). |
| `vendor_name` / `product_name` / `node_label` | não | `""` | Strings de Basic Information (≤ 32 caracteres). |
| `hardware_version_string` / `software_version_string` | não | `""` | Strings de Basic Information (≤ 64 caracteres). |
| `ble_commissioning` | não | `false` | Comissionamento via BLE (apenas Wi-Fi). |
| `topology` | não | `bridge` | `bridge` ou `composed`. |

### `button: platform: matter`

| Opção | Obrigatório | Descrição |
|---|---|---|
| `action` | sim | `open_commissioning_window` ou `factory_reset`. |
| `timeout` | não | Duração da janela de pareamento, 180 s – 900 s (padrão 300 s). |

### Exemplos

| Arquivo | Descrição |
|---|---|
| [examples/wifi-basic.yaml](examples/wifi-basic.yaml) | ESP32-C6 via Wi-Fi: relé, sensor de contato, temperatura, botões de gerenciamento. |
| [examples/wifi-ble-commissioning.yaml](examples/wifi-ble-commissioning.yaml) | ESP32-S3 + PSRAM, luz RGB, comissionamento BLE (sem credenciais Wi-Fi no YAML). |
| [examples/ethernet-bridge.yaml](examples/ethernet-bridge.yaml) | Bridge ESP32-S3 + W5500 expondo todos os tipos de entidade suportados. |
| [examples/composed-device.yaml](examples/composed-device.yaml) | Interruptor de 2 teclas com ESP32-C3 usando `topology: composed`. |

Copie `examples/secrets.yaml.example` para `secrets.yaml` antes de compilar os exemplos.

### Estrutura do repositório

```
components/matter/          componente externo do ESPHome
  __init__.py               schema, validação, geração do sdkconfig
  button/                   `button: platform: matter`
  matter_component.*        root node, aggregator, varredura, despacho
  matter_*_endpoint.*       um wrapper por domínio do ESPHome
  matter_ethernet_stub.cpp  deixa o ethernet: do ESPHome controlar o PHY
  external_platform/        manifesto de plataforma externa do CHIP
docs/                       guias de configuração e comissionamento (EN / PT-BR)
examples/                   exemplos YAML prontos para uso
```

### Limitações

- Não é certificado Matter: usa VIDs e certificados DAC de teste — para desenvolvimento, DIY e uso interno. Produtos comerciais precisam de certificação da CSA e credenciais de atestação próprias.
- Sem transporte Thread (apenas Wi-Fi / Ethernet), mesmo no ESP32-C6/H2.
- Sem OTA Matter (use o OTA do ESPHome).
- Os IDs de endpoint seguem a ordem das entidades; reorganizar entidades pode exigir readicionar o dispositivo em alguns controladores.

### Licença

Este repositório ainda não tem uma licença definida. O esp-matter e o connectedhomeip são distribuídos sob a licença Apache 2.0.
