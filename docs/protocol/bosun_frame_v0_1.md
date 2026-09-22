# Bosun Frame Protocol Specification

| Property | Value |
| --- | --- |
| Specification | BOSUN-FRAME-001 |
| Revision | 0.1 |
| Status | LOCKED |
| Date | September 21, 2026 |
| Suite / brand | Bosun |
| Repository | `bosun_embedded` |
| Host executable | `bosun` |
| R0 tool | `bosun frame` |
| Future shared protocol crate | `crates/protocol` |
| Future published crate | `bosun-embedded` |
| Governing pathway | Embedded Rust Learning Program v3 |
| Current development gate | R0 — Rust Readiness and Toolchain |
| Wire protocol version | `0x01` |

## Contents

- [1. Purpose](#1-purpose)
- [2. Wire format](#2-wire-format)
- [3. CRC-16 specification](#3-crc-16-specification)
- [4. Message types](#4-message-types)
- [5. Codec architecture](#5-codec-architecture)
- [6. Strict-decoding validation order](#6-strict-decoding-validation-order)
- [7. Diagnostic behavior](#7-diagnostic-behavior)
- [8. Error model](#8-error-model)
- [9. CLI contract](#9-cli-contract)
- [10. Rust project structure](#10-rust-project-structure)
- [11. Mechanical code-quality checks](#11-mechanical-code-quality-checks)
- [12. Golden vectors](#12-golden-vectors)
- [13. Required test matrix](#13-required-test-matrix)
- [14. R0 completion criteria](#14-r0-completion-criteria)
- [15. Deferred work](#15-deferred-work)
- [16. Design freeze](#16-design-freeze)

## 1. Purpose

Bosun Frame defines a small binary message format and a Rust implementation for encoding, decoding, and validating complete frames.

The initial implementation is a host-only utility exposed through the `bosun frame` subcommand. Its protocol logic will later be extracted into a shared, `no_std`-compatible crate used by STM32 firmware, host tools, hardware-in-the-loop tests, and the Raspberry Pi gateway.

The project is intended to develop practical competency in binary representation, endianness, memory safety, borrowing, error handling, serialization, protocol validation, and automated testing.

### 1.1 Required R0 capabilities

The tool shall support three operations:

* Encode: Convert structured message values into a complete binary frame.
* Decode: Validate a complete frame and convert its contents into structured message values.
* Doctor: Inspect a frame and report available fields, integrity results, and validation failures.

The protocol implementation shall be independently usable without invoking the CLI.

### 1.2 Scope boundaries

R0 operates on exactly one complete frame per input.

It does not implement serial-port communication, incremental stream decoding, synchronization, byte stuffing, COBS, CAN transport mapping, fragmentation, retransmission, encryption, or authentication.

These functions are outside the R0 deliverable.

The protocol's sequence number provides an identifier but does not itself implement acknowledgements, replay protection, or delivery guarantees.

### 1.3 Normative language

The terms MUST, MUST NOT, SHOULD, and MAY express requirements and optional behavior.

## 2. Wire format

A Bosun frame consists of a fixed eight-byte header, a variable-length payload, and a two-byte CRC trailer.

```text
┌────────┬─────┬──────┬──────┬──────┬─────────────┬───────┐
│ MAGIC  │ VER │ TYPE │ SEQ  │ LEN  │   PAYLOAD   │ CRC16 │
├────────┼─────┼──────┼──────┼──────┼─────────────┼───────┤
│ 2 B    │ 1 B │ 1 B  │ 2 B  │ 2 B  │ 0–256 B     │ 2 B   │
└────────┴─────┴──────┴──────┴──────┴─────────────┴───────┘
```

### 2.1 Field definitions

| Offset | Field | Type | Description |
| --- | --- | --- | --- |
| 0–1 | Magic | `[u8; 2]` | ASCII `BN`, represented by bytes `42 4E` |
| 2 | Version | `u8` | Wire protocol version |
| 3 | Message type | `u8` | Identifies the message schema |
| 4–5 | Sequence | `u16 LE` | Application sequence identifier |
| 6–7 | Length | `u16 LE` | Payload length, in bytes |
| 8… | Payload | Byte sequence | Exactly `len` bytes |
| Final 2 | CRC | `u16 LE` | CRC-16 over header and payload |

The magic is defined as two raw bytes, not as a `u16`.

All multibyte integer fields MUST be serialized in little-endian order. Implementations MUST NOT depend on the host CPU's native byte order.

### 2.2 Protocol constants

| Constant | Value | Meaning |
| --- | --- | --- |
| `MAGIC` | `[0x42, 0x4E]` | ASCII `BN` |
| `VERSION` | `0x01` | Supported wire version |
| `HEADER_LEN` | 8 | Fixed header size |
| `CRC_LEN` | 2 | CRC trailer size |
| `MIN_FRAME` | 10 | Minimum valid frame size |
| `MAX_PAYLOAD` | 256 | Maximum supported payload |
| `MAX_FRAME` | 266 | Maximum unescaped frame size |

The total frame size is:

```text
frame_size(len) = 10 + len
```

The `u16` length field can represent values larger than 256, but the application protocol MUST reject payloads exceeding `MAX_PAYLOAD`.

An empty-payload frame, such as Ping, is exactly 10 bytes.

### 2.3 Sequence number

The sequence number is an unsigned 16-bit integer with a valid range of `0..=65535`.

It is serialized in little-endian order.

For example:

```text
Sequence value: 0x1234
Wire bytes:     34 12
```

The codec MUST preserve the sequence number without assigning application semantics to it.

At the application layer, a Pong SHOULD echo the sequence number of its corresponding Ping. R0 does not enforce that convention.

Sequence rollover, outstanding-request tracking, retries, and duplicate suppression are outside the R0 codec.

### 2.4 Versioning

Specification revision v0.1 uses wire version `0x01`.

The specification revision and the wire version are separate concepts. A later compatible specification update does not automatically require a different wire-version byte.

An unsupported wire version MUST be rejected before interpreting the remaining fields using the v1 layout.

The protocol does not guarantee that future versions will retain the same header size, CRC algorithm, or payload representation.

## 3. CRC-16 specification

Bosun Frame v0.1 uses CRC-16/IBM-3740, formerly commonly identified as CRC-16/CCITT-FALSE.

The algorithm MUST be defined by its full parameters rather than the ambiguous designation `CRC-CCITT`.

### 3.1 Algorithm parameters

| Parameter | Value |
| --- | --- |
| Width | 16 bits |
| Polynomial | `0x1021` |
| Initial value | `0xFFFF` |
| Reflect input | False |
| Reflect output | False |
| Final XOR | `0x0000` |
| Reference input | ASCII `123456789` |
| Reference CRC | `0x29B1` |

The standard reference check MUST pass before generating protocol fixtures.

### 3.2 CRC coverage

The CRC covers the complete header and payload, beginning with the first magic byte and ending with the final payload byte.

```text
┌───────────────────────────────────────────────────┬─────────┐
│                 CRC-COVERED DATA                   │ TRAILER │
├────────┬─────────┬──────┬───────┬──────┬───────────┼─────────┤
│ MAGIC  │ VERSION │ TYPE │ SEQ   │ LEN  │ PAYLOAD   │ CRC16   │
└────────┴─────────┴──────┴───────┴──────┴───────────┴─────────┘
```

The CRC MUST NOT include its own trailer bytes.

For a valid frame with payload length `N`, the CRC input is the first `8 + N` bytes.

The computed CRC is appended as a little-endian `u16`.

For illustration only, the value `0x29B1` would be serialized as:

```text
B1 29
```

This illustrates byte order; it is not the computed CRC of a particular Bosun frame.

### 3.3 Verification

The decoder MUST calculate the CRC from the received header and payload and compare it against the stored trailer.

A mismatch MUST result in `CrcMismatch`.

The CRC provides accidental-corruption detection. It is not an authentication mechanism and does not protect against deliberate message modification.

A software CRC-16 implementation is the reference behavior. Use of an MCU hardware CRC peripheral is not required.

## 4. Message types

Bosun Frame v0.1 defines exactly four message types.

| Type ID | Message | Payload length | Purpose |
| --- | --- | --- | --- |
| `0x01` | `Ping` | 0 | Link-presence request |
| `0x02` | `Pong` | 0 | Link-presence response |
| `0x20` | `Data` | 1–256 | Tagged application data |
| `0x7F` | `Error` | 1 | Protocol/application error message |

All other message-type values are unsupported in v0.1.

### 4.1 Ping

```text
Type:    0x01
Payload: empty
Length:  0
```

A Ping message MUST have an empty payload.

Any nonempty Ping payload MUST be rejected as `InvalidPayload`.

### 4.2 Pong

```text
Type:    0x02
Payload: empty
Length:  0
```

A Pong message MUST have an empty payload.

Any nonempty Pong payload MUST be rejected as `InvalidPayload`.

A Pong SHOULD echo the sequence number of the Ping it answers. This convention is not enforced by the R0 codec.

### 4.3 Data

```text
Type: 0x20

Payload:
┌─────────┬──────────────────────┐
│ TAG     │ DATA                 │
├─────────┼──────────────────────┤
│ u8      │ 0–255 bytes          │
└─────────┴──────────────────────┘
```

The first payload byte is the tag.

The remaining bytes are uninterpreted application data.

The payload is defined as:

```text
payload = [tag: u8] || data

len = 1 + data.len()

data.len() = 0..=255
```

A Data frame with `len = 0` MUST be rejected because the required tag is missing.

A Data frame with `len = 1` is valid and contains only its tag.

A Data frame with `len = 256` is valid and contains one tag byte followed by 255 data bytes.

The tag accepts any `u8` value. The framing layer does not assign additional meaning to individual tag values.

### 4.4 Error

```text
Type:    0x7F
Payload: code: u8
Length:  1
```

An Error message MUST contain exactly one error-code byte.

A payload with length zero or length greater than one MUST be rejected as `InvalidPayload`.

All `u8` error-code values are accepted at the framing layer. Application-specific meanings may be defined in later gates.

The protocol message `Error` is distinct from the Rust error type `FrameError`.

A successfully decoded Error message is a valid frame, whereas `FrameError` describes a failure to encode or decode a frame.

### 4.5 Type reservations

| Range | Allocation |
| --- | --- |
| `0x00` | Reserved |
| `0x01–0x0F` | Link/control messages |
| `0x10–0x1F` | Unassigned |
| `0x20–0x2F` | Application messages |
| `0x30–0x7E` | Unassigned |
| `0x7F` | Error message |
| `0x80–0xFF` | Reserved for future definitions |

Reserved and unassigned values MUST produce `UnknownMessageType` in v0.1.

No additional messages are required for R0.

## 5. Codec architecture

The protocol codec and CLI have separate responsibilities.

```text
                 bosun CLI
                     │
          ┌──────────┴──────────┐
          │                     │
      File / stdin         Terminal output
          │                     ▲
          ▼                     │
      CLI routing               │
          │                     │
          ▼                     │
    Public codec API ───────────┘
          │
          ├── Encode
          ├── Decode
          ├── Inspect
          ├── CRC
          └── Typed errors
```

The codec MUST NOT depend on `clap`, filesystem operations, terminal I/O, or other CLI-specific functionality.

### 5.1 Encoding

Encoding converts a valid structured message and sequence number into a complete binary frame.

Conceptual operation:

```text
encode(message, sequence):

    1. Validate message payload
    2. Validate payload length
    3. Append magic
    4. Append version
    5. Append message type
    6. Append sequence as u16 LE
    7. Append payload length as u16 LE
    8. Append payload
    9. Calculate CRC over header + payload
   10. Append CRC as u16 LE
   11. Return complete frame
```

The encoder MUST reject unsupported message types, invalid payload schemas, and payloads exceeding `MAX_PAYLOAD`.

R0 MAY use dynamically allocated output storage. Allocation-free encoding is not required until R2 A2.

### 5.2 Strict decoding

Strict decoding accepts a complete byte slice and returns either a validated typed frame or a `FrameError`.

Conceptual API:

```text
decode(bytes: &[u8]) -> Result<Frame, FrameError>
```

This is an API illustration, not a prescribed final Rust signature.

A strict decoder MUST NOT return a successful frame unless all applicable structural, integrity, version, type, and payload-schema checks pass.

Where practical, the decoded payload SHOULD borrow the input bytes rather than unnecessarily copying them.

### 5.3 Inspection

Inspection is the diagnostic counterpart to strict decoding.

Conceptual API:

```text
inspect(bytes: &[u8]) -> FrameInspection
```

Inspection MUST produce a report even for empty, incomplete, or malformed input.

The report MUST distinguish:

* Fields that could be read.
* Fields that could not be read.
* Structural validation results.
* Integrity validation results.
* Semantic validation results.
* Whether displayed information can be trusted.

Inspection MAY report multiple independent findings when sufficient bytes are available to establish them safely.

The findings collection MUST have bounded capacity compatible with the eventual allocation-free embedded design.

An unbounded collection of diagnostic findings is not part of the protocol contract.

### 5.4 CRC implementation

CRC calculation SHOULD be separately testable.

Its implementation MUST NOT depend on the CLI or perform I/O.

The CRC reference test MUST be independent of the frame encoder.

## 6. Strict-decoding validation order

Validation order is part of the specification.

A decoder MUST determine the wire version before interpreting subsequent fields using the v1 layout.

1. Require the magic field.

   If fewer than two bytes are available, return `TruncatedFrame`.

2. Validate magic.

   If the first two bytes are not `42 4E`, return `BadMagic`.

3. Require and validate the version.

   If fewer than three bytes are available, return `TruncatedFrame`.

   If the version differs from `0x01`, return `UnsupportedVersion`.

4. Require the complete frame header.

   If fewer than eight bytes are available, return `TruncatedFrame`.

   If fewer than `MIN_FRAME` bytes are available, return `TruncatedFrame`.

5. Read type, sequence, and declared payload length.

   Interpret all multibyte fields as little-endian. Do not yet accept the message type semantically.

6. Enforce the payload limit.

   If the declared length exceeds `MAX_PAYLOAD`, return `PayloadTooLarge`.

7. Validate total input size.

   Calculate the expected size as `10 + declared_len`.

   If the input is shorter, return `TruncatedFrame`.

   If it is longer, return `TrailingBytes`.

8. Verify CRC.

   Calculate the CRC over the header and payload. Compare the result with the received CRC trailer.

   If they differ, return `CrcMismatch`.

9. Validate the message type.

   Reject unknown or reserved values with `UnknownMessageType`.

10. Validate the payload schema.

    Reject payloads inconsistent with their message type using `InvalidPayload`.

11. Return the typed frame.

    Successful decoding is permitted only after all previous checks pass.

### 6.1 Exactly one frame

R0's strict decoder accepts exactly one complete frame.

Extra bytes after the declared frame MUST produce `TrailingBytes`, even if those bytes appear to contain another valid frame.

The R2 streaming layer will handle concatenated frames separately.

### 6.2 Error precedence

Strict decoding reports the first failure according to the validation order.

For example, a supported-version frame whose declared payload length exceeds 256 MUST produce `PayloadTooLarge` before the decoder attempts to locate the claimed payload or CRC.

A frame with a bad CRC and unknown message type MUST produce `CrcMismatch` through strict decoding.

Inspection may report both findings, subject to its trust rules.

## 7. Diagnostic behavior

`bosun frame doctor` is the CLI representation of library inspection.

It SHOULD present readable header fields, available payload information, CRC status, and validation findings.

### 7.1 Trust classification

Diagnostic information is classified as:

| Classification | Meaning |
| --- | --- |
| Structural | A field exists and can be extracted |
| Integrity | CRC verification passed, failed, or could not be performed |
| Semantic | A supported-version field or payload conforms to its defined meaning |
| Untrusted | Information is readable but integrity validation failed |
| Unavailable | Insufficient information exists to determine the field |

A complete, supported-version frame with an unknown message type and a bad CRC may report both findings.

However, the message-type finding MUST be identified as untrusted.

### 7.2 Unsupported versions

If the magic is readable and the version is unsupported, inspection MUST report the magic and claimed version.

It MUST NOT interpret subsequent bytes using v1 field offsets or attempt a v1 CRC check.

### 7.3 Truncated input

Inspection MUST NOT invent absent fields.

For example, a six-byte input may contain a complete magic, version, type, and sequence, but not a complete length field.

The report may display those available fields.

It must identify the length, payload, and CRC as unavailable.

### 7.4 Incomplete or inconsistent frames

If a supported-version header is available but its declared length cannot be satisfied, the report MUST identify the appropriate structural failure.

It MUST NOT report a successful integrity check over a partial frame.

### 7.5 Example diagnostic

```text
Bosun Frame Diagnostic

Magic ............ BN
Version .......... 1
Message type ..... 0x03 (unknown, untrusted)
Sequence ......... 42
Declared length .. 2
Actual length .... 2

CRC stored ....... 0x....
CRC calculated ... 0x....
CRC .............. FAIL

Findings:
  - CrcMismatch
  - UnknownMessageType (untrusted)

Result: INVALID
```

The placeholder CRC values above are illustrative, not golden test values.

## 8. Error model

The codec MUST use typed errors.

| Error | Required meaning |
| --- | --- |
| `TruncatedFrame` | Input contains fewer bytes than required |
| `BadMagic` | Magic is not `BN` |
| `UnsupportedVersion` | Wire version is not supported |
| `PayloadTooLarge` | Payload length exceeds 256 |
| `TrailingBytes` | Input contains bytes beyond the declared complete frame |
| `CrcMismatch` | Stored and calculated CRC values differ |
| `UnknownMessageType` | Message type is undefined or reserved |
| `InvalidPayload` | Payload violates its message schema |

Encoding may require additional typed errors for invalid command values or output-buffer limitations.

`OutputBufferTooSmall` MAY be introduced when the encoding API becomes allocation-free at R2.

### 8.1 Error-handling rules

Malformed frame input MUST NOT cause an intentional panic.

The codec MUST NOT use `unwrap()`, `expect()`, `panic!()`, `todo!()`, `unimplemented!()`, or `unreachable!()`.

Input-dependent byte access MUST be bounds-checked.

Errors MUST provide enough information for meaningful diagnostics where available, such as expected and actual lengths or stored and calculated CRC values.

No implementation is required to identify the precise physical byte responsible for corruption. CRC validation detects disagreement, not the location of the fault.

## 9. CLI contract

The R0 CLI exposes exactly these subcommands:

```text
bosun frame encode
bosun frame decode
bosun frame doctor
```

No other Bosun subcommands are required in R0.

### 9.1 Encode

Example: Ping.

```bash
bosun frame encode \
    --type ping \
    --seq 3 \
    > ping.bin
```

Example: Pong.

```bash
bosun frame encode \
    --type pong \
    --seq 3 \
    > pong.bin
```

Example: Data.

```bash
bosun frame encode \
    --type data \
    --seq 3 \
    --payload 01ff \
    > data.bin
```

Example: Error.

```bash
bosun frame encode \
    --type error \
    --seq 3 \
    --payload 07 \
    > error.bin
```

#### Encode arguments

| Argument | Requirement |
| --- | --- |
| `--type` | Required |
| `--seq` | Required |
| `--payload` | Optional for empty-payload messages; required for Data and Error |
| `--hex` | Optional human-readable hexadecimal output |

The `--type` argument accepts `ping`, `pong`, `data`, and `error`, case-insensitively.

The `--seq` argument accepts decimal integers from 0 through 65535.

No implicit sequence-number default is defined.

#### Hexadecimal payload grammar

Payload input accepts hexadecimal digits `0–9`, `a–f`, and `A–F`.

Whitespace MAY separate hexadecimal digits.

The total count of hexadecimal digits MUST be even.

Odd-length input MUST be rejected.

The `0x` prefix is not part of the accepted payload grammar.

For example, both of the following represent the same payload:

```text
01ff
01 FF
```

When whitespace is supplied, the shell argument must be quoted appropriately.

The complete Data payload includes its tag. There is no separate `--tag` argument in v0.1.

Ping and Pong MUST reject nonempty supplied payloads.

#### Encode output

By default, encode writes raw binary bytes to standard output.

Diagnostics and errors MUST NOT contaminate binary stdout.

If standard output is an interactive terminal, the CLI MUST refuse raw binary output and suggest `--hex` or file redirection.

When `--hex` is supplied, the CLI writes a human-readable hexadecimal representation instead of raw binary.

### 9.2 Decode

```bash
bosun frame decode frame.bin
```

Decode reads exactly one complete binary frame, validates it, and prints a human-readable representation of its typed contents.

The path `-` denotes standard input:

```bash
bosun frame decode -
```

A valid frame produces successful output.

An invalid frame produces an appropriate typed diagnostic and a nonzero exit status.

### 9.3 Doctor

```bash
bosun frame doctor frame.bin
```

The path `-` also denotes standard input:

```bash
bosun frame doctor -
```

Doctor MUST attempt to report all safely determinable findings.

A valid frame returns exit status 0.

An invalid frame returns exit status 1.

### 9.4 Exit-status contract

| Status | Meaning |
| --- | --- |
| 0 | Successful operation / valid frame |
| 1 | Invalid frame, failed data operation, or operational failure |
| 2 | Command-line usage error |

Usage errors include missing required arguments, invalid flags, out-of-range CLI values, and malformed hexadecimal arguments.

A script can therefore distinguish an invalid binary frame from a malformed CLI invocation.

Distinct exit codes for every `FrameError` are not required.

## 10. Rust project structure

The R0 implementation remains inside the `bosun_embedded` workspace.

The package name for the host executable is `bosun`.

```text
bosun_embedded/
├── Cargo.toml
├── AGENTS.md
├── .grok/
│   └── instructions.md
│
├── tools/
│   └── bosun/
│       ├── Cargo.toml
│       ├── src/
│       │   ├── main.rs
│       │   ├── lib.rs
│       │   ├── frame.rs
│       │   ├── crc.rs
│       │   └── error.rs
│       │
│       └── tests/
│           ├── decode.rs
│           └── fixtures/
│               ├── ping.bin
│               ├── bad_crc.bin
│               └── data.bin
│
└── docs/
    └── gates/
        └── R0.md
```

### 10.1 Module responsibilities

`main.rs` contains CLI parsing, command routing, input/output handling, and diagnostic presentation.

`lib.rs` exposes the reusable codec API.

`frame.rs` contains frame representation, encoding, decoding, and inspection.

`crc.rs` contains the CRC implementation.

`error.rs` contains typed codec errors.

The precise internal Rust types and function signatures remain implementation decisions, provided they satisfy the specified behavior.

### 10.2 Library boundary

`main.rs` MUST use the public library API to access codec functionality.

The codec MUST NOT depend on CLI parsing or host I/O.

The dependency direction is:

```text
bosun CLI
    │
    ▼
bosun library API
    │
    ├── frame
    ├── crc
    └── error
```

### 10.3 R0 and R2 separation

Do not create `crates/protocol` during R0.

At R2 A2, extract the codec modules into `crates/protocol`, make the crate `no_std`-compatible, and preserve the existing `bosun frame` functionality.

R0 is not required to be `no_std` or allocation-free.

The implementation SHOULD avoid unnecessary allocation and host coupling so that extraction remains straightforward.

## 11. Mechanical code-quality checks

The codec modules MUST apply the agreed Clippy restrictions.

```rust
#![deny(
    clippy::unwrap_used,
    clippy::expect_used,
    clippy::panic,
    clippy::todo,
    clippy::unimplemented,
    clippy::unreachable,
    clippy::indexing_slicing,
    clippy::print_stdout,
    clippy::print_stderr,
    clippy::dbg_macro
)]
```

The restrictions apply to codec implementation modules, not wholesale to the CLI.

The CLI legitimately performs terminal I/O.

### 11.1 Required verification commands

```bash
cargo clippy -p bosun --all-targets -- -D warnings

cargo test -p bosun
```

CI SHOULD also flag prohibited host dependencies in the codec modules:

```text
std::fs
std::io
clap
```

These checks supplement code review and testing. They do not constitute a complete formal proof of panic freedom or memory safety.

### 11.2 Implementation rules

Frame fields MUST be serialized and parsed explicitly.

The implementation MUST NOT reinterpret input bytes as a `#[repr(C)]` structure containing multibyte integers.

Native structure padding, alignment, and endianness are not part of the wire contract.

Input-dependent slicing MUST be bounds-checked.

## 12. Golden vectors

Golden vectors provide an implementation-independent reference for the exact wire representation.

They MUST be generated from the frozen specification, not from the Bosun Rust encoder.

A short independent Python reference implementation MAY be used as test-support tooling.

Its CRC implementation MUST pass the standard reference check before generating fixtures.

### 12.1 Required fixtures

| Filename | Content | Purpose |
| --- | --- | --- |
| `ping.bin` | Valid Ping, sequence `0x1234`, empty payload | Header, endianness, zero-length payload, CRC |
| `bad_crc.bin` | Otherwise-valid Ping with one CRC trailer bit flipped | Isolated CRC failure |
| `data.bin` | Valid Data, sequence 3, payload `01 FF` | Structured and nonempty payload |

The existing pathway requires two fixtures; this specification retains those and adds `data.bin`.

### 12.2 Ping fixture

Before its CRC trailer, the valid Ping fixture contains:

```text
42 4E 01 01 34 12 00 00
```

Field interpretation:

```text
42 4E       Magic: BN
01          Version: 1
01          Type: Ping
34 12       Sequence: 0x1234
00 00       Payload length: 0
```

The complete fixture is 10 bytes, including its two CRC bytes.

### 12.3 Data fixture

Before its CRC trailer, the Data fixture contains:

```text
42 4E 01 20 03 00 02 00 01 FF
```

Field interpretation:

```text
42 4E       Magic: BN
01          Version: 1
20          Type: Data
03 00       Sequence: 3
02 00       Payload length: 2
01          Tag: 1
FF          Data: one byte
```

The complete fixture is 12 bytes, including its CRC trailer.

### 12.4 Independent generation procedure

The reference generator SHALL:

1. Validate CRC-16/IBM-3740 against `"123456789" → 0x29B1`.
2. Construct each fixture from the exact field definitions.
3. Calculate and append the appropriate CRC.
4. Produce `ping.bin` and `data.bin`.
5. Produce `bad_crc.bin` by flipping exactly one CRC trailer bit in an otherwise-valid Ping.
6. Record the resulting hexadecimal byte sequences for the specification's fixture appendix.

The actual fixture-specific CRC values are established by this independent generation procedure, not by adopting values from prior illustrative examples.

Once generated and independently verified, the fixture bytes become fixed interoperability references for v0.1.

## 13. Required test matrix

The implementation MUST include positive, negative, boundary, and diagnostic tests.

### 13.1 Positive tests

| Test | Expected result |
| --- | --- |
| CRC reference input | `0x29B1` |
| Valid Ping | Decodes successfully |
| Valid Pong | Decodes successfully |
| Valid Error, length 1 | Decodes successfully |
| Valid Data, length 1 | Tag-only payload accepted |
| Valid Data, length 2 | Tag and one data byte accepted |
| Valid Data, length 256 | Tag and 255 data bytes accepted |
| Sequence `0x1234` | Encodes as `34 12` |
| Sequence 65535 | Accepted |
| Minimum valid frame | Exactly 10 bytes |
| Maximum valid frame | Exactly 266 bytes |
| Encode/decode round trip | Original message recovered |
| Golden fixtures | Match independently established bytes |

### 13.2 Negative tests

| Test | Expected result |
| --- | --- |
| Empty input | `TruncatedFrame` |
| One-byte input | `TruncatedFrame` |
| Bad magic | `BadMagic` |
| Valid magic, missing version | `TruncatedFrame` |
| Unsupported version `0x02` | `UnsupportedVersion` |
| Incomplete header | `TruncatedFrame` |
| Declared length greater than 256 | `PayloadTooLarge` |
| Input shorter than declared frame | `TruncatedFrame` |
| Trailing extra byte | `TrailingBytes` |
| Unknown type | `UnknownMessageType` |
| Reserved type `0x00` | `UnknownMessageType` |
| Data with length 0 | `InvalidPayload` |
| Ping with length 1 | `InvalidPayload` |
| Pong with nonempty payload | `InvalidPayload` |
| Error with length 0 | `InvalidPayload` |
| Error with length 2 | `InvalidPayload` |
| Flipped CRC trailer bit | `CrcMismatch` |

Tests targeting semantic errors MUST otherwise construct structurally complete, CRC-valid frames unless the test explicitly covers multiple failures.

This prevents an earlier validation failure from masking the behavior the test is intended to exercise.

### 13.3 Diagnostic tests

`doctor` MUST correctly handle:

* A valid frame.
* Empty input.
* A six-byte prefix containing only partially available header fields.
* An unsupported wire version.
* An oversized declared payload.
* A truncated payload.
* Trailing bytes.
* A CRC mismatch.
* An unknown message type combined with a CRC mismatch.

For the combined unknown-type/CRC case, `doctor` MUST report both findings and identify the type information as untrusted.

For an unsupported version, `doctor` MUST stop interpreting the remaining bytes according to v1.

For a truncated header, `doctor` MUST NOT invent unavailable fields.

### 13.4 Additional verification

Randomized round-trip tests and decoder fuzzing are recommended extensions.

They are not prerequisites for completing R0, provided the mandatory tests and competency examination pass.

## 14. R0 completion criteria

The Bosun Frame portion of R0 is complete when:

* The `bosun frame` CLI supports encode, decode, and doctor.
* The codec is accessible through the package's public library API.
* The codec passes the prescribed positive, negative, boundary, and diagnostic tests.
* The independent golden fixtures are checked into the repository.
* The CRC reference test passes.
* The required Clippy and source checks pass.
* The developer can explain the wire format, byte order, CRC coverage, buffer checks, and error behavior.
* The Examiner's malformed-frame and live message-extension exercises are passed.

The broader R0 gate also requires the Rust competency assessment and the Nucleo-F411RE flash, run, logging, and breakpoint smoke test specified in the v3 pathway.

A passing frame codec alone does not complete the entire R0 gate.

## 15. Deferred work

### R2 A2 — Shared protocol and UART

At R2 A2:

* Extract the codec into `crates/protocol`.
* Convert the shared protocol code to `no_std`.
* Introduce fixed-capacity buffers where needed.
* Implement incremental receive and stream decoding.
* Select and document an appropriate transport-framing strategy.
* Connect `bosun` to the STM32 control node.
* Preserve the existing golden vectors and protocol semantics.

The stream decoder MUST NOT assume that the two-byte `BN` magic cannot appear inside payloads or corrupted data.

Byte stuffing, COBS, or another suitable framing mechanism must be evaluated as part of the transport design.

### R3 — Motion integration

Extend the message model for motion commands, telemetry, and fault states as required by the physical axis.

Do not add these messages in R0.

### R4 — Production hardening

Introduce protocol compatibility testing, CAN transport mapping, HIL fault injection, and endurance validation.

The CAN mapping may share application-message semantics without transmitting an identical UART envelope.

### R5 — Linux gateway

Reuse the shared protocol crate in the Raspberry Pi gateway.

The gateway must not contain a duplicate implementation of Bosun's application protocol.

## 16. Design freeze

**LOCKED · v0.1**

This specification establishes the baseline for R0 implementation.

Frozen decisions: magic, wire version, field layout, endianness, CRC algorithm and coverage, payload limit, message types, payload schemas, validation order, error categories, diagnostic behavior, CLI commands, and R0/R2 responsibility boundaries.

Changes to the wire representation or normative protocol behavior require a written specification revision.

Implementation details that do not alter the contract may evolve through normal development and review.

The required independent fixture generation and validation must be completed before implementation test results can be accepted as evidence of protocol conformance.
