# Earbuds Protocol Layer

This document captures the protocol-specific findings extracted from the Hermes bundle.

## Core idea

The desktop backend does not need to imitate the whole Android stack.

It mainly needs:

- transport
- protocol selection
- protocol encode/decode
- high-level device service on top

In the Hermes bundle:

- `buildCommand(command, payload)` delegates to `protocolFacade.encode(...)`
- `parseResponse(buffer)` delegates to `protocolFacade.decode(...)`

So the actual heart of the port is a protocol facade implementation for the supported protocol families.

## Protocol types

The bundle defines the following canonical protocol ids:

- `1 = UG1`
- `2 = UG2`
- `3 = UG3`

There is also compatibility handling for legacy type `4`:

- `4` is treated as old Zhongke Lanxun
- it is automatically normalized to `UG2`

This behavior is explicitly implemented in Hermes.

## Protocol mapping

### `PROTOCOL_TYPE_TO_NAME`

```json
{
  "1": "UG1",
  "2": "UG2",
  "3": "UG3"
}
```

### `PROTOCOL_NAME_TO_TYPE`

The bundle maps several aliases to canonical protocol ids:

- `UG1 -> 1`
- `UG2 -> 2`
- `UG3 -> 3`
- `Universal -> 1`
- `UGREEN_Universal -> 1`
- `Magic -> 2`
- `Zhongke_Lanxun -> 2`
- `Standard -> 3`
- `UG_Universal_Protocol -> 3`

## Protocol display names

- `1 -> UG1 协议`
- `2 -> UG2 协议`
- `3 -> UG3 协议`

## Protocol feature model

The bundle contains a protocol-level feature map.

### UG1

- anc: true
- gameMode: true
- eq: true
- dualLink: true
- find: true
- spatialAudio: false
- wearDetection: false
- highQuality: false
- windNoise: false
- audioShare: false
- ai: false

### UG2

- anc: true
- gameMode: true
- eq: true
- dualLink: true
- find: true
- spatialAudio: true
- wearDetection: false
- highQuality: true
- windNoise: true
- audioShare: false
- ai: false

### UG3

- anc: true
- gameMode: true
- eq: true
- dualLink: true
- find: true
- spatialAudio: true
- wearDetection: true
- highQuality: true
- windNoise: true
- audioShare: true
- ai: true

## `normalizeProtocolType`

Observed behavior:

- missing protocol defaults to `UG1`
- valid values `1/2/3` pass through
- legacy `4` becomes `UG2`
- any other invalid type falls back to `UG1`

Desktop backend should preserve this for compatibility.

## `getProtocolName` and `getProtocolType`

Observed behavior:

- unknown numeric type in name lookup throws in one helper layer
- higher-level helpers use safe fallbacks
- unknown name falls back to `UG1`

Recommended backend policy:

- strict mode in internal core
- forgiving mode at public API boundary

## Initialization flow

Important exported functions:

- `initEarbudsProtocol(protocolType)`
- `initDeviceConfig(config)`
- `initFromAPI(apiData)`
- `initWithLocalOrCustom(configOrName)`
- `smartInitEarbuds(protocolType, overrides?)`

Observed behavior:

- `initFromAPI` looks for `protocol_type` or `protocolType`
- if absent, defaults to `1`
- `smartInitEarbuds` normalizes protocol id, gets recommended config by protocol, merges overrides, then initializes

## `buildCommand`

Observed implementation:

- fails if protocol is not initialized
- then calls `protocolFacade.encode(command, payload)`

Desktop interpretation:

- each protocol implementation must expose `encode(command, payload)`

## `parseResponse`

Observed implementation:

- fails if protocol is not initialized
- then calls `protocolFacade.decode(buffer)`

Desktop interpretation:

- each protocol implementation must expose `decode(buffer)`

## Bluetooth listener path

The bundle's Bluetooth listener path does:

1. receive raw hex from Bluetooth bridge event
2. convert hex string to buffer
3. call `protocolFacade.decode(buffer)`
4. log decoded data
5. strip metadata keys `success`, `command`, `raw`
6. merge remaining keys into device store
7. add synthetic timestamps for:
   - `aiRecording -> aiRecordingTimestamp`
   - `findLeft/findRight -> findTimestamp`

This is important: decode results are not only command responses, they are also state-update deltas.

## High-level implication for backend

The backend should be structured around a protocol facade trait/interface:

```text
ProtocolFacade
  encode(command, payload) -> bytes
  decode(bytes) -> DecodedFrame
```

Where `DecodedFrame` should at minimum contain:

- `success`
- `command`
- `raw`
- `data`

And the `data` object should be normalized into state updates for the frontend/store.

## `queryDeviceInfo` and protocol split

The bundle sends different info request lists depending on protocol family.

### UG1 / UG2 style request

Command:

- `GET_INFO`

Request payload:

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

### UG3 style request

Command:

- `GET_INFO`

Request payload:

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

## Practical implementation order

Recommended backend order:

1. shared transport API
2. protocol type normalization helpers
3. `UG1` facade
4. `UG2` facade
5. `UG3` facade
6. state delta application logic
7. high-level query/set operations

## What remains unknown

Still not fully extracted:

- binary packet structure for each protocol
- full command enum for every operation
- exact decoded response schema per command

But the architecture is now clear enough to start building:

- protocol family routing
- encode/decode abstraction
- transport/backend shell
