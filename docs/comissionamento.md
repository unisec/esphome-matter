# Comissionamento e solução de problemas

> 🇺🇸 English version: [commissioning.md](commissioning.md)

## 1. Primeiro pareamento

Depois de gravar o firmware, abra os logs (`esphome logs dispositivo.yaml`). No boot, o componente imprime os dados de pareamento:

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

- **QR code**: cole a string `MT:...` em um gerador de QR (ex.: `https://project-chip.github.io/connectedhomeip/qrcode.html?data=MT:...`) e escaneie com o app do controlador.
- **Código manual**: digite o código de 11 dígitos no controlador ("Adicionar dispositivo → Mais opções → Inserir código").

### Wi-Fi com credenciais no YAML / Ethernet

O dispositivo já está na rede. O controlador o encontra via mDNS (comissionamento on-network). O celular/hub precisa estar na **mesma rede L2 com IPv6 habilitado**.

### Wi-Fi com `ble_commissioning: true`

1. Grave um firmware **sem** `wifi.ssid`/`networks` (apenas `ap:` para recuperação).
2. Adicione o dispositivo no app do controlador — o celular o encontra via BLE.
3. O controlador envia as credenciais Wi-Fi; o dispositivo entra na rede e conclui o comissionamento via IP.
4. As credenciais são copiadas para as preferências do ESPHome, então após reiniciar o `wifi:` do ESPHome reconecta sozinho.

## 2. Adicionando mais controladores (multi-admin)

Depois do primeiro pareamento o `setup_code` original **deixa de funcionar** (especificação Matter). Para adicionar um segundo ecossistema (ex.: Home Assistant + Apple Home):

- Pelo primeiro controlador: "Compartilhar dispositivo / Ativar modo de pareamento", **ou**
- Pressione o `button: platform: matter` com `action: open_commissioning_window`. Um novo código é impresso no log:

```
[I][matter]:   Enhanced Commissioning Window open for 300s:
[I][matter]:     Manual pairing code:  ...
[I][matter]:     QR code payload:      MT:...
```

Até 10 fabrics com PSRAM, 6 sem PSRAM.

## 3. Reset

- `button: platform: matter` com `action: factory_reset` — remove todas as fabrics e reinicia.
- Apagar a flash (`esptool.py erase_flash` e regravar) também limpa as fabrics.

Remover o dispositivo de um controlador apaga apenas a fabric daquele controlador.

## 4. Observações por controlador

| Controlador | Observações |
|---|---|
| Home Assistant | Requer o add-on/container *Matter Server* com IPv6 na rede do host. |
| Apple Home | Precisa de uma central (HomePod / Apple TV). Exibe aviso de "acessório não certificado" — aceite. |
| Google Home | O VID/PID de teste precisa estar cadastrado no Google Home Developer Console da conta. |
| SmartThings | Ative o modo desenvolvedor para aceitar dispositivos de teste. |
| eWeLink Cube / NSPanel Pro | Prefira `topology: composed` se os endpoints da bridge aparecerem offline. |
| Alexa | Aceita VIDs de teste na maioria dos dispositivos. |

## 5. Solução de problemas

| Sintoma | Causa provável / solução |
|---|---|
| `The matter component requires IPv6` | Adicione `network: enable_ipv6: true`. |
| `requires framework 'esp-idf'` | Defina `esp32.framework.type: esp-idf`. |
| Controlador não encontra o dispositivo | Celular/hub em outra VLAN/SSID, IPv6 ou mDNS bloqueado no roteador (desative "multicast filtering"/"IGMP snooping" ou habilite um refletor mDNS). |
| `skipping sensor '...' — no unit/device_class mapping` | A unidade do sensor não é suportada pelo Matter (veja a tabela de mapeamento). Use `internal: true` para silenciar. |
| Boot loop / `bad_alloc` com muitas entidades | Use uma placa com PSRAM e adicione `psram:`. |
| Comissionamento BLE falha na etapa do Wi-Fi | Use 2.4 GHz WPA2; evite SSID oculto; adicione PSRAM no ESP32 clássico. |
| Erro de conflito com componente BLE | Remova `esp32_ble_tracker`, `bluetooth_proxy`, `esp32_improv`, … quando `ble_commissioning: true`. |
| Entidades aparecem como "Endpoint N" | O controlador ignora o FixedLabel na topologia `composed` — use `bridge`. |
| IDs de endpoint mudaram após editar o YAML | Os endpoints são atribuídos na ordem de varredura (switch, binary_sensor, sensor, cover, fan, lock, valve, select, button, light, climate). Adicionar/remover entidades pode deslocar os IDs; alguns controladores precisam que o dispositivo seja adicionado novamente. |

A primeira compilação baixa o esp-matter (~1 GB incluindo o connectedhomeip) e demora bastante; as seguintes são incrementais.
