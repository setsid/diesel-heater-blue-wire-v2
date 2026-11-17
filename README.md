# Diesel heater blue-wire v2 protocol

Reverse-engineered notes on the pulse-width (AA/A5) protocol spoken on
the blue data wire of some Chinese diesel heaters. Worked out from
passive bench captures only. No firmware was read from the heater.

[![Licence: GPLv3](https://img.shields.io/badge/Licence-GPLv3-blue.svg)](LICENSE)

> **Aftermarket controller firmware is coming soon.** I'm bench-testing
> it before release. It will be published in this repo under the same
> licence.

## What's here

- **[blue-wire-v2-protocol.md](blue-wire-v2-protocol.md)**: the full
  write-up. Physical layer, bit coding, framing, the scrambler, a
  field-by-field map, and a worked example you can check your own decoder
  against.

## Is this your heater?

This is a different protocol from the common 25000 baud UART blue-wire
protocol that most published work describes. Tell them apart with a
scope on the blue wire:

- **Old protocol:** a continuous UART line, 25000 baud, 8 data bits, no
  parity, 1 stop bit, LSB first, about 40 us per bit. Documented by Ray
  Jones in his V9 note, [Hacking the Chinese Diesel Heater Communications
  Protocol](https://gitlab.com/mrjones.id.au/bluetoothheater/-/blob/f041cf8cad7d7d238597f63c912aa9bf9fb1e936/Documentation/V9%20-%20Hacking%20the%20Chinese%20Diesel%20Heater%20Communications%20Protocol.pdf).
- **This one (v2):** a line that idles high, each bit sent as a low pulse
  then a high pulse, messages opened by a sync low of about 3 ms, header
  bytes 0xAA from the controller and 0xA5 from the ECU, one controller
  message and one ECU reply every 201 ms.

## The protocol in one minute

- One blue wire, idle high, open-collector. The controller supplies the
  pull-up. This ECU does not, so a replacement controller must.
- 11 bytes per message. Byte 0 is a clear header (0xAA controller, 0xA5
  ECU). Bytes 1 to 9 are a scrambled 9-byte payload. Byte 10 is a
  checksum.
- The scrambler is the same both directions and carries no state between
  messages:

  ```
  p[j] = c[j] XOR 0x53 XOR crc8(c[j-1])      for j = 1..9
  crc8 = CRC-8, polynomial 0x31, init 0x00, no reflection, no final XOR
  ```

- Two checks on every message: the 11 wire bytes sum to 0xF6 (mod 256),
  and the 9 plaintext bytes sum to the header byte (mod 256).
- Byte 1 of the payload is a counter. The ECU echoes it. The counter
  picks the controller's page; the ECU's page is the high nibble of
  plaintext byte 8.

The full detail, including every field, is in the [protocol
document](blue-wire-v2-protocol.md).

## Safety

This drives a combustion heater. Supervise every running test, fit a
carbon monoxide alarm near the heater and its exhaust, and treat
flame-out and overheat handling as the parts that must be right. Getting
the fuelling fields wrong can over-fuel or starve the burn. If the
controller stops talking, the ECU shuts the heater down on its own after
about 5 seconds.

## Contributing captures

The write-up marks what is verified, inferred and still unknown. The
quickest way to fill the gaps is captures from a running heater, ideally
from more than one controller family. The most useful run covers start,
ignition, each power level, thermostat mode, prime and stop, plus each
error triggered on purpose. Record the raw 11-byte wire bytes in both
directions and a timestamped note of each action. Open an issue or a
pull request.

## Licence

GPLv3. See [LICENSE](LICENSE). Free to use, including the firmware when
it lands. If you build on it, keep the credit and release your changes
under the same licence.

Copyright (C) 2026 setsid.
