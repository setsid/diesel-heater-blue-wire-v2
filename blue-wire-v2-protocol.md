# Blue-wire v2 protocol (AA/A5 pulse-width)

Reverse-engineered from passive bench captures of one Chinese diesel
heater ECU and its stock controller. No firmware was read from either
device. Every claim here is backed by a capture or by a decoder that
reproduces the captured bytes. Fields that are not proven are marked
inferred or unknown.

## 1. What it is

This is the signalling on the single blue data wire between the stock
controller and the ECU on one diesel heater. It is a pulse-width coded,
scrambled, request and reply protocol. The controller sends an 11-byte
message, the ECU replies with 11 bytes, and this repeats about five
times a second while the heater is powered.

It is a different protocol from the older 25000 baud UART blue-wire
protocol that most published work describes. Tell the two apart on a
scope:

- Old protocol: a continuous UART line, 25000 baud, 8 data bits, no
  parity, 1 stop bit, LSB first, about 40 us per bit. It is documented
  by Ray Jones in his V9 note, [Hacking the Chinese Diesel Heater
  Communications Protocol](https://gitlab.com/mrjones.id.au/bluetoothheater/-/blob/f041cf8cad7d7d238597f63c912aa9bf9fb1e936/Documentation/V9%20-%20Hacking%20the%20Chinese%20Diesel%20Heater%20Communications%20Protocol.pdf).
  That document is the reference for the old protocol. This write-up
  does not cover it.
- v2 (this document): no UART framing. A line that idles high, with each
  bit sent as a low pulse then a high pulse, messages opened by a sync
  low of about 3 ms, and a header byte of 0xAA or 0xA5. One controller
  message and one ECU reply every 201 ms, with an idle gap between.

### The unit tested

One bench ECU taken from a working garage diesel heater, driven by its
own stock LCD controller. The controller shows the power level as two
digits, P-01 to P-10. It has a settings menu reached with PIN 1234,
whose screen is labelled P1H2 and holds four fields (operating mode,
supply voltage, power class, pump size), a spanner menu key, and a
"-PE-" remote-pairing mode. This heater's own settings are P (manual),
12 V, H (5 kW class), pump size 2 (22 mL).

The ECU circuit board markings and the controller brand are not recorded
in the source captures, so the exact part numbers are unknown. Manuals
for comparable controllers show either 1 to 6 or 1 to 10 power steps;
this one runs 1 to 10. Other heaters in this family may differ in the
field meanings below, so map any new unit before driving it.

## 2. Safety

This drives a combustion heater that burns diesel and moves hot exhaust.

- Supervise every running test. Do not leave a heater under test
  unattended.
- Behaviour on loss of the controller during a burn is untested here. A
  replacement controller must fail safe. Do not assume the ECU shuts
  down cleanly if your controller stops talking mid-run.
- Fit a carbon monoxide alarm in any occupied space near the heater or
  its exhaust.
- Getting the fuelling fields wrong can over-fuel or starve the burn.
  Treat flame-out and overheat handling as the parts that must be right.

## 3. Physical layer

One data wire, idle high. The loom carries three wires:

- blue: data.
- red: +5 V, from the ECU's own buck converter.
- black: ground.

The bus is open-collector (open-drain). A device pulls the line low to
send and releases it to let it float high. The high level is held by a
pull-up.

**The pull-up lives in the controller.** This ECU does not provide one.
It has a weak pull-down on the blue wire, a series resistor of about
499R into its input, and a clamp diode. With no controller attached the
line sits low. A replacement controller must supply the pull-up itself.

### Interface that worked

Bench-proven on 2026-11-08. The ECU answered a replacement controller
only with this arrangement.

```
        red +5 V (from the ECU)
              |
             R2  2.2k        pull-up (the controller must supply this)
              |
  blue wire --+----------------------------------- ECU input
              |                                     (~499R series,
     +--------+---------+                            weak pull-down,
     |                  |                            clamp diode)
  [R1 270R]        2N2222A: collector on blue
     |             base driven from the TX pin through a resistor
  RX listen        emitter to ground
  (MCU input,
   high-Z, no
   internal pull)
```

- Pull-up R2: 2.2k from the blue wire to the loom's +5 V. The line then
  idles at about 4.08 V on a 5.05 V supply.
- Drive: a 2N2222A with its collector on the blue wire pulls the line to
  0 V when the transmit pin turns it on.
- Listen: a separate input taps the wire through R1 270R. That input must
  have no internal pull, so it does not fight R2.
- A 5 V TVS on the line is useful for protection.

Do not drive the wire with a plain push-pull output. It would fight R2
and hold the line at the logic-high voltage instead of releasing it.

### Gotchas

- **A stock controller left on the blue wire kills it.** Even with its
  own 5 V removed, a stock controller still connected to the blue wire
  loads the line and stops the ECU answering a replacement. It must be
  fully off the blue wire, not just unpowered. Several early failures to
  get a reply were traced to this.
- A 680R pull-up (idle about 4.31 V) did not get a reply. 2.2k did.
- A low pull-up, around 330R, made the stock controller itself report a
  comms fault (E-07) and stop sending.
- For reference, the stock controller idles the line at about 3.82 V,
  rises to about 4.34 V during messages, and pulls to 0 V for a low.

## 4. Bit layer

Each bit is one low pulse followed by one high pulse. The width of the
low part carries the bit:

| Bit | Low | High |
|---|---|---|
| 1 | about 300 us | about 100 us |
| 0 | about 100 us | about 300 us |

Decode by comparing the two halves: a long low then a short high is a 1,
a short low then a long high is a 0.

- Bit period: about 401.8 us from the controller and about 408.7 us from
  the ECU. The two devices run free-running RC clocks about 1.7 per cent
  apart. About 2.5 kbit/s.
- Bit order: most significant bit first within each byte.
- Message frame: a sync low of about 3.1 ms, then about 100 us high,
  then 88 data bits (11 bytes), then one closing low pulse.
- The controller prefixes a short pre-pulse, about 180 us low then
  220 us high, before its sync. The ECU sends its sync with no pre-pulse.

### Exchange period

One exchange is a controller message followed by the ECU's reply, which
starts straight after the controller's last bit.

- One exchange every 201 ms.
- Each message lasts about 38.5 ms.
- About 122 ms idle between exchanges.
- With no controller on the line the ECU sends nothing. It does not reply
  to old-protocol UART frames.

## 5. Frame layer

Each message is 11 bytes on the wire, c0 to c10.

- c0 is the header, sent in clear: 0xAA from the controller, 0xA5 from
  the ECU.
- c1 to c9 carry nine scrambled plaintext bytes p1 to p9.
- c10 is an outer checksum.

### Scrambling

```
c0   = header (0xAA controller, 0xA5 ECU), not scrambled
c[j] = p[j] XOR 0x53 XOR crc8(c[j-1])            for j = 1..9
c10  = (0xF6 - (c0 + c1 + ... + c9)) mod 256      outer checksum
p9   = (header - (p1 + ... + p8)) mod 256         inner checksum

crc8(v) = CRC-8 over the single byte v: polynomial 0x31, init 0x00,
          no input or output reflection, no final XOR. Equivalently
          v * x^8 mod (x^8 + x^5 + x^4 + 1).
```

Decoding is `p[j] = c[j] XOR 0x53 XOR crc8(c[j-1])`. It needs no state
carried between messages. A valid message passes both checks:

- the 11 wire bytes sum to 0xF6 mod 256, and
- plaintext bytes p1 to p9 sum to the header byte mod 256.

Reference CRC-8 values: crc8(0x00) = 0x00, crc8(0x01) = 0x31,
crc8(0x80) = 0x7A. The function is linear over GF(2), which is what the
feedback form of the scrambler exploits.

### Counter and pages

- p1 is a counter. It increments by one each exchange and wraps at 256.
- The ECU echoes the controller's counter in its reply, so a reply can
  be matched to its request.
- The counter selects the controller page: page A on odd counters, page B
  on even. The controller alternates the two.
- The ECU cycles four pages. Identify the ECU page from the high nibble
  of plaintext byte 8 (values 0 to 3). The ECU's page order follows its
  own reply sequence, so do not use the counter to identify the ECU
  page. Use byte 8's high nibble.

### Minimal decode and encode

```
decode(c[0..10]):
    assert c[0] in {0xAA, 0xA5}
    assert sum(c[0..10]) mod 256 == 0xF6
    for j in 1..9: p[j] = c[j] XOR 0x53 XOR crc8(c[j-1])
    assert sum(p[1..9]) mod 256 == c[0]
    return counter = p[1], data = p[2..8]

encode(header, counter, data[2..8]):
    p[1] = counter; p[2..8] = data
    p[9] = (header - sum(p[1..8])) mod 256
    c[0] = header
    for j in 1..9: c[j] = p[j] XOR 0x53 XOR crc8(c[j-1])
    c[10] = (0xF6 - sum(c[0..9])) mod 256
    return c[0..10]
```

## 6. Fields

Plaintext bytes are numbered p1 to p9 as on the wire. p1 is the counter.
p2 to p8 carry the data. p9 is the inner checksum.

Status column:

- **verified**: changed exactly as predicted when the operator acted, or
  proven by the decoder over every captured message.
- **inferred**: tracks a physical quantity or matches the controller
  manuals, but is not independently calibrated, or was seen to change
  only in a single run.
- **unknown**: never seen to change, or meaning not established.

### Controller page A (counter odd)

| Byte | Values seen | Meaning | Status |
|---|---|---|---|
| p2 | 0x20 | constant | unknown |
| p3 | 0x03 | constant | unknown |
| p4 | 0, then +1 per command | command sequence number | verified |
| p5 | 0 idle, 1 start, 2 stop, 5/4/3 during prime, 6 at power-up | command | start verified; stop and prime codes inferred |
| p6 | 1 to 10 | power level, P-01 to P-10, plain number | verified |
| p7 | 0, 20, 11 | command parameter, possibly prime length | inferred |
| p8 | 0x00 | constant | unknown |

Start is byte p4 stepping up by one and byte p5 going to 1. It is
confirmed on the wire and acknowledged by the ECU. Stop is p5 = 2 with
p4 stepping up; it has not been confirmed against a running heater. The
prime codes (5 then 4 then 3) and the p7 parameter come from one prime
only.

### Controller page B (counter even)

| Byte | Values seen | Meaning | Status |
|---|---|---|---|
| p2 | 0x00 | constant | unknown |
| p3 | 0x00 | constant | unknown |
| p4, p5 | room temperature, 0.1 C, 16-bit little endian | controller's measured room temperature | verified |
| p6 | thermostat target, C | target in thermostat mode; reads 35 in mode A | verified |
| p7 | settings bit field | installer settings, see below | bits verified; meanings inferred |
| p8 | 0x00 | constant | unknown |

The below-zero encoding of room temperature is unknown. Only positive
values were seen.

Settings bit field (page B, byte p7). Bit positions are confirmed on the
bus. The meanings come from the controller's own menu and manuals.

| Bit | Menu field | Values | Meaning | Status |
|---|---|---|---|---|
| 1 | mode | 1 = P, 0 = t or A | P manual, t thermostat, A automatic | bit verified; meaning inferred |
| 0 | voltage | 0 = 12 V, 1 = 24 V | the ECU echoes this as page 2 reading 300 or 301 | verified |
| 2 | power class | 1 = H, 0 = L | H 5 kW, L 2 kW | bit verified; meaning inferred |
| 3, 4 | pump size | 00 = 1, 01 = 2, 10 = 8 | 16 / 22 / 28 mL | bits verified; meaning inferred |

This heater sends byte p7 = 0x0E: P, 12 V, H, pump 2. For another
heater, p7 must match that heater's own settings, because voltage, power
class and pump size change the ECU's fuelling.

### ECU pages (identify by byte p8 high nibble)

Page 0:

| Byte | Meaning | Status |
|---|---|---|
| p2, p3 | fan speed, rpm, 16-bit little endian | inferred |
| p4, p5 | fan drive, units unknown, tracks fan speed | inferred |
| p6, p7 | heat exchanger temperature, 0.1 C, signed 16-bit little endian | inferred |

Pages 1 and 3:

| Byte | Meaning | Status |
|---|---|---|
| p4 | supply voltage, 0.1 V, about 0.3 to 0.4 V under the terminal voltage | verified |
| p5 | load flags: bit 7 glow plug, bit 6 fan, bit 4 fuel pump | inferred |

Page 3 only:

| Byte | Meaning | Status |
|---|---|---|
| p2, p3 | fuel pump pulse period, 0.1 ms, little endian (6250 = 1.60 Hz at P1, 3000 = 3.33 Hz at P5); meaningful only while the pump flag is set | inferred |
| p6, p7 | glow plug current, 0.01 A, little endian | inferred |

Page 1 only:

| Byte | Meaning | Status |
|---|---|---|
| p2, p3 | 0x2C 0x01 (300), constant on this unit | unknown |

Page 2:

| Byte | Meaning | Status |
|---|---|---|
| p2, p3 | 300, or 301 while the 24 V setting is selected: the ECU's echo of the voltage setting | verified for the echo |
| p4, p5 | glow-plug related, counts up while the glow plug is on | inferred |
| p6 | the power level the ECU is actually running; steps one level about every 10 s towards the requested level; 0 when not running | inferred |
| p7 | glow-plug related, 39 to 70 while on, 0 when off; possibly voltage or power | inferred |

All ECU pages, byte p8:

| Nibble | Meaning | Status |
|---|---|---|
| high (0 to 3) | page marker | verified |
| low | status or fault nibble, see below | states inferred |

Any ECU data byte reading 0x00 above and not listed is unknown.

### ECU status nibble (byte p8 low nibble)

| Value | State | Status |
|---|---|---|
| 0 | booting, first second after power-up | inferred |
| 1 | starting: fan spin-up, glow plug preheat | inferred |
| 3 | running | inferred |
| 4 | stopping: clearing burn with the glow plug, then fan cool-down | inferred |
| 5 | idle, ready | inferred |
| 6 | glow plug fault, shown on the bench with no glow plug fitted | inferred |
| 7 | priming | inferred |

These states come from one full garage cycle captured listen-only with
operator step markers and zero decode errors. The correlation is strong
but it is a single run, so each state is inferred until seen again.

A separate event value of 0x10 appears in ECU byte p5 on pages 1 and 3:
in the first replies after power-up, and in one reply after each change
of the run request. It may be an acknowledgement or event flag. Its
meaning beyond that is unknown. Note that 0x10 is also bit 4 of the load
flags (the pump flag), so the two uses of this byte need care.

## 7. Worked example

The first exchange of the idle bench capture. Both frames have the heater
off and the bench ECU showing a glow plug fault.

```
controller wire: AA 07 E4 55 D8 EE 8C 54 E9 09 74
ECU reply wire:  A5 29 5D 61 D9 DF 3C 15 E3 28 56
```

### Controller frame

Outer check: the 11 wire bytes sum to 0xF6. Good.

Descramble p[j] = c[j] XOR 0x53 XOR crc8(c[j-1]):

```
p1 = 0x07 XOR 0x53 XOR crc8(0xAA)=0x27 = 0x73
p2 = 0xE4 XOR 0x53 XOR crc8(0x07)=0x97 = 0x20
p3 = 0x55 XOR 0x53 XOR crc8(0xE4)=0x05 = 0x03
p4 = 0xD8 XOR 0x53 XOR crc8(0x55)=0x8B = 0x00
p5 = 0xEE XOR 0x53 XOR crc8(0xD8)=0xBD = 0x00
p6 = 0x8C XOR 0x53 XOR crc8(0xEE)=0xDE = 0x01
p7 = 0x54 XOR 0x53 XOR crc8(0x8C)=0x07 = 0x00
p8 = 0xE9 XOR 0x53 XOR crc8(0x54)=0xBA = 0x00
p9 = 0x09 XOR 0x53 XOR crc8(0xE9)=0x49 = 0x13
```

Plaintext p1 to p9: `73 20 03 00 00 01 00 00 13`.

Inner check: p1 to p9 sum to 0xAA, the header. Good.

Decoded: counter 0x73 (odd, so page A). Byte p6 = 1, so power level
P-01. Bytes p4 and p5 are 0, so no run request. The constants p2 = 0x20
and p3 = 0x03 are present as always.

### ECU reply

Outer check: the 11 wire bytes sum to 0xF6. Good.

```
p1 = 0x29 XOR 0x53 XOR crc8(0xA5)=0x09 = 0x73
p2 = 0x5D XOR 0x53 XOR crc8(0x29)=0x0E = 0x00
p3 = 0x61 XOR 0x53 XOR crc8(0x5D)=0x32 = 0x00
p4 = 0xD9 XOR 0x53 XOR crc8(0x61)=0x8A = 0x00
p5 = 0xDF XOR 0x53 XOR crc8(0xD9)=0x8C = 0x00
p6 = 0x3C XOR 0x53 XOR crc8(0xDF)=0x2A = 0x45
p7 = 0x15 XOR 0x53 XOR crc8(0x3C)=0xB8 = 0xFE
p8 = 0xE3 XOR 0x53 XOR crc8(0x15)=0xB6 = 0x06
p9 = 0x28 XOR 0x53 XOR crc8(0xE3)=0x92 = 0xE9
```

Plaintext p1 to p9: `73 00 00 00 00 45 FE 06 E9`.

Inner check: p1 to p9 sum to 0xA5, the header. Good.

Decoded: counter 0x73, echoing the controller. Byte p8 = 0x06, so the
high nibble is 0 (page 0) and the low nibble is 6 (glow plug fault).
Page 0 bytes p6 and p7 are 0x45 0xFE, which as a signed little-endian
16-bit value is -443, a heat exchanger temperature of about -44.3 C.
That is the open-sensor reading of the bench ECU with no heat exchanger
probe fitted. Fan speed and fan drive (p2 to p5) are 0.

A decoder that reproduces these two lines, both checksums, counter,
page selection and the decoded fields, is reading the protocol
correctly.

## 8. Unknowns

Confirming these needs captures from running heaters, ideally from more
than one controller family, with step markers noting what the operator
did and when.

- **Full power range on a running heater.** Only 1 to 10 was seen, on a
  controller that shows two digits. The valid range on a running heater,
  and whether the ECU clamps an out-of-range level, are unconfirmed. With
  the heater off the ECU accepts any level byte silently.
- **Prime.** The prime codes (controller byte p5 = 5 then 4 then 3) and
  the byte p7 parameter come from a single prime. The exact meaning of
  p7, and whether it is the prime length, are unconfirmed.
- **Run states and stop.** The status nibble states 1, 3 and 4 come from
  one garage cycle. The stop command (byte p5 = 2) has not been
  confirmed against a running heater, because every start attempt on the
  bench was refused with a latched fault.
- **Admin and installer settings.** A search of 15,534 decoded messages
  found no pump Hz, fan rpm, voltage or sensor-type setting in routine
  traffic, in either direction, and the admin PIN never appeared on the
  bus. Whether these are exchanged only while the admin menu is open, or
  held in the controller and never sent, is unknown.
- **Constant bytes.** Controller page A p2 (0x20) and p3 (0x03); ECU page
  1 p2, p3 (300); ECU page 2 p4, p5 and p7 (glow-plug related). Meanings
  unknown.
- **The 0x10 event flag** in ECU byte p5. Meaning beyond "seen after
  power-up and after a run-request change" is unknown.
- **Error-code numbering.** Only the glow plug fault (status 6) was seen.
  Error-code numbers differ between controller families, so the mapping
  for this ECU is confirmed only when each code is triggered on the bench
  and found in the decoded bytes.
- **Units and calibration.** Fan drive units, the glow-plug "voltage or
  power" byte, and the exact fan-speed scaling are not independently
  calibrated.
- **Below-zero room temperature** encoding in controller page B.

Captures welcome. The most useful are a full run through start, ignition,
each power level, thermostat mode, prime and stop, plus each error
triggered deliberately, with the raw 11-byte wire bytes in both
directions and a timestamped note of each operator action.
