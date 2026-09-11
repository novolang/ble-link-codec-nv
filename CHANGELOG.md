# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

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
