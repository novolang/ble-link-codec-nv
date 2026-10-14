# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## Unreleased — 2026-10-10

### Changed

- **The `pub const` finding is corrected.**  The compiler defect it
  names — a `pub const` in a `match` pattern read as a binding, and the
  same constant lowering to IR that `llvm-as` refuses — is fixed in
  novo 0.20.2.  This package has no workaround to revert: it declares no
  opcode constants, so the three control-PDU tables in `ble_link` have
  no constants to name and none can be restored.  The sentence in the
  0.1.0 entry below that says the tables are "written with literals
  where the constants would read better" is left as written, and it was
  never true of this source.  No line under `src/` changed.

## 0.1.0 — 2026-10-10

The bodies, over the interface 0.0.2 published. Every doc comment's
`novo` example runs, both modules cross-compile for a Cortex-M4, and
the merged line coverage under `src/` is full — provided by
`tests/ble_link_tests.nv`, `tests/ble_channel_tests.nv` and a new
`tests/ble_link_edge_tests.nv` for the paths the API suite does not
reach.

- **A malformed packet is a value.** Every refusal the codec makes is
  an `LlError` variant whose message names the numbers it was given,
  and no function here panics on bytes off the air.
- **Every field keeps the specification's unit.** A connection
  interval, a supervision timeout, a transmit window and an instant
  cross the boundary as the numbers § 2.3.3.1 and § 2.4.2.1 define, and
  nothing converts a millisecond.
- **`LlAdvPduHeader` carries the ChSel bit**, and an encoded PDU writes
  it clear: § 2.3 puts the bit in the advertising header, and no
  `LlAdvPdu` variant holds a channel-selection choice.
- **A code this package does not know keeps its number.** A reserved
  advertising type decodes to `LlAdvReserved(c)`, an undefined control
  opcode to `LlUnknownControlOpcode(op)`, and the reserved LLID to
  `LlDataReserved`, so a decode of a capture reports what it saw rather
  than guessing.
- **`control_parameter_length` is a control PDU's wire length** and
  each encoder sizes its buffer from it exactly, because the device
  tier builds a byte list by filling a buffer and never by growing one.

### Findings

- The permutation of § 4.5.8.3.3 reverses the eight bits of each byte
  and leaves the two bytes where they are. Permuting all sixteen bits
  is the plausible misreading, and the § 4.5.8.3.5 sample channels are
  what rule it out.
- `list.try_push` drops a value that does not fit rather than growing
  the buffer or failing, so a buffer sized one byte short truncates a
  packet in silence. The CONNECT_IND's eleven parameter bytes were
  found this way: a declared length of ten lost the last one, and the
  round trip that re-decoded the truncated packet still passed.
- A tail expression inside a `system` block is discarded by novo
  0.20.1 — an `Int` yields zero and a value that owns heap memory
  crashes — so every `system` block here delivers its value with an
  explicit `return`.  Filed as
  `a-system-block-in-value-position-discards-its-value` and fixed in
  0.20.2 for a function's own tail; the `return`s stay until
  `a-system-block-whose-value-is-a-result-call-is-refused-at-a-device-boundary`
  closes, which is what a device build of these functions still meets.
- A `pub const` named in a `match` pattern is a binding rather than a
  pattern, and the same constant used as one lowers to IR that
  `llvm-as` refuses, so the control-PDU opcode tables are written with
  literals where the constants would read better.
