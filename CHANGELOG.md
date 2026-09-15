# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

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
