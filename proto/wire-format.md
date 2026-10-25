# Garden node wire format

Version 1, draft.

Nodes send raw, untyped numbers keyed by slot. A manifest, sent whenever the probe set changes, tells the hub what each slot is. The hub owns units, meaning and location, and keeps every raw payload for provenance. Readings carry their age rather than a timestamp, so nodes need no wall clock. One command encoding serves LoRaWAN downlinks, the ST25DV NFC tag and the bench shell. A typical sample set is 12 to 50 bytes, which fits one DR1 frame.

## Transport

LoRaWAN 1.0.4, Class A, OTAA, US915 sub-band 2 (channels 8 to 15 plus 65), with ADR on. LoRaWAN's session keys encrypt and authenticate payloads, and FCnt gives replay protection, so the application layer adds no crypto of its own.

Maximum application payload (N) per uplink data rate, from the US915 regional parameters:

| Data rate | Modulation | Max payload (bytes) |
| --- | --- | --- |
| DR0 | SF10, 125 kHz | 11 |
| DR1 | SF9, 125 kHz | 53 |
| DR2 | SF8, 125 kHz | 125 |
| DR3 | SF7, 125 kHz | 242 |
| DR4 | SF8, 500 kHz | 242 |

With nodes a short distance from the gateway, ADR should settle on DR3. The format still has to work at DR0, where a node joins and falls back. Any frame can be split at sample-set or record boundaries, and a single sample set can be split across frames (see [Data uplink](#data-uplink-fport-1)). Downlinks are limited to 53 bytes so they fit RX2 (DR8).

| FPort | Direction | Content | Confirmed |
| --- | --- | --- | --- |
| 1 | Up | Data: sample sets | No |
| 2 | Up | Manifest (protobuf) | Yes |
| 3 | Up | Status (protobuf) | No |
| 10 | Down | Commands (protobuf) | n/a |
| 10 | Up | Command responses (protobuf) | No |
| 200 to 202 | Both | Reserved: LoRaWAN remote multicast, fragmentation and clock sync (FUOTA, profile packs) | Per spec |
| 224 | Both | Reserved: LoRaWAN certification test | n/a |

## Conventions

- **Byte order:** little-endian for every fixed-width integer.
- **varint:** unsigned LEB128, with 7 bits per byte, low group first, and the high bit set when more bytes follow. A u32 takes at most 5 bytes.
- **zigzag:** a signed value `n` maps to `(n << 1) ^ (n >> 31)`, so small negatives stay short.
- **Frame header (every port, both directions):** one byte, `version:4 | flags:4`. This document defines version 1. The hub decodes versions N and N-1, so firmware and hub can be updated in either order.
- **Two encodings.** The data frame (FPort 1) is hand-specified below, because it's the size-critical frame and must work at DR0. Everything else (manifest, status, commands and responses) is protobuf, defined in [garden.proto](garden.proto) and kept minimal: field numbers below 16 so every tag is one byte, packed repeated scalars, and nanopb size limits in [garden.options](garden.options). Unknown fields are skipped, so adding a field never needs a version bump.

### Value types

The manifest gives each slot a value type. The node never converts units: it sends what the bus delivered, and the hub applies scale and units per sensor profile.

| Code | Type | Encoding | Typical use |
| --- | --- | --- | --- |
| 0x01 | `u8` | 1 byte | Status codes |
| 0x02 | `u16` | 2 bytes LE | Battery mV, Modbus register |
| 0x03 | `i16` | 2 bytes LE | DS18B20 raw (1/16 °C), Modbus register |
| 0x04 | `u32` | 4 bytes LE | Modbus register pair, counters |
| 0x05 | `i32` | 4 bytes LE | Modbus register pair |
| 0x06 | `dec` | varint of `zigzag(mantissa) << 3 \| places` | SDI-12 values |

`dec` keeps an SDI-12 decimal exactly as the sensor sent it. An SDI-12 value is a sign and up to 7 digits with an optional decimal point, so `+23.45` becomes mantissa 2345 with 2 places. The low 3 bits carry the number of decimal places (0 to 7), and the zigzag-encoded mantissa sits above them. Most readings take 2 or 3 bytes, with no float rounding and no per-model knowledge on the node. The varint and zigzag rules are the same as protobuf's.

## Data uplink (FPort 1)

A data frame is a header, the manifest generation, and one or more sample sets back to back. The decoder reads sets until the frame ends.

| Field | Size | Meaning |
| --- | --- | --- |
| header | 1 | `version:4 \| flags:4`. Flag bit 0 means backlog pending (more sets queued). Bits 1 to 3 are reserved and must be 0. |
| mgen | 1 | Manifest generation the slots refer to; wraps at 256 |
| *per sample set:* | | |
| age | varint | Seconds between sampling and this transmission |
| present | ceil(S/8) | Slot bitmap. Bit i (LSB first) is set if slot i has a value. S is the slot count from the manifest. |
| values | variable | One value per set bit, in slot order, each encoded as its slot's type |

**Time.** The hub timestamps each set as the gateway receive time minus `age`. Gateway time comes from GNSS, and the node only needs its monotonic RTC counter, so nodes need no wall clock and no clock-sync traffic. The clock-sync package (FPort 202) stays only because FUOTA needs it.

**Missing values.** A clear bit means the slot has no value this time, whether the probe timed out, returned garbage, or was powered off. The reason goes in the status uplink's per-bus error counters, not in every data frame.

**Splitting.** When a set doesn't fit the current data rate, the node sends it as several sets for the same sample instant, each with a subset of bits set. Each frame recomputes `age` at its own send time. The hub merges sets with the same device, mgen and computed sample time, allowing 2 s for rounding. No continuation flag is needed.

**Batching.** Normally one set goes out per wake. After missed transmissions (duty cycle, a failed join), queued sets go out oldest first, packed into as few frames as the data rate allows. The backlog flag stays set until the queue drains. Queued sets live in the flash log (see [Flash log](#flash-log)), so they survive a reboot.

### Worked example (vector 001)

node-1 has four slots: battery mV (`u16`), DS18B20 air temperature (`i16`, 1/16 °C), SDI-12 soil moisture (`dec`) and SDI-12 soil temperature (`dec`). The readings are 3612 mV, -1.25 °C, 23.4 % VWC and 4.81 °C, sampled just now, at manifest generation 7.

```
10        header: version 1, no flags
07        mgen 7
00        age 0 s
0F        present: slots 0-3
1C 0E     slot 0 u16 = 3612 (battery mV)
EC FF     slot 1 i16 = -20 (= -1.25 °C)
A1 1D     slot 2 dec = 234, 1 place (23.4)
92 3C     slot 3 dec = 481, 2 places (4.81)
```

That's 12 bytes, which fits DR1 and above. At DR0 (11 bytes) it would go as two frames, for example slots 0 and 1, then slots 2 and 3. A larger node with 15 slots (four SDI-12 probes with three values each, two DS18B20s and battery) needs about 47 bytes per set, still one DR1 frame.

## Manifest uplink (FPort 2)

The manifest lists the node's probes in order, and their values in order define the slots. It goes out confirmed after join, after any change to the probe table (which increments mgen), daily, and on request.

Each frame is `header`, `mgen:u8`, `seq:u8` (frame index within this manifest, from 0), then a `Manifest` message. Header flag bit 0 marks the last frame. A frame never splits a `Probe` entry. Protobuf merges repeated fields on concatenation, so the hub concatenates the message bodies in `seq` order and decodes them as one `Manifest`. Entries with the same `idx` are merged too, which lets a long SDI-12 `ident` travel in its own entry and frame.

| Bus | `addr` |
| --- | --- |
| `BUS_INTERNAL` | 1 byte sensor id: 1 = battery mV, 2 = MCU temperature, 3 = supply voltage (powered nodes) |
| `BUS_SDI12` | 1 byte, the ASCII address character |
| `BUS_ONEWIRE` | 8 bytes, the ROM ID including family code and CRC |
| `BUS_MODBUS` | 1 byte, the slave address |

`port` is the physical connector number printed on the board's silkscreen, so the manifest also says which socket each probe is plugged into. `profile_id` and `pack_crc` name the sensor profile the probe is read with.

The last frame carries `ManifestEnd` with the probe, slot and frame counts. The hub treats the manifest as complete when it holds frames 0 to the last one and the counts match. LoRaWAN's MIC already protects each frame, so the manifest carries no CRC of its own. The hub keys stored manifests by device, mgen and a hash it computes over the decoded content. Until a manifest is complete, the hub stores data for that mgen raw and decodes it afterwards.

For node-1's three probes (battery, one DS18B20, one two-value SDI-12 soil probe), the `Manifest` body is 52 bytes and the frame 55 (vector 002). That's one frame at DR2 and above, or two at DR1. Each SDI-12 identity adds up to 39 bytes as its own `Probe` entry.

## Status uplink (FPort 3)

A status uplink goes out after join, after a reset, daily, and on request. The frame is `header` followed by a `Status` message: firmware version, git hash, MCUboot slot and profile-pack CRC; reset cause and count; uptime; per-bus error counters since the last status; interval, mgen and queue depth; and battery voltage.

`Power.min_mv`, the lowest voltage under transmit load, predicts the end of life of a Li-SOCl2 cell, because the resting voltage stays flat until near the end.

## Commands and responses (FPort 10)

A downlink is `header` followed by a `Commands` message holding one or more `Command`s, at most 53 bytes in total. Each command has a `seq` and exactly one operation. The node answers on uplink FPort 10 with `header` and a `Responses` message, sent as soon as duty cycle allows. That uplink also opens the next receive window, so follow-up commands arrive quickly. The node remembers the last 8 `seq` values and replays the stored response rather than re-executing, which makes network-server retries safe.

| Operation | Fields | Effect |
| --- | --- | --- |
| `set_interval` | `minutes` (5 to 1440) | Sample and send interval |
| `rescan` | `bus_mask` | Rescan buses; sends a new manifest if anything changed |
| `send_manifest` | none | Resend the current manifest |
| `send_status` | none | Send status now |
| `sdi12_set_addr` | `port`, `from`, `to` | Issue `aAb!`, then rescan that port |
| `modbus_define` | `port`, `addr`, `profile_id` | Add a Modbus probe read with that profile. The response returns `probe_idx`. |
| `probe_remove` | `idx` | Drop a probe from the table |
| `set_warmup` | `port`, `ms` | Delay between probe power-on and first command; overrides the profile |
| `backfill` | `from_age_s`, `to_age_s` | Resend sets from the flash log in that age range |
| `reboot` | `magic` = 0xB007 | Reboot after the response is sent |
| `debug` | `op`, `payload` | Development firmware only, for example SDI-12 pass-through. Production builds answer `RESULT_NOT_PERMITTED`. |

Setting the interval to 30 minutes is a 9-byte downlink: `10 0A 06 08 01 12 02 08 1E` (vector 003). The OK response is 5 bytes: `10 0A 02 08 01` (vector 004).

**One codec, three transports.** The same command and response bytes travel over a LoRaWAN downlink, over the ST25DV NFC tag, and through the bench shell (`garden cmd 0A 06 08 01 12 02 08 1E`). Over NFC, a phone writes an NDEF record holding the command bytes, the node picks it up on the tag's RF-write interrupt, and it writes the response back as a record. Android Chrome can do this through Web NFC, and iOS stays read-only until there's a native app. The tag's static area always holds two read-only records: the node page URL with the DevEUI, and the latest status.

NFC skips LoRaWAN's crypto, so it relies on physical access. That's an accepted boundary: the worst case is lost or false data, and the hub's approval step for new probes limits what false data can do.

## Flash log

The node keeps a ring buffer of encoded sample sets in a dedicated flash partition. Its size is a Kconfig option, defaulting to 8 KB (4 pages, about 160 sets, roughly 3 days at a 30-minute interval). It's there to cover connectivity lapses, it doubles as the send queue, and `backfill` reads from it. At 48 appends a day, page wear stays far inside the STM32WL's rated erase cycles.

## Sensor profile packs

A profile describes how to read one sensor model. Profiles are data, not code: the node runs a small table-driven interpreter. Loadable code (Zephyr LLEXT) is deliberately avoided to keep the attack surface flat.

- **Source:** `proto/profiles/*.toml`, one file per model, each with a 16-bit profile id. A file holds the bus, serial settings, warm-up time, and the read recipe: the SDI-12 command (`aM!`, `aC!`, `aM1!` and so on) and expected value count, or the Modbus `fn`, `reg`, `count` and `vtype` for each value. Hub-only fields go in the same file: units, scale and offset, and plausible ranges.
- **Build outputs:** a binary pack for nodes (packed records with a CRC header), and decode metadata compiled into the hub. Nothing is defined twice.
- **On the node:** a dedicated flash partition. Each firmware release links in a default pack, and the pack can be replaced on its own. The status message reports `pack_crc`.
- **Matching:** SDI-12 probes match a profile automatically from the vendor and model in their `aI!` response. Modbus devices can't identify themselves, so the admin assigns a profile and the hub sends `modbus_define`.
- **Delivery:** with the firmware image, over LoRaWAN using the fragmentation package (FPort 201, the same transport as FUOTA), over NFC, or on the bench through mcumgr.

## Hub handling

- **Raw first.** The hub stores every uplink's raw bytes with FPort, FCnt, gateway receive time, RSSI and SNR before decoding. Each decoded reading points back to its raw row, so every number on a chart traces to the exact bytes the node sent. That's the provenance trail, and it means a decoder bug can be fixed and re-run over history.
- **Unknown mgen.** Data for a manifest the hub hasn't completed stays raw. The hub sends `send_manifest` and decodes the backlog once the manifest arrives.
- **New probes need approval.** A manifest with a probe the hub hasn't seen before (a new ROM ID or a new SDI-12 identity) puts it on an "unassigned" list in the admin UI, where someone sets its location. Its data is stored from the start and appears once it has a location.
- **Gaps.** The hub counts FCnt gaps per node and shows them on the hub health page. The admin can issue `backfill` for a gap.
- **Probe identity across moves.** A probe is identified by its ROM ID, or by the SDI-12 vendor, model and serial from `aI!`, not by its slot or port. A probe moved to another port or node keeps its history.
- **Commands.** The admin UI queues commands through ChirpStack's gRPC API, tracks each `seq`, and shows each command as pending, done or failed.

## Implementation and test vectors

This directory is the contract between the node firmware and the hub. The hub repository pulls in this repository as a git submodule pinned to a tag and reads `proto/` from it, so a protocol change shows up as an explicit submodule bump on the hub side.

- **Node:** C. The data frame codec is hand-written with no heap: sets are encoded straight into the LoRaWAN TX buffer against a remaining-bytes budget for the current data rate. The protobuf messages use nanopb, generated from `garden.proto` and `garden.options` with static buffers.
- **Hub:** a Rust crate, `garden-proto`, with the data frame codec written `no_std` and the protobuf messages generated by prost.
- **Shared vectors:** `vectors/*.json`, each holding a manifest, a hex payload and the expected decoded output. Zephyr `ztest` checks that the encoder produces the exact bytes, and `cargo test` checks the decoder against the same files.
- **Fuzzing:** `cargo fuzz` on the hub decoders, since they parse radio input, and a libFuzzer harness for the node's command handling built for `native_sim`.

## Decisions

- Flash log in v1, with Kconfig-sized storage defaulting to 8 KB.
- LoRaWAN 1.0.4. Zephyr's LoRaMac-node backend is deprecated upstream in favour of LoRa Basics Modem. The firmware uses only Zephyr's `lorawan` API, so a backend swap shouldn't reach it.
- Sensor profile packs replace inline Modbus register definitions.
- NFC writes need no extra gate. Physical access is the accepted boundary.
- mgen stays 8-bit. A 16-bit generation would cost a byte per data frame, which matters at DR0, and the hub already tells generations apart by a hash of the manifest content.
- Minimal protobuf (nanopb on the node, prost on the hub) for manifest, status, commands and responses; the data frame stays hand-specified.

## Open questions

- Pack format details: a fixed record layout or a small TLV, and the maximum pack size (4 KB?).
- NFC write path: NDEF command records through Web NFC, Android only for now; a native app later.