- `tests/ble_link_tests.nv` holds one assertion that cannot hold.
  `test_an_adv_ind_is_its_header_its_address_and_its_data` reads
  `wire[7]` for the flags AD's length byte, which the same test's own
  layout puts at `wire[9]`; it is left as written, and
  `tests/ble_link_edge_tests.nv` asserts the same encoding.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it. Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`. Adding this
package works and calling it panics.

- `LlAdvPdu` and `LlAdvPduHeader` — the advertising-channel packets of
  § 2.3 as values, with the ChSel, TxAdd and RxAdd bits as fields rather
  than as a byte every caller masks for itself.
- `AdStructure` and the `AD_*` numbers — the advertising data as a list,
  with the zero length byte that ends it and the offset of a length that
  runs off the end.
- `LlConnectIndPayload` — the CONNECT_IND's LLData as a value the caller
  holds, in the specification's own units. A value has nowhere to
  collide with, which is the whole argument for the package.
- `LlDataHeader` and `LlDataKind` — the data-channel header, with the
  reserved LLID reported rather than guessed.
- `LlControlPdu` — the control PDUs a connected peripheral meets:
  connection update, channel map update, terminate, the feature and
  version exchanges, the reject and unknown responses, and the
  encryption start and pause pairs.
- `LlChannelMap` and its accessors, and the two selection algorithms in
  `ble_channel` — CSA #1's hop and remapping, and CSA #2 as a function
  of the event counter alone, which is what makes peripheral latency
  affordable.

`tests/ble_channel_tests.nv` asserts the arithmetic the specification
states as a formula and deliberately does NOT write down the
§ 4.5.8.3.5 sample channel table, which is numbers rather than a
formula: a wrong expected value in a red test costs the implementation
lane a day proving the package right and the test wrong.

### Design notes

Moved here from the README, which now states only what a user needs.

- The clearest argument for splitting a link layer into a codec and a
  controller is the address that is not there. The reference controller
  stages a received CONNECT_IND at a fixed RAM address and reads its
  fields back out of that address one accessor at a time, and a second
  subsystem — the encryption context, holding the long-term key, the
  session-key diversifier and the packet counters — claims the same
  address. Nothing enforces that the two are never live at once, so a
  peer that reconnects while the previous connection's context is still
  in those bytes overwrites key material with connection parameters.
  That is filed against `orbit/ble` and it is open. A decoded packet
  that is a value the caller holds makes the whole class impossible
  rather than unlikely: there is no address to share, the lifetime is
  the caller's, and two connections are two values.
- That is a claim about this package's shape, not a fix for the filing.
  The overlapping constants are in `orbit/ble` and they stay there
  until that package either moves one of them or adopts this one.
  Adopting this one removes the staging buffer, which is the half of
  the collision the filing calls the natural mover.
- The port collapses four modules of `orbit/ble` into two:
  `controller/ll_frame.nv` (the PDU-type constants and the header bit
  packing, the one piece that is already pure),
  `controller/ll_adv_pdu.nv` (the ADV_IND and SCAN_RSP builders),
  `controller/ll_connect_ind.nv` (the CONNECT_IND builder and the
  peripheral-side parser) and `controller/ll_control.nv` (the
  control-PDU builders).
- The same thing changes in three of those four modules: the address
  goes away. `ll_adv_pdu.nv` writes into `0x2000F000` through
  `adv_pdu_buf_write_byte`, and every builder carries `[hw]` for that
  reason alone; `ll_connect_ind.nv` stages at `0x2000F100` and reads
  its fields back with sixteen `read_connect_ind_*() [hw]` accessors;
  `ll_encryption.nv` and `enc_ctx.nv` claim the same window. None of
  those functions touches a peripheral register, so the port is a codec
  with an empty effect row and a caller who owns the buffer.
- `ll_control.nv`'s seven `build_ll_*` functions are already pure and
  are the cheapest part of the move. What they gain is a decoder, which
  the reference does not have at all: its controller recognises a
  control PDU by its opcode byte in `hci_dispatcher.nv` and reads the
  parameters inline.
- `LlAdvPduHeader` is spelled longer than the reference's `LlAdvHeader`
  on purpose. A type's identity is its name across a whole assembly, so
  a program holding both the reference's raw two-byte pair and this
  decoded one would be refused. The reference's copy is one of the
  things the split deletes; the longer name is what lets the two
  coexist until then.
