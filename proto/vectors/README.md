# Test vectors

Each file pairs a payload with its expected decoding. The node encoder must produce `payload_hex` byte for byte from the decoded content, and the hub decoder must produce `decoded` from `payload_hex`.

- `payload_hex`: the LoRaWAN FRMPayload after decryption, as space-separated hex bytes.
- `manifest`: for data frames, the id of the vector that holds the manifest the slots refer to.
- 1-Wire ROM IDs carry a valid Dallas/Maxim CRC-8 in the last byte. Manifest END records use CRC-16/CCITT-FALSE (poly 0x1021, init 0xFFFF, no reflection, no final XOR), stored little-endian.

Vector 001 is the worked example in [wire-format.md](../wire-format.md).
