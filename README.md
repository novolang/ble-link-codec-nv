# ble-link-codec-nv

The link layer is the bottom of the Bluetooth Low Energy (LE) protocol
stack: it is what puts packets on the air and takes them off again. It
is specified in the [Bluetooth Core Specification](https://www.bluetooth.com/specifications/specs/core-specification/),
Volume 6, Part B. This package is the half of the link layer that is not
a radio: the packet formats, and the arithmetic that says which channel
a connection uses next. A host stack never sees these packets, because
it talks to a controller instead over the Host Controller Interface,
which is [hci-codec-nv](https://novo-lang.org/packages/hci-codec-nv).
This package is for the programs on the other side of that interface: a
controller's own firmware, and a sniffer reading the air.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What the link layer is

An LE radio uses 40 channels. Three of them are **advertising
channels**, where a device that wants to be found broadcasts, and 37 are
**data channels**, where a connection lives. A packet on either is a
**protocol data unit** (PDU): a header that says what kind of packet it
is and how long it is, then a payload.

An **advertising PDU** has a two-byte header. The first byte is a
four-bit type, a reserved bit, the ChSel bit and the TxAdd and RxAdd
bits, from the low bit up. The second byte is the payload length.
Section 2.3, Table 2.3, lists the types. The largest payload an uncoded
radio carries is 37 bytes.

| Code | PDU | What it is |
| --- | --- | --- |
| 0x0 | ADV_IND | Connectable and scannable undirected advertising |
| 0x1 | ADV_DIRECT_IND | Connectable directed advertising, to one peer |
| 0x2 | ADV_NONCONN_IND | A broadcast that answers nothing |
| 0x3 | SCAN_REQ | A scanner asking the advertiser for more |
| 0x4 | SCAN_RSP | The advertiser's answer |
| 0x5 | CONNECT_IND | The packet that creates a connection |
| 0x6 | ADV_SCAN_IND | Scannable undirected advertising |
| 0x7 | ADV_EXT_IND | Extended advertising, carried here as its bytes |

The payload of an advertising PDU carries the advertiser's address and
its **advertising data**. Advertising data is a list of **AD
structures**, each a length byte, a type byte and the bytes of that
type. The Core Specification Supplement, Part A section 1, defines the
list and the assigned type numbers. A length byte of zero ends it, which
is how a buffer padded with zeroes is read correctly.

A CONNECT_IND carries the two addresses and then 22 bytes called the
**LLData**: the access address that names the connection on the air, the
CRC seed, the transmit window, the connection interval, the peripheral
latency, the supervision timeout, the channel map, the hop increment and
the initiator's clock accuracy. Section 2.3.3.1 defines it. Every number
is in the specification's own units, and none of them is a time in
milliseconds.

A **data-channel PDU** has a two-byte header of its own, defined in
section 2.4. Two bits of it, the LLID, say what the payload is.

| LLID | Meaning |
| --- | --- |
| 0b00 | Reserved. A packet carrying it is a fault |
| 0b01 | A continuation fragment of an L2CAP message, or an empty packet |
| 0b10 | The start of an L2CAP message |
| 0b11 | A **control PDU**, which is the link layer talking to itself |

Control PDUs run the procedures a connection needs: changing the
parameters, changing the channel map, exchanging features and versions,
starting and pausing encryption, and ending the link. Section 2.4.2
defines them.

A connection **hops**. Each connection event uses a different data
channel, chosen by an algorithm both sides run independently and which
must give the same answer on both without a message passing. Section
4.5.8 defines two. Algorithm #1 adds a hop increment to the previous
channel and wraps at 37. Algorithm #2 is a function of the event counter
alone, so a peripheral that slept through forty events computes the
right channel for the forty-first without walking the thirty-nine it
missed. Both then consult the **channel map**, five bytes with one bit
per data channel, and a channel that is not in the map is replaced by a
used one.

| Quantity | Value |
| --- | --- |
| A device address | 6 bytes |
| The largest advertising payload | 37 bytes |
| The largest data payload without Data Length Extension | 27 bytes |
| The largest data payload with it | 251 bytes |
| Data channels | 37 |
| A channel map | 5 bytes |
| A feature set | 8 bytes |
| A CONNECT_IND's LLData | 22 bytes |
| A hop increment | 5 to 16 |

This package performs no input or output. It reads no register, writes
to no address and consults no clock. A decoded packet is a value the
caller holds, so two connections are two values and nothing is staged in
a buffer that a second subsystem might also be using. The timing, the
antenna and the direct memory access stay in the controller.

## Install

```
novo pkg add ble-link-codec-nv
```

## Example

```novo
use ble_link
use ble_channel

fn main() [io]
    // The advertising data of an ADV_IND: one AD structure, the Flags,
    // saying the device is generally discoverable and is LE only.
    let data: [u8] = [0x02 as u8, 0x01 as u8, 0x06 as u8]

    // Read the advertising data as the list of AD structures it is.
    match ble_link.decode_ad_structures(data)
        Err(e)  => println("the advertising data is wrong: ${e.message()}")
        Ok(ads) =>
            // Pick out one structure by its assigned type number.
            match ble_link.find_ad_structure(ads, ble_link.AD_FLAGS)
                None    => println("this advertiser sent no flags")
                Some(f) => println("flags: ${f.data[0] as Int}")

    // Which data channel the next connection event uses, by algorithm
    // #1: the previous channel plus the CONNECT_IND's hop increment,
    // wrapped at 37, with every channel in the map.
    match ble_channel.next_channel_csa1(35, 5, ble_link.all_channels())
        Err(e) => println("this connection has no channel left: ${e.message()}")
        Ok(ch) => println("the next event is on channel ${ch}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented: ble_link.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `ble_link` | The packet formats. Device addresses, the advertising PDUs and their header, the AD structures and their assigned type numbers, the CONNECT_IND payload, the data-channel header, the control PDUs, and the channel map with its accessors. |
| `ble_channel` | The two channel-selection algorithms, as pure functions of the values the caller already holds. |

## How to choose an entry point

**`decode_adv_header` reads the two header bytes only.** The type and
the length decide whether the payload is worth reading, and on a device
that decision is made before the payload has finished arriving. Call it
first on a receive path. `decode_adv_pdu` reads the whole packet into a
typed value, which is what a sniffer or a host-side tool wants.

**Channel selection #1 needs the previous channel; #2 does not.** Use
whichever the connection negotiated: the ChSel bit in the advertising
header says the advertiser supports #2, and the CONNECT_IND's own ChSel
bit selects it. A peripheral using connection latency wants #2, because
#1 has to be iterated once per missed event and #2 does not.

**`unmapped_channel_csa2` is the first half of algorithm #2.** It gives
the channel before the map is consulted. Use it in a conformance
harness, where knowing which half of the algorithm disagrees with the
specification is the point. An ordinary caller wants `channel_csa2`.

## The rules a user needs

1. **The bytes start at the PDU header.** The preamble, the access
   address and the CRC are the radio's, and every controller worth the
   name has already stripped them. `decode_adv_pdu` is given the two
   header bytes and what follows (Core Vol 6 Part B section 2.3).
2. **Every field is in the specification's own units.** A connection
   interval is counted in 1.25 ms steps and a supervision timeout in
   10 ms steps (section 2.3.3.1). Nothing here converts them, because
   the specification's number is the one a controller programs its timer
   with.
3. **An `instant` is a connection-event count, not a time.** A
   connection update and a channel-map update both take effect at one
   (section 2.4.2.1).
4. **A length byte is not a field you set.** An `AdStructure` carries
   its type and its data, and the encoder writes the length from them. A
   value holding both could disagree with itself.
5. **A zero length byte ends the advertising data.** A length that runs
   past the end of the buffer is `LlBadAdStructure`, and it names the
   offset, which is what a capture tool needs to show where the data
   went wrong (Core Specification Supplement Part A section 1).
6. **The address-type bit is part of the address.** `LlAddress.random`
   is the TxAdd or RxAdd bit of the header. The same six bytes are a
   different device depending on that bit, so the two travel together
   (section 1.3).
7. **A code this package does not know keeps its number.** A reserved
   advertising PDU type decodes to `LlAdvReserved` carrying the code,
   and the reserved LLID decodes to `LlDataReserved`. A decode of a
   capture says what it saw rather than guessing.
8. **An unknown control opcode leaves you owing a reply.**
   `decode_control_pdu` answers `LlUnknownControlOpcode` carrying the
   byte, and the peer is owed an LL_UNKNOWN_RSP quoting it (section
   2.4.2.10).
9. **The acknowledgement bits are the caller's.** The two sequence
   bits and the more-data bit of a data header belong to the sender's
   state machine. This package carries them and decides none of them,
   because a codec that chose a sequence number would be a link layer
   (section 2.4).
10. **A map that uses no channel has no answer.** Selection returns
    `LlNoChannelUsed` rather than channel 0, because channel 0 would be
    a connection transmitting where it was told not to. A connection in
    that state must end. `used_channel_count` still counts such a map,
    which is how a caller finds out (section 4.5.8).
11. **The hop increment is the connection's, from 5 to 16.** It arrives
    in the CONNECT_IND and is an argument to `next_channel_csa1` rather
    than state, because this package holds no connection (section
    4.5.8.2).
12. **Algorithm #2 needs a channel identifier, not an access address.**
    It is `(access_address >> 16) xor (access_address & 0xFFFF)`,
    constant for the life of the connection, so `channel_identifier`
    is called once and its result passed to every event (section
    4.5.8.3.1).
13. **Every multi-byte field is little-endian and every device address
    is least-significant byte first.** That is the link layer's
    convention throughout.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers both modules. This code's production home
is a controller, so a link-layer codec that did not compile for one
would be a library about radios for machines that have none.

```bash
novo build --target=nrf52-qemu src/main.nv
```

`tests/embedded_probe.nv` is that claim as a program that either builds
or does not. It builds, producing a Cortex-M4 executable. The package
has no dependencies, so that build needs nothing but the package and the
probe.

The probe writes no packet down, and a device does not have to. At the
embedded tier a list literal, a boxed struct literal and a
payload-carrying enum variant are all heap allocations. On a device the
bytes arrive from the radio's receive buffer and the values arrive from
the connection, so the caller already holds everything these functions
take.

## What is not included

- **The Common Extended Advertising Payload.** `AdvExtInd` carries its
  bytes and stops there. Extended advertising's payload is a format of
  its own with an optional-field bitmap, and a half-modelled version of
  it would be worse than none.
- **The PHY, data-length and connection-parameter-request procedures.**
  LL_PHY_REQ and its pair, LL_LENGTH_REQ and its pair, and
  LL_CONNECTION_PARAM_REQ and its pair are absent. Each is a procedure
  with a state machine attached, and publishing the packet without the
  procedure would suggest a support a consumer does not have.
- **The CRC and the whitening.** Both are per-bit transformations a
  radio does in hardware. A software implementation of either is a test
  fixture rather than a library.
- **Encryption.** LL_ENC_REQ and LL_ENC_RSP are coded as packets.
  Deriving the session key and running AES-CCM over a packet is not
  here.
- **Anything with a timer in it.** The connection event, the advertising
  interval, the window widening and the supervision timeout are all the
  controller's, and the controller is what this package is not.
- **The radio.** Modulation, the preamble, access-address matching and
  the antenna are hardware. Bytes arrive as bytes and leave as bytes.

## Related packages

- [hci-codec-nv](https://novo-lang.org/packages/hci-codec-nv) is the
  interface a host uses instead of this one. Speak HCI when the radio is
  behind a controller's firmware, and speak the link layer when you are
  that firmware.
- [l2cap-nv](https://novo-lang.org/packages/l2cap-nv) is the layer above
  the data channel. An LLID of 0b10 marks the start of the L2CAP message
  it decodes.
- [att-nv](https://novo-lang.org/packages/att-nv) is the Attribute
  Protocol and [smp-nv](https://novo-lang.org/packages/smp-nv) is the
  Security Manager. Both run on L2CAP channels, two layers above this
  one.
- The standard library has no Bluetooth module. `std.net` is sockets and
  has nothing to do with a radio.

## Tests

```bash
novo test tests/ble_link_tests.nv       # 36 tests against the signatures
novo test tests/ble_channel_tests.nv    # 9 tests over the two algorithms
```

The byte strings the suite asserts against are the Bluetooth Core
Specification's own, Volume 6 Part B: the section 2.3 advertising
header, the section 2.3.1.1 ADV_IND, the section 2.3.3.1 CONNECT_IND,
the section 2.4 data header and the section 2.4.2 control PDUs. The AD
structures come from the Core Specification Supplement, Part A
section 1. The suite checks that each PDU type has its Table 2.3 code,
that every packet survives encoding and decoding, that a truncated
packet is refused by length, and that a channel map's accessors agree
with the bits.

The channel suite asserts the arithmetic the specification states as a
formula: algorithm #1's addition modulo 37, the remapping of a channel
the map does not use, and algorithm #2's channel identifier. It does not
write down the section 4.5.8.3.5 sample channel table, which is numbers
rather than a formula. What it asserts instead is the property every
answer must have whatever that table says: a channel from 0 to 36 that
the map actually uses. The implementation transcribes the table from the
Core Specification.

The tests compile today and fail at run, each on the `not implemented`
panic that is its body. That is the expected state of an interface
release. They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| The `AD_*`, `ADDRESS_LEN`, `*_PAYLOAD_*`, `DATA_CHANNEL_COUNT`, `CHANNEL_MAP_LEN` and `FEATURE_SET_LEN` constants | yes (they are constants) |
| `ble_link.encode_adv_pdu`, `.decode_adv_pdu`, `.adv_pdu_type` | no |
| `ble_link.encode_adv_header`, `.decode_adv_header`, `.adv_type_code`, `.adv_type_of` | no |
| `ble_link.encode_connect_ind_payload`, `.decode_connect_ind_payload` | no |
| `ble_link.encode_ad_structures`, `.decode_ad_structures`, `.find_ad_structure` | no |
| `ble_link.encode_data_header`, `.decode_data_header` | no |
| `ble_link.encode_control_pdu`, `.decode_control_pdu`, `.control_opcode` | no |
| `ble_link.channel_map`, `.all_channels`, `.channel_used`, `.used_channel_count`, `.used_channel_at` | no |
| `ble_link.LlError.message` | no |
| `ble_channel.next_channel_csa1` | no |
| `ble_channel.channel_identifier`, `.channel_csa2`, `.unmapped_channel_csa2` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
