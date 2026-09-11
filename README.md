# ble-link-codec-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

The Bluetooth LE link-layer packet formats with no radio under them
(Core Vol 6 Part B): the advertising PDUs and the AD structures they
carry, the CONNECT_IND payload, the data-channel header, the control
PDUs a connection exchanges, and the two channel-selection algorithms as
pure functions.

The link layer is the one place in a BLE stack where a radio is
genuinely present, and this package is the half of it that is not.  Every
doc comment names the section its format comes from, because the only
way to check a codec is against the specification that defines it.

## Adding it, and checking it

```bash
novo pkg add ble-link-codec-nv    # into your novo.toml
novo pkg build                    # type- and effect-check the package
novo test tests/ble_link_tests.nv
novo test tests/ble_channel_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion below the first constants fails with `not implemented:
ble_link.<fn>`.  They turn green one at a time as bodies land.

## The one example that will work

```novo
use ble_link

// A scanner's whole receive path: bytes from the radio, a decision.
fn on_advertisement(bytes: [u8]) -> ?AdStructure
    match ble_link.decode_adv_pdu(bytes)
        Err(_)  => None
        Ok(pdu) =>
            match pdu
                AdvInd(_, data) =>
                    match ble_link.decode_ad_structures(data)
                        Err(_)  => None
                        Ok(ads) => ble_link.find_ad_structure(ads, ble_link.AD_COMPLETE_LOCAL_NAME)
                _ => None
```

## The layer, and why

`core`.  Everything here is arithmetic over bytes the caller already
holds and values the caller already has: the channel map is a value, the
event counter is an argument, the hop increment is an argument.  The
timing, the antenna and the DMA stay in the controller, which is the
only part of a link layer that genuinely needs them.

`tests/embedded_probe.nv` is that claim in a form that either builds or
does not, and **it builds**: `novo build --target=nrf52-qemu` produces a
Cortex-M4 ELF.  This package has no dependencies, so the shard audit's
`core-embedded` row runs that build itself and passes.

## The load-bearing interface

One struct, and it is a struct rather than a buffer on purpose.

```novo
pub struct LlConnectIndPayload
    access_address: Int
    crc_init: Int
    window_size: Int
    window_offset: Int
    interval: Int
    latency: Int
    timeout: Int
    channel_map: LlChannelMap
    hop_increment: Int
    sleep_clock_accuracy: Int
```

**A value has nowhere to collide with.**  The reference controller
stages a received CONNECT_IND at a fixed RAM address and reads its
fields back out of that address one accessor at a time, and a second
subsystem — the encryption context, which holds the long-term key, the
session-key diversifier and the packet counters — claims the same
address.  Nothing enforces that the two are never live at once, so a
peer that reconnects while the previous connection's context is still in
those bytes overwrites key material with connection parameters.  That
is filed against `orbit/ble` and it is open.

A codec whose decoded packet is a value the caller holds makes that
class of defect **impossible rather than unlikely**: there is no address
to share, the lifetime is the caller's, and two connections are two
values.  It is the single clearest argument for splitting a link layer
into a codec and a controller, and it is why this package's header
comment says what it does not hold is an address.

That is a claim about the codec's shape, not a fix for the filing.  The
overlapping constants are in `orbit/ble`, and they stay there until
either that package moves one of them or it adopts this one; adopting
this one removes the staging buffer, which is the half of the collision
the filing calls the natural mover.

The second decision worth stating is the units:

```novo
    interval: Int     // 1.25 ms steps, as § 2.3.3.1 writes it
    timeout: Int      // 10 ms steps
```

Nothing is converted to milliseconds.  The number a controller programs
its timer with is the specification's own, and a codec that helpfully
converted would be handing back a number its caller has to convert
again — losing the exact value on the way in both directions.

## What is not here, and what a consumer should expect

**Extended advertising's payload format.**  `AdvExtInd` carries its
bytes.  The Common Extended Advertising Payload is a format of its own
with an optional-field bitmap, and a half-modelled version of it would
be worse than none.

**The PHY, data-length and connection-parameter-request procedures** —
LL_PHY_REQ/RSP/UPDATE_IND, LL_LENGTH_REQ/RSP, LL_CONNECTION_PARAM_REQ
and its response.  The control set here is the one a connected
peripheral meets on a 4.2 link plus the encryption pairs; each of the
absent ones is a procedure with a state machine attached, and adding the
PDU without the procedure would suggest a support this package's
consumer does not have.

**The CRC and the whitening.**  Both are per-bit transformations a radio
does in hardware, and a software implementation of either is a test
fixture rather than a library.

**Anything with a timer in it.**  The connection event, the advertising
interval, the window widening, the supervision timeout: all of them are
the controller's, and all of them are why the controller stays in
`orbit/ble`.

## The reference implementation

`orbit/ble`'s controller, spread across four modules this package makes
two: `controller/ll_frame.nv` (the PDU-type constants and the header bit
packing — the one piece that is already pure), `controller/ll_adv_pdu.nv`
(the ADV_IND and SCAN_RSP builders), `controller/ll_connect_ind.nv` (the
CONNECT_IND builder and the peripheral-side parser) and
`controller/ll_control.nv` (the control-PDU builders).

The thing that changes in the port is the same thing in three of those
four modules: **the address goes away**.  `ll_adv_pdu.nv` writes into
`0x2000F000` through `adv_pdu_buf_write_byte` and every builder carries
`[hw]` for that reason alone; `ll_connect_ind.nv` stages at `0x2000F100`
and reads its fields back with sixteen `read_connect_ind_*() [hw]`
accessors; `ll_encryption.nv` and `enc_ctx.nv` claim the same window.
None of those functions touches a peripheral register — `[hw]` is on
them because a fixed RAM address is `[hw]` — so the port is a codec with
an empty effect row and a caller who owns the buffer.

`ll_control.nv`'s seven `build_ll_*` functions are already pure and are
the cheapest part of the move; what they gain is a decoder, which the
reference does not have at all — the controller recognises a control PDU
by its opcode byte in `hci_dispatcher.nv` and reads the parameters
inline.

The Bluetooth Core Specification Vol 6 Part B and the Core Specification
Supplement Part A are the sources of the test vectors.

## Status

| item | implemented |
| --- | --- |
| the `AD_*`, `ADDRESS_LEN`, `*_PAYLOAD_*`, `DATA_CHANNEL_COUNT`, `CHANNEL_MAP_LEN` and `FEATURE_SET_LEN` constants | yes — they are constants |
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
