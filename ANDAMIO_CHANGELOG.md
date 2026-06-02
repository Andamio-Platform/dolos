# Andamio Custom Changelog

This file tracks the Andamio-specific commits layered on top of upstream
[`txpipe/dolos`](https://github.com/txpipe/dolos) releases. Each `andamio.N`
tag is the upstream release SHA + the commits listed under that heading,
rebased cleanly (no conflicts).

## `v1.2.0-andamio.2`

Base: [`txpipe/dolos v1.2.0`](https://github.com/txpipe/dolos/releases/tag/v1.2.0)

Rebase of the andamio patch stack from a `v1.0.3` base onto `v1.2.0`. All
retained patches cherry-picked cleanly (no conflicts). The `andamio.N`
counter is continuous: `andamio.2` was the planned `tx_hash` ship; it lands
on a `v1.2.0` base rather than `v1.0.3`.

### `minibf` — optional API `base_path` (unchanged)

Mount `minibf` behind a path prefix (e.g. `/api/v0`). No upstream equivalent.

- `feat(minibf): add optional base_path configuration`
- `fix(minibf): add comprehensive base_path validation`
- `fix(minibf): replace panic with ServeError::ConfigError for invalid base_path`

### `minibf` — source address-utxo `tx_hash` from `TxoRef`

`GET /api/v0/addresses/{address}/utxos` returned an empty `tx_hash` for any
UTxO whose source block had aged out of the archive window (archive miss →
`block_data == None` → `.unwrap_or_default()` → `""`). Breaks the sponsorship
sidecar's sponsor-input assembly and Mesh's `BlockfrostProvider`
(`InvalidStringError: expected length '64', got 0`). Source `tx_hash` from the
UTxO's own `TxoRef` (always present) instead.

- `fix(minibf): source address-utxo tx_hash from TxoRef, not archive block_data`

Ports byte-clean onto v1.2.0: the handler at `crates/minibf/src/mapping.rs`
is unchanged from v1.0.3 aside from the `UtxoBlockData` → `BlockRefMeta`
type rename (the `c469c7fa` refactor); no logic change.

### `validate` — testing-only tx-submit validation changes (unchanged)

- `fix(validate): catch phase-2 panic and skip on failure` — phase-2 script
  evaluation can panic inside pallas on malformed Plutus data; catch the
  panic and skip validation for that tx rather than tearing down the node.
- Reference inputs included in the UTxO lookup; phase-1 ledger validation
  disabled. **Testing-only** — never enable on mainnet.

### REMOVED: `grpc watch` — `fill_input_as_output`

The `v1.0.3-andamio.1` stack carried `fill_input_as_output` to populate
`TxInput.as_output` on the `WatchTx` stream (regular inputs via WAL/archive,
reference inputs via the state store), because upstream returned
`as_output = nil`. **This patch is dropped on the v1.2.0 base — upstream now
populates `as_output` natively** (via the v1.1.0 `LedgerContext::get_utxos`
archive fallback, [`txpipe/dolos#962`](https://github.com/txpipe/dolos/pull/962)).

Verified empirically before dropping:

- Stock v1.2.0 `WatchTx` populates `as_output` on **100%** of inputs across
  570 txs / ~2,000 inputs — spent (1165/1165) and reference (835/835).
- andamioscan-predicate differential, fork (`v1.0.3-andamio.1`, patched) vs
  stock-v1.2.0: **24/24 Andamio txs byte-identical** on `inputs` +
  `reference_inputs` `as_output` (address, coin, datum, assets); identical
  tx-set (no predicate-match divergence).
- andamioscan ignores the upstream `idle` action (`GetApply()` nil-guard,
  no else branch) — dropping the patch's idle-suppression side effect is safe.

Dropping `fill_input_as_output` also avoids re-porting it across the v1.1.1
`v1alpha`/`v1beta` gRPC-serve split (#987).

---

## `v1.0.3-andamio.1`

Base: [`txpipe/dolos v1.0.3`](https://github.com/txpipe/dolos/releases/tag/v1.0.3)

### `minibf` — optional API `base_path`

Lets `minibf` be mounted behind a reverse proxy at a path prefix (e.g.
`/api/v0`) instead of always at the root.

- `feat(minibf): add optional base_path configuration`
- `fix(minibf): add comprehensive base_path validation`
- `fix(minibf): replace panic with ServeError::ConfigError for invalid base_path`

### `grpc watch` — populate `TxInput.as_output` *(superseded — see v1.2.0-andamio.2)*

Resolved upstream as of v1.1.0; removed on the v1.2.0 rebase.

### `validate` — catch phase-2 panics

- `fix(validate): catch phase-2 panic and skip on failure`
