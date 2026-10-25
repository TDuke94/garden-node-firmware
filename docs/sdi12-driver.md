# SDI-12 bus driver

Zephyr has no in-tree SDI-12 support, so the node firmware carries its own
bus driver. This note covers the shape of it. The SDI-12 v1.4 specification
is the reference for anything not stated here.

## Scope

- Data recorder (master) side only. The node never acts as a sensor.
- One driver instance per SDI-12 port. A port is one bus with up to 62
  addressed sensors on it.
- Commands used: `a!`, `?!`, `aI!`, `aAb!`, `aM!`/`aMC!`, `aC!`/`aCC!`,
  `aD0!`..`aD9!`. Continuous (`aR0!`) and high-volume (`aHA!`, `aHB!`)
  commands are out of scope for v1.

## Electrical

SDI-12 is a single 5 V data line with inverted logic: marking (1) is below
1 V, spacing (0) is 3.5 V to 5 V. 12 V sensor power runs on a separate
conductor.

The board puts a 5 V buffer with a direction pin between the MCU UART and
the line, plus a switched rail for sensor power. The driver needs:

| Signal    | Use                                                   |
|-----------|-------------------------------------------------------|
| UART TX   | Inverted, 1200 baud, 7 data bits, even parity, 1 stop |
| UART RX   | Same framing                                          |
| DIR GPIO  | High while the recorder drives the line               |
| Power     | Regulator or GPIO for the sensor supply rail          |

STM32 USARTs and the LPUART support TX/RX inversion and 7E1 in hardware, so
no bit-banging is needed.

## Devicetree

The bus is a child of its UART, the same pattern as Zephyr's Modbus serial
binding.

```dts
&usart1 {
    current-speed = <1200>;
    parity = "even";
    data-bits = <7>;
    tx-invert;
    rx-invert;

    sdi12_0: sdi12 {
        compatible = "garden,sdi12";
        dir-gpios = <&gpioa 9 GPIO_ACTIVE_HIGH>;
        vin-supply = <&probe_12v>;
        warmup-ms = <500>;
    };
};
```

`warmup-ms` is the default time from power-on to the first break. The
`SetWarmup` command in the wire format overrides it per port at runtime.

## API

```c
int sdi12_power(const struct device *bus, bool on);

/* Send one command, wake the bus first if it has gone to sleep, and copy
 * the response (without CR/LF) into resp. Handles retries and echo. */
int sdi12_transact(const struct device *bus, const char *cmd,
                   char *resp, size_t len);

/* Helpers built on sdi12_transact. */
int sdi12_identify(const struct device *bus, char addr, char *ident, size_t len);
int sdi12_scan(const struct device *bus, uint64_t *present);
int sdi12_measure(const struct device *bus, char addr, bool concurrent,
                  bool crc, struct sdi12_value *vals, size_t *nvals);
```

`struct sdi12_value` holds a mantissa and decimal places, the same shape as
the `dec` type in [`proto/wire-format.md`](../proto/wire-format.md), so
readings go from the bus to the uplink without floating point.

All calls on one bus are serialised by a mutex. The driver keeps no probe
table; that lives in the application and is filled from `sdi12_scan` and
`sdi12_identify`.

## Timing

| Step                         | Value                                     |
|------------------------------|-------------------------------------------|
| Break (spacing)              | at least 12 ms                            |
| Marking after break          | at least 8.33 ms before the first command |
| Sensor response start        | within 15 ms of the command's last stop bit |
| Recorder releases the line   | within 7.5 ms of the last stop bit        |
| Gap between response chars   | at most 1.66 ms                           |
| Retry if no response         | after 16.67 ms; at least 3 tries within 100 ms of a break |
| Sensor goes back to sleep    | after 100 ms of marking                   |

The break is made by dropping the UART to 600 baud and sending `0x00`. In
7E1 that is 9 spacing bit times, 15 ms, with no pinmux switching. A GPIO
fallback (pinctrl sleep state, drive the pin) is kept behind a Kconfig
option in case a sensor rejects the short marking stop bit in the middle of
the break.

The driver skips the break when the previous transaction ended less than
about 80 ms ago, because the sensors are still awake.

Bus echo: the buffer feeds TX back to RX while DIR is high. The driver
disables RX while transmitting and flushes the FIFO before turning the bus
around, instead of matching echoed bytes.

## Measurement flow

For each scheduled sample set:

1. Power the rail, wait `warmup-ms`.
2. Start a concurrent measurement (`aCC!`) on every sensor on the port. Each
   reply gives a wait time and value count.
3. Sleep until the longest wait has passed. Service requests are ignored in
   concurrent mode, so the CPU can sit in Stop2.
4. Read each sensor with `aD0!`, `aD1!`... until it has returned all its
   values, checking the CRC.
5. Power the rail off.

Concurrent mode keeps rail-on time close to the slowest sensor's
measurement time, instead of the sum of all of them. Sensors that only
support `aM!` (v1.2 and older, seen in `aI!`) fall back to sequential
measurement.

CRC is CRC-16 with polynomial 0xA001 (reflected), initial value 0, sent as
three ASCII characters. A CRC failure counts as a bad frame in the status
`BusErrors` counters and the read is retried once.

## Scan and identity

`sdi12_scan` sends `a!` for each address `0-9`, `a-z`, `A-Z` with a single
try per address. Most ports have one to four sensors, so a full scan is
about 62 short timeouts, well under 2 s. The application then calls
`aI!` on every address that answered and compares the result with the probe
table, as described in the manifest section of the wire format.

The `Sdi12SetAddr` command maps to `aAb!`, then re-reads `aI!` to confirm.

## Power management

The UART is only enabled between power-on and power-off of the rail. Outside
a sample set the driver suspends the UART through Zephyr device PM and holds
DIR low, so the buffer draws only its quiescent current.

## Testing

- **Unit tests on native_sim.** An emulated SDI-12 sensor
  (Zephyr `emul` style) behind a fake UART exercises the parser, CRC,
  retries and timing logic under twister with no hardware.
- **Bench emulator.** A small Zephyr app under `tests/sdi12-sensor-sim/`
  runs on a second board as one or more fake sensors, with knobs for slow
  responses, bad CRCs and dropped characters. This covers most of bring-up.
- **Logic analyzer.** Saleae Logic 2's built-in Async Serial analyzer
  decodes SDI-12 bytes when set to 1200 baud, 7 bits, even parity and
  inverted. Break and marking times are measured with the timing markers.
  Probe on the MCU side of the buffer (3.3 V) or on the 5 V line.
- **Real probe.** At least one commercial SDI-12 sensor on the bench before
  field use, to catch wake-up and response-time quirks.

## Layout in the tree

```
drivers/sdi12/            driver, Kconfig, CMakeLists
include/garden/sdi12.h    public API
dts/bindings/sdi12/       garden,sdi12.yaml
tests/drivers/sdi12/      native_sim tests and emulator
tests/sdi12-sensor-sim/   bench emulator app
```

## Open questions

- Whether a port needs the GPIO break fallback in practice. Decide after
  testing with a real sensor.
- Whether to support `aR0!` continuous sensors, which some soil probes use
  in place of `aM!`.
