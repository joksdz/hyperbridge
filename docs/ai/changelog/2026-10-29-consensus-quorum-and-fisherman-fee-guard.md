# BSC quorum ceiling and the fishermen fee guard

Two independent correctness fixes.

## BSC attestation quorum

`verify_bsc_header` accepted `floor(2N/3)` vote-address-set participants. BSC parlia requires
`CeilDiv(len(snap.Validators)*2, 3)`. The two agree only when the validator count is a multiple
of three; otherwise the client accepted an attestation one vote short of what parlia needs to
justify the block, and two conflicting quorums could then overlap in no more than the tolerated
fault count. The threshold now uses `(2 * N).div_ceil(3)`.

BSC mainnet runs a 21-validator cabinet, where floor and ceiling both give 14, so no deployed
configuration was affected. The fix matters if the cabinet size ever stops being a multiple of
three.

## Fishermen call fee guard

`veto_state_commitment`, `blacklist_dispute_game` and `blacklist_arbitrum_claim` dispatch with
`Pays::No`, so `compute_fee_raw` waives the fee during validation, before the dispatch body runs
its `IsCollator` check. `PrioritizeVeto` only raised pool priority for collators and let every
other signer through, so any signed account could submit unlimited fisherman calls that fail with
`UnauthorizedAction` and cost nothing.

`PrioritizeVeto::validate` now rejects a fisherman call whose signer is outside the active
collator set with `InvalidTransaction::Call`, so such extrinsics never reach a block. Collator
calls keep `TransactionPriority::MAX`. The pallet doc comments naming the dispatch-level
`IsCollator` check as the DOS guard were wrong and now point at the extension.

## Tests

`bsc-verifier` gains `requires_parlia_ceiling_quorum` (13 of 20 validators is refused, 14 passes
the quorum gate) and `mainnet_cabinet_quorum_unchanged` (13 and 14 of 21 behave as before). The
pre-existing quorum tests all used 21 validators, where floor and ceiling coincide, which is why
none of them covered this.

`pallet-ismp-testsuite` gains `extension_refuses_fisherman_calls_from_non_collators`, which
asserts the call is annotated `Pays::No`, that a non-collator is refused with
`InvalidTransaction::Call`, and that a collator still validates at `TransactionPriority::MAX`;
and `extension_leaves_other_calls_alone`, which pins the default priority for unrelated calls.
