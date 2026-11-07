# Referência de configuração

> 🇺🇸 English version: [configuration.md](configuration.md)

## Componente `matter:`

```yaml
matter:
  vendor_id: 0xFFF1
  product_id: 0x8000
  setup_code: "20202021"
  discriminator: 3840
  # opcionais
  vendor_name: "Unicontrol"
  product_name: "Matter Relay"
  node_label: "Relé da Sala"
  hardware_version_string: "rev-a"
  software_version_string: "1.0.0"
  ble_commissioning: false
  topology: bridge
```

| Opção | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `id` | ID | não | auto | ID da instância de `MatterComponent`. |
| `vendor_id` | hex uint16 | **sim** | — | Vendor ID Matter. Use `0xFFF1`–`0xFFF4` (VIDs de teste da CSA) durante o desenvolvimento. |
| `product_id` | hex uint16 | **sim** | — | Product ID Matter. Faixa de teste `0x8000`–`0x801F`. |
| `setup_code` | string (8 dígitos) | **sim** | — | Código de pareamento (PIN). Deve estar entre `00000001` e `99999998` e **não** pode ser um valor reservado (`00000000`, `11111111` … `99999999`, `12345678`, `87654321`). Use aspas para preservar zeros à esquerda. |
| `discriminator` | inteiro `0`–`4095` | **sim** | — | Discriminador de 12 bits usado para localizar o dispositivo no comissionamento. Use valores diferentes para dispositivos na mesma rede. |
| `vendor_name` | string ≤ 32 | não | `""` | `BasicInformation.VendorName`. Vazio mantém o padrão do SDK. |
| `product_name` | string ≤ 32 | não | `""` | `BasicInformation.ProductName`. |
| `node_label` | string ≤ 32 | não | `""` | `BasicInformation.NodeLabel` (nome visível do dispositivo). |
| `hardware_version_string` | string ≤ 64 | não | `""` | `BasicInformation.HardwareVersionString`. |
| `software_version_string` | string ≤ 64 | não | `""` | `BasicInformation.SoftwareVersionString`. |
| `ble_commissioning` | bool | não | `false` | Habilita o comissionamento Matter via BLE (o controlador envia as credenciais Wi-Fi por BLE). Exige `wifi:`. |
| `topology` | `bridge` \| `composed` | não | `bridge` | Organização dos endpoints — veja [Topologia](#topologia). |

### Validações feitas na configuração

O componente recusa a compilação quando:

- o framework não é `esp-idf`;
- a variante do chip não é `esp32`, `esp32s3`, `esp32c3`, `esp32c6` ou `esp32h2`;
- falta `network: enable_ipv6: true` (Matter funciona apenas sobre IPv6);
- não há `wifi:` nem `ethernet:`;
- `ble_commissioning: true` sem `wifi:`;
- `ble_commissioning: true` junto com `esp32_ble`, `esp32_ble_tracker`, `esp32_ble_beacon`, `bluetooth_proxy`, `improv_serial` ou `esp32_improv`;
- a quantidade de entidades expostas exige mais de 255 endpoints Matter.

### Topologia

| Valor | Estrutura | Quando usar |
|---|---|---|
| `bridge` (padrão) | `root (0)` → `Aggregator (1)` → um *Bridged Node* por entidade, cada um com `BridgedDeviceBasicInformation.NodeLabel` = nome da entidade. | Muitas entidades independentes. Cada uma aparece como um acessório separado no Apple Home / Google Home / SmartThings / Home Assistant. |
| `composed` | `root (0)` → endpoints das entidades diretamente (dispositivo composto). | Um único produto com várias funções (ex.: interruptor de 2 teclas) ou controladores que não lidam bem com bridges (eWeLink Cube / NSPanel Pro). Alguns controladores mostram nomes genéricos, pois só o `FixedLabel` carrega o nome da entidade. |

Todo endpoint também recebe um cluster `FixedLabel` (`name` = nome da entidade, truncado em 16 bytes) e um cluster `UserLabel` gravável, persistido na NVS.

## Mapeamento Entidade → Matter

Toda entidade **não interna** é exposta automaticamente. Use `internal: true` para esconder uma entidade do Matter (isso também a esconde da API nativa do Home Assistant).

| Domínio ESPHome | Tipo de dispositivo Matter | Observações |
|---|---|---|
| `switch` | On/Off Plug-in Unit (0x010A) | Liga/desliga bidirecional. |
| `light` (binária) | On/Off Light (0x0100) | |
| `light` (com brilho) | Dimmable Light (0x0101) | OnOff + LevelControl. |
| `light` (temperatura de cor) | Color Temperature Light (0x010C) | + ColorControl (mireds). |
| `light` (RGB) | Extended Color Light (0x010D) | |
| `binary_sensor` | Contact Sensor (0x0015) | Padrão para qualquer `device_class`. |
| `binary_sensor` com `device_class: motion` / `occupancy` / `presence` | Occupancy Sensor (0x0107) | |
| `sensor` `°C` / `device_class: temperature` | Temperature Sensor (0x0302) | |
| `sensor` `%` + `device_class: humidity` | Humidity Sensor (0x0307) | Só `%` é ambíguo (bateria) e é ignorado. |
| `sensor` `hPa` / `kPa` / `device_class: pressure`/`atmospheric_pressure` | Pressure Sensor (0x0305) | |
| `sensor` `lx` / `device_class: illuminance` | Light Sensor (0x0106) | |
| `sensor` `m³/h` | Flow Sensor (0x0306) | |
| outros `sensor` | — | Ignorados com aviso no log. |
| `cover` | Window Covering (0x0202) | Lift + posição (0–100 %). |
| `fan` | Fan (0x002B) | Liga/desliga, percentual; MultiSpeed quando `speed_count` está definido. |
| `lock` | Door Lock (0x000A) | Comandos Lock / Unlock. |
| `valve` | Water Valve (0x0042) | Abrir / fechar. |
| `select` | Mode Select (0x0027) | Cada opção vira um modo. |
| `climate` | Thermostat (0x0301) | Precisa de HEAT ou COOL. AUTO exige temperatura alvo de dois pontos. |
| `button` | Generic Switch (0x000F) | Emite eventos `InitialPress` + `ShortRelease` — use como gatilho de automação. |
| `button` `platform: matter` | — | **Não** é exposto; ação de gerenciamento local (veja abaixo). |

## `button: platform: matter`

Botões que executam ações de gerenciamento Matter **no próprio dispositivo**. Ficam visíveis no ESPHome / Home Assistant / web_server, mas nunca são expostos na fabric Matter.

```yaml
button:
  - platform: matter
    name: "Abrir Janela de Pareamento Matter"
    action: open_commissioning_window
    timeout: 5min
  - platform: matter
    name: "Reset de Fábrica Matter"
    action: factory_reset
```

| Opção | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `action` | `open_commissioning_window` \| `factory_reset` | **sim** | Ação executada ao pressionar. |
| `timeout` | tempo `180s`–`900s` | não (padrão `300s`) | Apenas para `open_commissioning_window`. |
| todas as opções de [button](https://esphome.io/components/button/) | | | `name`, `icon`, `entity_category`, … |

- **`open_commissioning_window`** — abre uma *Enhanced Commissioning Window* com **novo passcode e discriminador aleatórios**, para adicionar um segundo controlador (multi-admin). O novo código manual e o payload do QR code são impressos no log.
- **`factory_reset`** — remove todas as fabrics Matter e reinicia. O dispositivo volta ao estado inicial de pareamento.

### Chamando a partir de uma lambda

```yaml
on_press:
  - lambda: |-
      esphome::matter::MatterComponent::instance()->open_commissioning_window(300);
```

```yaml
  - lambda: |-
      esphome::matter::MatterComponent::instance()->factory_reset();
```

## Configuração automática do SDK

O componente define automaticamente as opções de `sdkconfig` do ESP-IDF / CHIP, incluindo:

- `CONFIG_ESP_MATTER_MAX_DYNAMIC_ENDPOINT_COUNT` — calculado a partir do número de entidades (+2 root/aggregator, +4 de folga, mínimo 16). Se você definir esse valor em `esp32.framework.sdkconfig_options`, o componente respeita a sua escolha.
- Drivers de comissionamento Wi-Fi / Ethernet conforme o transporte configurado.
- NimBLE + CHIPoBLE apenas quando `ble_commissioning: true`.
- `CONFIG_MAX_FABRICS` = 10 com PSRAM, 6 sem PSRAM.
- Com `psram:` presente: `SPIRAM_USE_MALLOC` (alocações do CHIP vão para a PSRAM) e `CHIP_SYSTEM_CONFIG_POOL_USE_HEAP`.
- Transporte TCP, intervalos de retransmissão MRP maiores e fila de eventos maior — ajustados para bridges com muitos endpoints e vários controladores.
- IPv4 mantido habilitado no CHIP (alguns hubs só alcançam o dispositivo via IPv4).
