# UGREEN Earbuds Desktop Bridge

This note captures the bridge contract extracted from the Android APK and Hermes bundle for the `Headphones` React Native module. The goal is to reimplement the native side for Linux/Windows without shipping a React Native desktop app.

## Key conclusion

The desktop port does not need React Native as a runtime.

The heavy Android/RN stack can be split into:

- a lightweight desktop backend that implements the native bridge contract
- a lightweight frontend, preferably plain web UI in Tauri or another thin shell
- optional reuse of `assets/static/device` where practical

Most earbuds business logic already exists in the Hermes bundle and can be mirrored or gradually replaced.

## Native modules referenced by the Hermes bundle

The `Headphones` bundle directly references these native modules:

- `UGRNBluetoothBridge`
- `UGRNUniversalModule`
- `UGRNNavigationBridge`
- `UGRNLogModule`

The desktop port only needs compatible replacements for these interfaces. `UGRNLogModule` is optional.

## Bluetooth bridge contract

`UGRNBluetoothBridge` is wrapped by a JS `BluetoothBridge` helper created from `NativeModules + NativeEventEmitter`.

### Expected event names

- `onBluetoothDataReceived`
- `onBluetoothConnectionStateChanged`
- `onBluetoothDeviceDiscovered`
- `onBluetoothEnableStateChanged`

### JS-side behavior

The helper:

- stores `bridge`
- creates `emitter = new NativeEventEmitter(bridge)`
- registers listeners for the four events above
- treats missing bridge as a fatal capability error

### Desktop implication

A desktop backend must expose:

- device discovery stream
- connection state stream
- raw protocol data receive stream
- adapter enabled/disabled state stream

## High-level earbuds SDK surface used by the bundle

The Hermes bundle exports and uses a stable-looking SDK surface. These names are visible in the bundle export table and call sites.

### Core plumbing

- `setBluetoothBridge`
- `getBluetoothBridge`
- `registerBluetoothListener`
- `sendCommand`
- `buildCommand`
- `parseResponse`
- `getProtocolFacade`
- `getCurrentProtocolType`
- `initEarbudsProtocol`
- `initDeviceConfig`
- `smartInitEarbuds`
- `initWithLocalOrCustom`
- `loadDeviceConfigByName`
- `getAllSupportedDevices`
- `checkDeviceSupport`

### Queries

- `queryDeviceInfo`
- `queryFirmwareVersion`
- `queryBoxMac`
- `queryConnectedPhones`
- `queryPhoneCodecType`
- `queryHighQualityInfo`
- `queryTouchSensitivity`
- `queryFindStatus`

### State-changing commands

- `setEqMode`
- `setCustomEQ`
- `deleteCustomEQ`
- `setGameMode`
- `setSpatialAudio`
- `setAncMode`
- `setWindNoise`
- `setDualLink`
- `setFindEarbuds`
- `setHighQuality`
- `setWearDetection`
- `setTouchSensitivity`
- `setAudioShare`
- `setAI`
- `setPromptLanguage`
- `setPromptVolume`

### Maintenance / lifecycle

- `factoryReset`
- `otaDisconnect`
- `batchCommands`
- `resetAllSettings`
- `resetEarbudsStores`

## Device and protocol state visible in JS

The bundle maintains a `useDeviceInfoStore` and related selectors/actions. Important fields seen in the bundle:

- `deviceUniqueCode`
- `macAddress`
- `deviceName`
- `deviceCustomName`
- `productSerialNo`
- `boxMac`
- `phoneBluetoothName`
- `protocolType`
- `firmwareVersion`
- `hardwareVersion`

The device config database inside the bundle also includes many feature flags, for example:

- `isNoiseCancellingEnabled`
- `isGameMode`
- `isSpatialAudioEnabled`
- `isEqualizerEnabled`
- `isDualDevicePairingEnabled`
- `isHeadphoneFinderEnabled`
- `isHighQualityDecodingEnabled`
- `isWearDetectionEnabled`
- `isWindNoiseEnabled`
- `isAudioSharingAvailable`
- `isAiAssistantEnabled`
- `isOverEarHeadphones`
- `hasAncButton`
- `hasBox`
- `isVoicePromptEnabled`
- `isFirmwareUpgradeEnabled`

## Navigation bridge

`UGRNNavigationBridge` is referenced as a native service for route control and back navigation.

The JS side clearly uses:

- `navigate(...)`
- `goBack()`

There is also explicit handling for the case when `UGRNNavigationBridge` is unavailable.

### Desktop implication

For a non-RN desktop app this bridge can be much smaller:

- route push / replace
- go back
- optional route change event

This can be mapped to frontend routing instead of a native module.

## Universal module

`UGRNUniversalModule` is referenced from `NativeModules` and appears to act as a shared utility/native service.

At this stage it should be treated as a compatibility bucket for:

- platform helpers
- device or app metadata
- native features not specific to raw Bluetooth transport

The exact minimal subset should be extracted later from concrete call sites.

## Logging bridge

`UGRNLogModule` is used only for optional log forwarding. The JS logger:

- hooks `console.log/info/warn/error`
- forwards messages to `UGRNLogModule.log({ type, message })`

### Desktop implication

Not required for a first working port. Plain local logging is enough.

## Architecture recommendation for Linux/Windows

Recommended target:

- frontend: plain web app, preferably not React Native
- shell: Tauri is a good fit because memory footprint is lower than Electron
- backend: Rust service implementing Bluetooth discovery, connection, notifications, command transport, OTA helpers

Suggested layering:

1. `desktop-bluetooth-adapter`
   Exposes scan, connect, disconnect, subscribe, write.
2. `earbuds-protocol-core`
   Reimplements `buildCommand`, `parseResponse`, protocol dispatch, protocol-specific facades.
3. `device-service`
   Reimplements query/set operations listed above.
4. `frontend bridge`
   Thin IPC surface mirroring the small subset the UI actually needs.

## Immediate next step

Before building UI, extract exact call signatures for:

- `sendCommand`
- `queryDeviceInfo`
- `queryFirmwareVersion`
- `setAncMode`
- `setEqMode`
- `setDualLink`
- `setHighQuality`
- `setWearDetection`
- `factoryReset`

These will define the first desktop backend API.
