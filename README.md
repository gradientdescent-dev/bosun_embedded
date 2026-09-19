# Bosun

Host tools, shared protocol, and firmware for a small motion workcell.

| Piece | Name |
|---|---|
| Suite | Bosun |
| Binary | `bosun` |
| First command | `bosun frame` |
| Repo | `bosun_embedded` |

Rust is the implementation language. The Nucleo-F411RE is the control node. The micro:bit is a trainer only.

## Status

**R0 — Rust readiness.** `bosun frame` encodes, decodes, and checks Bosun frames on the host. Firmware and the Linux gateway are later gates.

## Build

From the repo root:

```bash
cargo test -p bosun
cargo run -p bosun -- frame encode --type ping --seq 3 --payload 01ff
cargo run -p bosun -- frame decode frame.bin
cargo run -p bosun -- frame doctor frame.bin
```

Requires a recent stable Rust toolchain (`rustup`).

## Frame v0.1

Little-endian. Magic `BN`.

```
magic[2] | version | msg_type | seq:u16 | len:u16 | payload[len] | crc16
```

CRC-CCITT covers the header and payload, not the CRC field. Types: `Ping`, `Pong`, `Error`, `Data`.

## Layout

```
bosun_embedded/
├── Cargo.toml           # workspace
├── tools/bosun/         # the only host binary
├── crates/              # protocol, drivers, motion_core (from R2+)
├── firmware/            # nucleo-f411 (R2), microbit trainer (R1)
├── services/gateway/    # Pi supervisor (R5)
└── docs/gates/          # competency records
```

Run all `cargo` commands from this root. Do not add a second host binary.

## Program

Competency-gated path: R0 → R1 Discovery → R2 STM32 node → R3 motion → R4 hardening → R5 Pi gateway → R6 specialization. Details live in `docs/` and `AGENTS.md`.
