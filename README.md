# ugreenapp

Rust CLI for a Linux/Windows port of UGREEN Connect.

Current status:
- real `UG1` control works on `UGREEN HiTune Max5c` over Linux RFCOMM
- verified commands: `query-info`, `query-firmware`, `set-anc`, `set-eq`, `set-prompt-language`, `set-prompt-volume`
- the main code in this repository was created with AI assistance and then tested/refined manually

## Build

```bash
cargo build --release
```

Binary:

```bash
./target/release/ugreen_core
```

## CLI

Useful commands:
- `scan`
- `connect max5c`
- `info`
- `fw`
- `anc off|transparency|light|medium|deep|adaptive`
- `eq bass|pop|classic`
- `lang english|chinese`
- `vol 1..15`

Use `--json` with any command for full state output, for example:

```bash
./target/release/ugreen_core --json query-info
```

Inside the REPL:

```text
scan --json
info --json
```

`factory-reset` is intentionally gated behind `--yes-really-reset-device`.

## Docs

- `docs/earbuds-desktop-bridge.md`
- `docs/desktop-app-shape.md`
- `docs/protocol-layer.md`
