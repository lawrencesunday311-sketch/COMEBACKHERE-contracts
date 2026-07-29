# compliance-client

Re-exports the auto-generated Soroban cross-contract client for the
`Compliance` contract (`ComplianceContractClient`, aliased as `ComplianceClient`),
so other contracts (e.g. `Treasury`, via `SettlementWorkflow`) can call
`is_allowed` without duplicating the ABI binding boilerplate.

## Cross-contract call failure modes

Soroban cross-contract calls have no network-style timeout or retry semantics —
a call either completes synchronously within the current transaction or the
whole invocation aborts. When calling `is_allowed` through this client, callers
should be aware of:

- **Compliance contract not deployed / wrong contract ID** — the call fails
  immediately and the calling transaction aborts, rolling back any state
  changes made earlier in the same transaction.
- **Compliance contract paused** — `is_allowed` does not check the contract's
  `Paused` flag (only administrative mutations like `allow_address` /
  `block_address` do), so it still returns its normal `bool` result while paused.
- **Compliance contract panics** — the panic propagates through the call
  boundary and aborts the calling transaction, identical to a missing-contract
  failure. There is no automatic retry; a fresh transaction must be resubmitted
  by the off-chain orchestrator after the underlying issue is resolved.

See `ARCHITECTURE.md` at the repo root for the full cross-contract call map and
failure-mode documentation.

## ABI-version coupling risk

`ComplianceClient` wraps the auto-generated `ComplianceContractClient` and
`Deref`s to it directly (see `lib.rs`), so every method this crate exposes —
`is_allowed`, `require_allowed`, `require_allowed_for_treasury`, and anything
reached through the `Deref` — is only as correct as the compiled shape of
`contracts/compliance`'s public interface at build time. There is no runtime
version check: if `Compliance`'s function signatures change (a renamed
method, a reordered/added argument, a changed return type) without a matching
update here, this crate will either fail to compile against the new contract
or, worse, compile against a stale one and call the wrong shape at runtime.

`tests/abi_parity_test.rs` (#53) is the safety net for this: it calls every
wrapped method against a real, freshly-registered `ComplianceContract`
instance, so a signature drift fails to *compile* rather than surfacing as a
runtime cross-contract error in `Treasury` or `SettlementWorkflow`. Any change
to `contracts/compliance`'s public API should be accompanied by running this
test (and updating it) before it's considered safe to merge.
