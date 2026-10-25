# Test vectors

Each file pairs a payload with its expected decoding. Node and hub encoders must produce `payload_hex` byte for byte from the decoded content, and decoders must produce `decoded` from `payload_hex`.

- `payload_hex`: the LoRaWAN FRMPayload after decryption, as space-separated hex bytes, starting with the one-byte frame header.
- `manifest`: for data frames, the id of the vector holding the manifest the slots refer to.
- `message` and `body`: for protobuf frames, the message type in [garden.proto](../garden.proto) and its content in protobuf JSON style. Protobuf vectors were generated with protoc and checked by round-trip decoding. Fields are serialized in field-number order, which is what nanopb and prost both emit.
- 1-Wire ROM IDs carry a valid Dallas/Maxim CRC-8 in the last byte.

Vector 001 is the worked example in [wire-format.md](../wire-format.md).
