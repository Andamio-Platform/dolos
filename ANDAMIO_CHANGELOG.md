# Andamio Custom Changelog

This file tracks the Andamio-specific commits layered on top of upstream
[`txpipe/dolos`](https://github.com/txpipe/dolos) releases. Each `andamio.N`
tag is the upstream release SHA + the commits listed under that heading,
rebased cleanly (no conflicts).

## `v1.0.3-andamio.1`

Base: [`txpipe/dolos v1.0.3`](https://github.com/txpipe/dolos/releases/tag/v1.0.3)

### `minibf` — optional API `base_path`

Lets `minibf` be mounted behind a reverse proxy at a path prefix (e.g.
`/api/v0`) instead of always at the root. Needed for the shared-gateway
deployment where multiple services live under one host.

- `feat(minibf): add optional base_path configuration`
- `fix(minibf): add comprehensive base_path validation`
- `fix(minibf): replace panic with ServeError::ConfigError for invalid base_path`

### `grpc watch` — populate `TxInput.as_output`

Upstream [`txpipe/dolos#977`](https://github.com/txpipe/dolos/issues/977):
the `WatchTx` stream returns `TxInput.as_output = nil` for regular inputs,
which breaks any indexer that needs to read a spent UTxO's datum. This
stack resolves both regular and reference inputs.

- `grpc: fill_input_as_output` — hydrate regular inputs from the WAL
  `LogValue.inputs` map (pre-resolved at block-apply time).
- `fix(grpc): hydrate watch as_output via archive fallback and state store for refs`
  - Archive fallback for regular inputs whose source tx is older than WAL
    retention (tx-hash index → block by slot → tx outputs), with per-tx
    output caching.
  - Reference inputs resolved via `domain.state().get_utxos` since they
    are not consumed and therefore not in the WAL inputs map.
  - All failures silently yield `as_output: None` so the stream never
    hard-errors on a missing index entry.

Consumer-side cleanup this unblocks:
[`Andamio-Platform/andamioscan#12`](https://github.com/Andamio-Platform/andamioscan/issues/12).

### `validate` — catch phase-2 panics

- `fix(validate): catch phase-2 panic and skip on failure` — phase-2 script
  evaluation can panic inside pallas on malformed Plutus data; catch the
  panic and skip validation for that tx rather than tearing down the node.
