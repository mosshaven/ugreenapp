# Desktop App Shape

## Primary form

The main product should be a GUI app, not a pure CLI tool.

Reason:

- the app is centered around device state, ANC, EQ, OTA, prompts, battery, connection and feature toggles
- these flows are stateful and visual
- a CLI is useful for debugging and automation, but it is the wrong primary UX

## Recommended shape

- frontend: lightweight web UI
- shell: Tauri
- backend: Rust
- optional helper: separate CLI for protocol testing and device diagnostics

## Why not React Native desktop

- higher memory overhead
- unnecessary runtime complexity
- the Android app already separates into reusable protocol/business logic and a UI layer
- the desktop port only needs a compatible bridge/backend, not RN runtime parity

## Product split

### GUI app

Purpose:

- normal user-facing application
- connect to device
- show state
- control features
- run OTA
- locate earbuds
- manage settings

### Optional CLI

Purpose:

- scan devices
- inspect raw notifications
- send low-level commands
- dump parsed device info
- reproduce protocol bugs
- automate regression tests

Examples:

- `ugreenctl scan`
- `ugreenctl connect <mac>`
- `ugreenctl info`
- `ugreenctl anc off`
- `ugreenctl eq rock`
- `ugreenctl raw send GET_INFO`

## Backend modules

### `bluetooth_adapter`

Responsibilities:

- adapter state
- scanning
- connect / disconnect
- notifications
- write requests / commands

### `earbuds_protocol`

Responsibilities:

- protocol selection
- command building
- response parsing
- protocol-specific quirks

Mirrors JS concepts already found in Hermes:

- `buildCommand`
- `parseResponse`
- `getProtocolFacade`
- `getCurrentProtocolType`

### `device_service`

Responsibilities:

- high-level operations used by UI
- query and set commands
- state aggregation

### `ipc_bridge`

Responsibilities:

- exposes stable frontend API
- mirrors only the subset of the native bridge that the desktop UI actually needs

## Initial frontend/backend contract

This is the first useful desktop IPC surface inferred from Hermes call sites.

### Events

- `bluetooth.device_discovered`
- `bluetooth.connection_changed`
- `bluetooth.data_received`
- `bluetooth.adapter_state_changed`
- `nav.route_changed`

### Core commands

- `bluetooth.scan_start()`
- `bluetooth.scan_stop()`
- `bluetooth.connect(device_id)`
- `bluetooth.disconnect(device_id?)`
- `bluetooth.send_raw(payload)`

### Device queries

- `device.query_info(include_max_packet_length?: boolean)`
- `device.query_firmware_version()`
- `device.query_box_mac()`
- `device.query_connected_phones()`
- `device.query_phone_codec_type()`
- `device.query_high_quality_info()`
- `device.query_touch_sensitivity()`
- `device.query_find_status()`

## Signatures inferred from Hermes call sites

These are not final types, but they are strong enough for the first backend draft.

### `queryDeviceInfo`

Observed behavior:

- guarded by a lock
- timeout around 2000 ms
- sends command `GET_INFO`
- payload depends on protocol type

Observed inputs:

- optional boolean flag controlling whether `maxPacketLength` is requested
- optional extra argument used by call sites, likely request options/context

Observed request shape for non-UG3:

```json
{
  "list": [
    "maxPacketLength",
    "battery",
    "firmwareVersion",
    "ANCType",
    "ANCDepth",
    "equalizer",
    "gameMode",
    "deviceName",
    "boxMac",
    "windNoise",
    "highQualityDecoding",
    "inBox",
    "connectedDeviceInfo",
    "controlInfo",
    "spatialSoundEffects",
    "deviceDualConnection",
    "promptSoundInfo"
  ]
}
```

Observed request shape for UG3:

```json
{
  "list": [
    "INFO_BATTERY",
    "INFO_VERSION",
    "INFO_BLE_NAME",
    "INFO_EQ",
    "INFO_BUTTONS",
    "INFO_GAME",
    "INFO_PROMPT",
    "INFO_ANC",
    "INFO_ANC_DEPTH",
    "INFO_SPATIAL",
    "INFO_DUAL",
    "INFO_HQ",
    "INFO_BOX_MAC",
    "INFO_WIND_NOISE",
    "INFO_WEAR",
    "INFO_AI",
    "INFO_IN_BOX",
    "INFO_CONNECTED",
    "INFO_CUSTOM_EQ",
    "INFO_TOUCH_SENSITIVITY"
  ]
}
```

### `sendCommand`

Observed usage:

- shared generic transport helper
- called as `sendCommand(commandName, payload?, options?)`

Concrete observed command:

- `sendCommand("GET_INFO", request, extra)`

### `setAncMode`

Observed usage:

- `setAncMode("off")`
- `setAncMode("transparency")`
- `setAncMode("anc")`
- `setAncMode("anc", depth)`

Desktop backend should expose:

- `device.set_anc_mode(mode, depth?)`

### `setEqMode`

Observed usage:

- `setEqMode(eq_mode)`

Desktop backend should expose:

- `device.set_eq_mode(mode)`

### `setPromptLanguage`

Observed usage:

- `setPromptLanguage(sound_type)`

Likely values:

- Chinese
- English
- ring

Desktop backend should expose:

- `device.set_prompt_language(value)`

### `setPromptVolume`

Observed usage:

- `setPromptVolume(volume)`

Desktop backend should expose:

- `device.set_prompt_volume(value)`

### `setWearDetection`

Observed usage:

- `setWearDetection(value)`

Desktop backend should expose:

- `device.set_wear_detection(enabled_or_mode)`

### `factoryReset`

Observed usage:

- `factoryReset()`

After success the app also calls delete-device cleanup separately.

Desktop backend should expose:

- `device.factory_reset()`

## Practical next implementation target

If implementation starts now, the first milestone should be:

1. Bluetooth discovery and connection
2. `send_raw`
3. `query_info`
4. `query_firmware_version`
5. `set_anc_mode`
6. `set_eq_mode`
7. `set_prompt_language`
8. `set_prompt_volume`
9. `set_wear_detection`
10. `factory_reset`

That is enough to prove the architecture without React Native.
