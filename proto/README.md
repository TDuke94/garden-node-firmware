# proto

The contract between garden nodes and the hub: the LoRaWAN payload format, shared test vectors and sensor profiles. The hub repository pulls in this repository as a git submodule pinned to a tag and reads `proto/` from it.

- [wire-format.md](wire-format.md): the version 1 payload specification
- [vectors/](vectors/): byte-exact test vectors for encoder and decoder tests
- [profiles/](profiles/): sensor profile sources
