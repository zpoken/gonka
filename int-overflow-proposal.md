# Proposal: a consistent integer-overflow policy for Gonka

*Responds to [#1222](https://github.com/gonka-ai/gonka/issues/1222).*

## 1. Summary

We propose a written overflow policy, a small shared checked-math package, and a CI check for the four core Go modules. Work starts with inputs users control and with money paths. Code that is already provably safe stays as it is.

A scan of `upgrade-v0.2.16` finds 379 conversion diagnostics (gosec G115), 712 arithmetic candidates on 64-bit integers, and 50 typed calls to SDK or decimal methods that narrow (30 SDK, 20 shopspring). These inventories still need site-by-site classification: some sites are already guarded, but there is no shared way to record which ones and why.

One example: if governance set `InitialEpochReward` near the uint64 maximum, `CalculateFixedEpochReward` would return a reward of zero instead of failing. This is a governance-parameter boundary case; the local test does not establish a public attack or current mainnet impact. It is the kind of silent wrong result the policy is meant to rule out.

The key design point is what happens on overflow. A transaction can simply be rejected. An epoch transition cannot drop a participant, a debt or a validator update. EndBlock already separates recoverable errors from errors that must halt ([module.go:390](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/inference-chain/x/inference/module/module.go#L390)), and the policy keeps that split.

### 1.1 What is planned

- **Rules**: what to do on overflow in each context (transaction, block hook, query, service, client, contract, PoC code), with code examples.
- **A shared package**: `inference-chain/pkg/safemath` for checked arithmetic and conversions, plus adapters for SDK and decimal types.
- **A CI check** in chain, devshard, decentralized-api and common that blocks new unchecked conversions. Existing findings do not block anyone.
- **Testing and upgrades**: anything that changes chain behavior ships in a coordinated upgrade, with tests.

### 1.2 Why it should be done

- The same kind of fix keeps being made one PR at a time: #544 (escrow and cost math), #1100 and #1101 (settlement sums, validation weights). A shared rule makes the next fix routine.
- Existing helpers behave differently. The keeper's `checkedMul` returns `true` on overflow, while devshard's `safeMul` returns `true` on success; saturation helpers use different caps.
- Three of the four Go modules have no lint check in CI.
- There are hundreds of candidate sites, and no agreed way to classify which ones need a fix.
- Point fixes lack a shared reference: #625 was closed so it could be folded into this effort, and #1379 and #1017 are still open and need coordination.
- A guard can itself cause harm: #1062 was withdrawn because it added a new way for transactions to fail. The rules say when a guard is worth it.
- Some failures are silent: decimal `IntPart()` wraps, and the reward example above returns zero instead of an error. A conversion linter cannot see these.

### 1.3 Rationale for the approach

| Decision | Why this choice | Alternative not chosen |
|---|---|---|
| Rules by context | A transaction can be rejected; an epoch transition cannot discard dependent state, so halting or recovery needs a design that keeps invariants. | One rule everywhere (always error, or always saturate). |
| Check new lines first | Stops new problems now without blocking anyone on 379 old findings. | Blocking on the full tree from day one. |
| Small stdlib-only package | All four modules can import it, and off-chain code does not pull in Cosmos. | A large generic library up front. |
| Separate SDK and decimal adapters | These types need their own rounding and nil handling. | Treating them like plain integer casts. |
| Keep behavior unless there is a reason to change it | Even an error-code change is a consensus change. | Bulk rewrites assumed to be harmless. |
| Annotate safe code instead of rewriting it | Less churn, and the reason it is safe is written down (P6). | Wrapping every operation in a helper. |

## 2. Background and existing work

[#1222](https://github.com/gonka-ai/gonka/issues/1222) asks for a standard way to handle overflow, applied consistently, with an automated check. 

| Work | Status | What it means here |
|---|---|---|
| [#544](https://github.com/gonka-ai/gonka/pull/544) | Merged (v0.2.8): checked escrow and cost math, token bounds | The pattern to follow |
| [#535](https://github.com/gonka-ai/gonka/pull/535) | Decimal-parser panic fixed | Library panics need explicit handling |
| v0.2.15 decimal-exponent fix (`MaxDecimalExponentAbs = 18`, `Decimal.Validate()`) | Released: an unbounded decimal exponent could stall the chain | Main precedent for P7. `ToDecimal()` itself is still unchecked (54 callers), so each caller must remember `Validate()` |
| [#1100](https://github.com/gonka-ai/gonka/pull/1100), [#1101](https://github.com/gonka-ai/gonka/pull/1101), [#1267](https://github.com/gonka-ai/gonka/pull/1267) | Merged (v0.2.14) | Already done; do not redo |
| [#1379](https://github.com/gonka-ai/gonka/pull/1379) | Open; author preparing a smaller revision | Claim-path casts; coordinate with the author |
| [#1017](https://github.com/gonka-ai/gonka/pull/1017) | Open | Supply-cap economics; handle separately from helper adoption |
| [#625](https://github.com/gonka-ai/gonka/pull/625), [#1014](https://github.com/gonka-ai/gonka/pull/1014), [#1015](https://github.com/gonka-ai/gonka/pull/1015), [#884](https://github.com/gonka-ai/gonka/pull/884) | Closed: folded into this effort, already covered, or redesigned | Respect each closure reason |
| [#1062](https://github.com/gonka-ai/gonka/pull/1062) | Withdrawn: the guard would add a new way for transactions to fail | Each new guard needs a reason, or a documented bound instead |
| [#1671](https://github.com/gonka-ai/gonka/issues/1671), [#1293](https://github.com/gonka-ai/gonka/pull/1293), [#1858](https://github.com/gonka-ai/gonka/pull/1858) | Open: SDK fork rebase, retry cap for BLS cleanup, Wasm artifact sync | Coordinate ordering with their owners |

## 3. Current state

### 3.1 Tooling

The chain's lint config enables `forbidigo` (it bans `panic` and `Must*`) and the default `govet`. Banning names does not catch library methods that panic internally. The `dont_panic.yml` workflow pins golangci-lint `v2.6` and runs on chain PRs. Devshard, decentralized-api and common have no lint check in CI. [Config](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/inference-chain/.golangci.yml), [workflow](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/.github/workflows/dont_panic.yml).

### 3.2 Helper inventory

Paths are relative to `inference-chain/` unless another module is named. Similar names do not mean the same behavior:

| Helper / source | Current contract |
|---|---|
| [`keeper/safecast.go`](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/inference-chain/x/inference/keeper/safecast.go#L11): `safeInt32FromInt64`, `safeUint32FromInt64`, `safeUint64FromInt64`, `safeUint32FromUint64`, `safeInt32FromUint64`, `safeUint8FromUint32` | Six `(value,error)` conversions |
| Same file: `clampInt32FromInt` | Clamp to int32; optional warning callback; intended for cosmetic queries |
| [`keeper/bitcoin_rewards.go:145–168`](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/inference-chain/x/inference/keeper/bitcoin_rewards.go#L145) | `saturatingAddUint64Max(int64,uint64)` caps at MaxInt64 and assumes nonnegative first input; `positiveUint64` floors nonpositive values to zero; `addUint64Saturating` caps at MaxUint64 |
| [`keeper/msg_server_poc_v2_commit.go:261`](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/inference-chain/x/inference/keeper/msg_server_poc_v2_commit.go#L261): `checkedMul` | **Boolean means overflow**, not success |
| [`module/delegation_pipeline.go:192,939`](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/inference-chain/x/inference/module/delegation_pipeline.go#L192) | `checkedRawWeightAdd` rejects negative added values/upper overflow; `sumInt64Safe` sums sorted nonnegative values and returns success boolean |
| [`devshard/state/machine.go:21–50`](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/devshard/state/machine.go#L21) | `safeAdd`/`safeMul`: boolean means success; `tokenCost` preserves `ErrCostOverflow` |
| [`decentralized-api/cosmosclient/tx_manager/gas_estimate.go:371`](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/decentralized-api/cosmosclient/tx_manager/gas_estimate.go#L371) | `saturatingAdd`/`saturatingMul`: MaxUint64 saturation for estimates |
| [`calculations/inference_state.go`](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/inference-chain/x/inference/calculations/inference_state.go#L18), [`types/settle_amount.go`](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/inference-chain/x/inference/types/settle_amount.go#L8) | Checked cost/escrow boundaries and capped aggregate; these domain contracts must survive refactoring |

Another 11 files do the same checks inline with `bits.Add64`/`Mul64`. Correct inline code can stay.

### 3.3 Scan results

| Module, v0.2.16 tree | G115 v2.6.0 | G115 v2.14.0 | int64/uint64 arithmetic candidates | Money-name subset |
|---|---:|---:|---:|---:|
| inference-chain | 173 | 154 | 286 | 129 |
| devshard | 147 | 129 | 302 | 37 |
| decentralized-api | 94 | 74 | 103 | 20 |
| common | 24 | 22 | 21 | 4 |
| **Total** | **438** | **379** | **712** | **190** |

Scanned on 2026-09-30 at `upgrade-v0.2.16` (`beb159be5`) with golangci-lint v2.6.0 and v2.14.0 and a small type-aware arithmetic scanner. "Money-name" means the expression mentions coins, amounts, rewards, weights or similar. The devshard v6 line (`f3464409e`) gives similar numbers (149 / 308 / 39) and is not included in the totals. The `versioned` and `edge-api` modules have no G115 findings.

### 3.4 Interpretation and priority

These numbers are a to-do list, not a bug count. G115 flags conversions only and has false positives. The arithmetic scanner lists `+ - *` on 64-bit integers without checking whether a guard already protects them; it skips machine `int`, `++/--`, shifts and division.

The 30 SDK calls (`Int.Int64`, `Int.Uint64`, `LegacyDec.TruncateInt64`, `LegacyDec.RoundInt64`) panic when the value does not fit. 18 are in `x/` and 12 in `app/`, mostly old upgrade handlers.

The 20 shopspring calls (`IntPart`, `CoefficientInt64`), all in `x/inference/`, do not panic. They silently wrap instead, which G115 cannot see. Decimal division also rounds before truncation, so `Div().IntPart()` can differ from exact integer division.

Priority goes to values users control and to money paths: claim payouts, reward calculation, and the transfer-restriction exemption that sums `coin.Amount.Uint64()` across denominations. [Restrictions code](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/inference-chain/x/restrictions/keeper/send_restriction.go#L128).

### 3.5 SDK fork and validator power

Gonka's cosmos-sdk fork adds 71 commits to v0.53.3, touching 40 Go files (+2,415/−1,415).

The fork sets `DefaultPowerReduction` to 1, so PoC weight becomes CometBFT voting power directly. CometBFT caps total voting power at MaxInt64/8 (about 1.15×10¹⁸) and rejects a validator set above it. Nothing in the chain or the fork checks this cap before the update. [SDK reduction](https://github.com/gonka-ai/cosmos-sdk/blob/3d4749057f020b12ff0208c2976e9e1545e43eb0/types/staking.go#L24), [staking entry point](https://github.com/gonka-ai/cosmos-sdk/blob/3d4749057f020b12ff0208c2976e9e1545e43eb0/x/staking/keeper/compute.go#L101), [CometBFT bound](https://github.com/cometbft/cometbft/blob/v0.38.21/types/validator_set.go#L20).

On mainnet (2026-09-30) 23 validators hold 642,120 total power, roughly 10¹² times below the cap. This is hardening, not an urgent issue. [node1 GET](http://node1.gonka.ai:8000/chain-api/cosmos/base/tendermint/v1beta1/validatorsets/latest?pagination.limit=1000), [node2 GET](http://node2.gonka.ai:8000/chain-api/cosmos/base/tendermint/v1beta1/validatorsets/latest?pagination.limit=1000).

The check should run on the final validator set (after filtering, deduplication and guardian adjustments), before anything is written. If the cap would be exceeded, the epoch transition returns an error and does not fall back to the previous validator set. Today a `SetComputeValidators` error is only logged, so this is a deliberate consensus change, not existing behavior. The call site, which today marks the epoch group unchanged even when the update fails, is fixed in the same change. Validator changes take effect at H+2, so tests cover that window. [Call site](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/inference-chain/x/inference/module/module.go#L585), [activation ordering](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/inference-chain/x/inference/module/module.go#L482).

The fork also lacks the upstream fix for `offset + limit` wrapping in query pagination ([cosmos-sdk#26430](https://github.com/cosmos/cosmos-sdk/pull/26430), merged only on upstream `main`), as well as several bounds fixes from v0.53.4–v0.53.8. Both belong with the fork rebase in [#1671](https://github.com/gonka-ai/gonka/issues/1671).

Separately, `val.Tokens == power` in `compute.go` compares `math.Int` pointers, so it is never true. Fixing it with `.Equal()` changes which writes and hooks run, so it is a consensus change and needs its own tests. [Comparison and update path](https://github.com/gonka-ai/cosmos-sdk/blob/3d4749057f020b12ff0208c2976e9e1545e43eb0/x/staking/keeper/compute.go#L168).

### 3.6 Other languages and boundaries

| Area | Finding and proposed action |
|---|---|
| Rust contracts | All three contracts build with `overflow-checks = true`. Clippy reports 2, 11 and 0 arithmetic/cast warnings. In `liquidity-pool`, a failed multiplication falls back to the old price and still divides by 1000, so the price drops about 1000× instead of failing (see P11). Deployed code must be matched to source first (#1858). [Profiles/pricing](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/inference-chain/contracts/liquidity-pool/src/state.rs#L88), [artifact work](https://github.com/gonka-ai/gonka/pull/1858) |
| Solidity | `^0.8.19` with no `unchecked` blocks. Arithmetic is checked, but casts are not; the existing casts are bounded and only need annotations (see P12). [Source](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/proposals/ethereum-bridge-contact/contracts/BridgeContract.sol#L675) |
| PoC Python/torch | The hash multiplies two 32-bit values in an int64 tensor, which wraps before the 32-bit mask. Results match a big-integer reference on CPU; CUDA was not tested. Add CPU and CUDA golden vectors before changing anything (see P13). [Plugin](https://github.com/gonka-ai/gonka-vllm-plugins/blob/cb382f58b0215a1d8a6264dac469ee7f3b248d45/src/gonka_poc/poc/gpu_random.py#L30), [legacy input](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/mlnode/packages/pow/src/pow/random.py#L18) |
| mlnode → DAPI | Line 97 converts an int64 nonce with a bare `int32(...)`. Validate the whole batch before storing it (see P9). [Handler](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/decentralized-api/internal/server/mlnode/post_generated_artifacts_v2_handler.go#L82) |
| Explorer/docs | Amounts are converted with `Number`/`parseFloat` and lose precision above 2⁵³. Keep exact values as strings/BigInt and round only for display (see P10). [gonkascan](https://github.com/gonka-ai/gonkascan/blob/bf4516c0d52da99aaf7460f0bc3287ff22ebd711/frontend/src/utils.tsx#L38), [docs sample](https://github.com/gonka-ai/gonka-docs/blob/c9ee34572b11bf9f9a80bec27f157ec285a46a5d/docs/cross-chain-transfers/widget-integration.md#L893) |

Other repositories in the organization were only looked at briefly and are not part of this proposal.

## 4. Policy by execution context

Every arithmetic operation, narrowing or sign change either uses a checked helper or has a documented bound that is enforced somewhere. What happens on overflow depends on where the code runs and what the value means:

| ID | Context | Required behavior |
|---|---|---|
| P1 | Transaction input and economic state changes | Reject out-of-range values with the existing domain error, before any write. Check in the handler too, not only in `ValidateBasic` or the API (nested messages skip some checks). Never silently cap a payout or debt. |
| P2 | BeginBlock, EndBlock, epoch transitions | Invariant-critical failures halt; making a path halt that today only logs is a consensus change. Recoverable steps need an explicit, deterministic retry (next block for per-block steps, a recorded pending-work marker for epoch-triggered steps). Never turn an overflow into a silent skip. |
| P3 | Explicit semantic caps | Saturate only where the protocol defines the cap. Name it, and say whether it is MaxInt64 or MaxUint64. |
| P4 | Wire, storage and SDK boundaries | Checked conversion against the target width and the protocol's allowed range. Handle nil SDK values. Use a wider intermediate where `(a*b)/d` would overflow but the result fits. |
| P5 | Queries | Return exact values for money, consensus data, proofs and cursors. Clamp only cosmetic fields. |
| P6 | Proven bounded operations | Plain Go is fine when a comment names the bound and where it is enforced. |
| P7 | Library operations | Call SDK and decimal narrowing through checked adapters that keep the original rounding. A direct call is allowed only behind a guard, with a suppression that names it (P6). |
| P8 | Devshard replicated state | Changes to what the state machine accepts or hashes are protocol changes and need a version bump. |
| P9 | Off-chain input and queues | Validate values from the network, database or config before use; return 4xx on bad input. Retry only failures that can succeed later. |
| P10 | Clients | Carry chain integers as strings and compute with BigInt or a decimal library. Round only for display. |
| P11 | Rust | Keep `overflow-checks = true`. Propagate checked-math errors with `?`; no `unwrap_or` fallback on arithmetic unless the protocol defines it (P3). |
| P12 | Solidity | Pragma ≥ 0.8. Every `unchecked` block and narrowing cast gets a bound comment or `SafeCast`. |
| P13 | Consensus-related Python | Explicit dtypes and byte order; document intended wraparound and pin it with CPU and CUDA golden vectors. |

The same rules apply to division by zero, `MinInt / -1`, negating `MinInt`, shifts, counters, height/time math and allocation sizes. Stored numeric types and the replay of old blocks do not change.

When a money path is migrated, write down in a short comment or doc: units, the allowed range, the widths of intermediate results, which way it rounds and where the remainder goes, and what happens on failure. Bounds must hold for any value governance can set, not just the defaults; today `SetParams` does not validate on its own. [Setter](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/inference-chain/x/inference/keeper/params.go#L49).

Order matters too. A checked sum over mixed-sign values can succeed in one order and fail in another, and Go map iteration order is random. Sum in a fixed order (or in a wide type, checking once at the end), and test permutations.


### 4.1 Rules in practice

Each rule below has the proposal, how to apply it, and an example from `upgrade-v0.2.16`. `safemath` is the shared helper package described below. Snippets are illustrations, not final code.

#### P1: Transaction inputs and economic state changes

**Proposal.** A message handler rejects an out-of-range value with an error before it makes any irreversible change. It never silently caps or wraps a payout, debt or balance.

**How.**
1. Validate input ranges in `ValidateBasic` (cheap, stateless) **and** again in the handler wherever the value is combined with state (sums, products, conversions).
2. Do every conversion or arithmetic on an economic value through `safemath`, or document the enforced bound next to it (P6), and return the module's existing domain error (e.g. `ErrArithmeticOverflow`, `ErrTokenCountOutOfRange`). Do not invent a new error code during a refactor.
3. Do all checks before the first write. Where writes are unavoidable, use the existing `CacheContext` pattern so a failed message leaves no partial state.

**Example.** `msg_server_claim_rewards.go:90,110` converts payouts with bare casts:

```go
// today
ms.PayParticipantFromEscrow(cacheCtx, payoutAddress, int64(escrowPayment), ...)

// proposed
amt, err := safemath.Int64FromUint64(escrowPayment)
if err != nil {
    return nil, errorsmod.Wrapf(types.ErrArithmeticOverflow, "escrow payment %d", escrowPayment)
}
ms.PayParticipantFromEscrow(cacheCtx, payoutAddress, amt, ...)
```

`AddToCoinBalance` (`account_helpers.go:21`) already follows this rule and is the reference pattern. This is the remaining scope of #1379.

#### P2: BeginBlock, EndBlock and epoch transitions

**Proposal.** Keep the existing split between recoverable and unrecoverable failures (`module.go:390`). An overflow never turns into an automatic skip of a participant, debt or validator update.

**How.** Classify every block-hook unit that does checked math into one of two classes:
- **Invariant-critical** (epoch state, validator set, settlement): on overflow, return the error. Some epoch-state failures already halt; others (e.g. a `SetComputeValidators` error) are only logged today, so making them halt is a consensus change and is listed as one.
- **Recoverable** (pricing cache, statistics, capacity cache; the `// Don't return error` sites at `module.go:218,417,1035`): run the unit in a `CacheContext`; on failure, discard the cache, log at Error, and emit an event **outside** the discarded cache. Then retry deterministically:
  - a step that runs every block simply runs again next block;
  - a step that runs only at one epoch height (e.g. `CacheAllModelCapacities`, which runs only when `blockHeight == SetNewValidators()`) is not called again on its own, so the failure must write a pending-work marker that a later block picks up, with defined behavior if the epoch has moved on.

A unit may move from invariant-critical to recoverable only with a written recovery design and multi-block failure tests. A discarded cache still keeps the gas it consumed.

**Example.** `module/power_capping.go:45,57` sums weights with plain `+=` during the epoch transition. Proposed: accumulate with `safemath.AddInt64`. On overflow, return an error, because validator weights are invariant-critical. Do not skip the participant.

#### P3: Explicit semantic caps (saturation)

**Proposal.** Saturate only where the protocol itself defines a cap and its accounting consequence. The cap is named and documented, never a generic fallback.

**How.** Keep saturation helpers next to the domain they serve, with a comment naming the cap, the bound type (MaxInt64 vs MaxUint64) and what happens to the excess. Do not replace them with a generic `SaturatingAdd`.

**Example.** `bitcoin_rewards.go:145` `saturatingAddUint64Max(a int64, b uint64)` caps at **MaxInt64** and assumes `a ≥ 0`. It stays a named domain helper, and the comment records the precondition. `addUint64Saturating` in the same file caps at **MaxUint64**: a different cap, so a different helper.

#### P4: Wire, storage and SDK boundaries

**Proposal.** Every narrowing or sign change at a proto field, store value or SDK type is a checked conversion against the destination's width **and** the protocol's domain.

**How.**
1. Replace the six unexported `safeXFromY` helpers in `keeper/safecast.go` with `safemath` conversions. Keep their errors identical while doing so.
2. Where a domain is narrower than the type (e.g. a nonce range), check the domain, not only the type.
3. For decimals, specify units, scale and rounding before converting.

**Example.** `keeper/safecast.go` already returns errors; it becomes a thin wrapper over `safemath` and is later removed. New boundary code calls `safemath` directly.

#### P5: Queries

**Proposal.** Queries return exact values for anything financial, consensus-relevant, a proof or a cursor. Clamping is allowed only for cosmetic fields, with a documented API contract.

**How.** Use a checked adapter and return a gRPC error (`codes.OutOfRange`) instead of panicking or truncating. If a field type is too narrow for real values, change the field in a versioned API rather than clamping it.

**Example.** `query_inference_participant.go:46` returns `Balance: balance.Amount.Int64()`, which panics if a balance exceeds int64:

```go
// proposed
bal, err := sdkbridge.IntToInt64(balance.Amount)
if err != nil {
    return nil, status.Error(codes.OutOfRange, "balance exceeds int64")
}
```

`clampInt32FromInt` stays for its documented cosmetic use only.

#### P6: Proven bounded operations

**Proposal.** Plain Go arithmetic or casts are fine where a bound is **enforced** elsewhere. The bound must be written next to the code, with where it is enforced.

**How.** Use a lint suppression that names the rule and the bound. Reviewers check that the cited enforcement exists. Re-check the proof whenever the parameter, caller or width changes.

**Example.** `x/bls/keeper/msg_server_verifier.go:213` casts `uint32(len(epochBLSData.Participants))`. In practice the count is tiny (mainnet has 23 validators), but we found no check that *enforces* a maximum: DKG initiation only rejects an empty list, and `ITotalSlots` limits slots, not participants. Under P6, "small in practice" is not enough. The fix is to add one explicit bound where the participant list is built (e.g. reject more than `math.MaxUint32` participants, or a protocol limit), then cite it:

```go
//nolint:gosec // G115: len(Participants) <= MaxDKGParticipants, enforced in InitiateDKG.
if proof.DealerIndex >= uint32(len(epochBLSData.Participants)) {
```

`x/bls/keeper` has 14 G115 diagnostics (golangci-lint v2.14.0; 26 with v2.6). Those on participant-list lengths can share this one bound; the rest (heights, slot counts, key and signature indexes) need their own classification.

#### P7: Library operations (SDK and shopspring decimal)

**Proposal.** Do not call a library method that panics or silently wraps on narrowing without a check. Use a checked adapter that preserves the method's semantics; a direct call is allowed only behind a guard, with a suppression that names it (P6).

**How.**
1. Add `sdkbridge.IntToInt64`, `IntToUint64`, `DecTruncateInt64` and `DecRoundInt64` (SDK), and, for shopspring, `DecimalToInt64` / `DecimalToUint64` (integer part) plus a separate `CoefficientToInt64` (coefficient with its exponent unchanged), all returning errors.
2. The `forbidigo` lint rule (see Enforcement) flags direct calls to `Int.Int64`, `Int.Uint64`, `LegacyDec.TruncateInt64`, `LegacyDec.RoundInt64`, `Decimal.IntPart` and `Decimal.CoefficientInt64`.
3. Make the conversion chokepoint itself checked: replace `Decimal.ToDecimal()` (54 production callers today) with a variant that validates the exponent and returns an error, so the v0.2.15 bound no longer depends on each caller calling `Validate()` first.
4. Audit the precision of a division **before** truncation separately. shopspring's `Div` rounds to 16 places first, so `Div().IntPart()` can differ from the exact integer quotient.

**Example.** `bitcoin_rewards.go:306` ends `CalculateFixedEpochReward` with:

```go
// today: IntPart() wraps above MaxInt64, and the result becomes 0
result := currentReward.IntPart()
if result < 0 { return 0, nil }
return uint64(result), nil

// proposed
r, err := decimaladapter.DecimalToUint64(currentReward) // truncates, checks range
if err != nil { return 0, err }
return r, nil
```

Pair this with a domain bound on `InitialEpochReward` in parameter validation, which today checks only the type.

#### P8: Devshard replicated state

**Proposal.** The devshard state machine (`devshard/state`, `devshard/types`) is replicated and signed, so it follows consensus rules: any change to what it accepts, stores or hashes is a protocol change.

**How.**
1. Inside apply and settle paths, use `safemath` with typed errors, as `tokenCost` already does with `ErrCostOverflow`.
2. Ship any change in accepted inputs or failure behavior behind a devshard protocol version, and test old/new hosts against each other.
3. Rejection must never break a signed commitment or a settlement obligation.

**Example.** `devshard/state/machine.go:572` does `Balance -= FeePerNonce` right after an explicit `Balance < FeePerNonce` check, which is correct under P6 and only needs the annotation. `safeAdd`/`safeMul` in the same file return `true` on **success**, the opposite of the keeper's `checkedMul`; migrating either to `safemath` must preserve each caller's meaning, with tests.

#### P9: Off-chain inputs and queues

**Proposal.** Services (DAPI, devshard host/gateway, common) validate every value that comes from the network, a database or config before narrowing or storing it. They distinguish **permanently invalid input** from **transient failure**.

**How.**
1. On ingress, check width and protocol domain; reject malformed caller input with a typed error or a 4xx.
2. Validate a whole batch before writing any of it.
3. In retry loops, retry only failures that can succeed later (network, timeout, state-dependent). Record permanently invalid items durably with a reason, rather than retrying them forever or dropping them silently.

**Example.** `decentralized-api/internal/server/mlnode/post_generated_artifacts_v2_handler.go:97` narrows the mlnode nonce with `int32(a.Nonce)`, where the JSON field is `int64`:

```go
// proposed: validate the whole batch first
for _, a := range body.Artifacts {
    if _, err := safemath.Int32FromInt64(a.Nonce); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, "nonce out of range")
    }
}
```

The later cast at line 229 is already `int32` and needs no change.

#### P10: Clients (explorers, SDKs, docs, scripts)

**Proposal.** Chain integers (amounts, heights, nonces, weights) travel as decimal strings and are computed with exact integer or decimal types. Rounding happens only at display.

**How.**
1. JS/TS: parse with `BigInt(str)`. Use `Number` only for values proven below 2⁵³, or for displays explicitly marked approximate.
2. Convert ngonka to GNK at render time with BigInt or string math.
3. Backends that serve JSON to browsers emit u64 values as strings.
4. Never build a transaction from a displayed or float value.

**Example.** `gonkascan/frontend/src/utils.tsx:38`:

```ts
// today: loses precision above 2^53 ngonka
export function toGonka(amount: string | number): number {
  return Number(amount) / 1e9
}

// proposed: exact; returns a display string (callers updated)
export function toGonka(amount: string): string {
  const v = BigInt(amount);
  const frac = (v % 1_000_000_000n).toString().padStart(9, "0");
  return `${v / 1_000_000_000n}.${frac}`;
}
```

The docs widget sample (`cross-chain-transfers/widget-integration.md:893`, `parseFloat(balance.amount)`) gets the same fix, because integrators copy it. gonkascan's `isBlockHeight` (`Number.isSafeInteger`) is an example of correct bounded `Number` use.

#### P11: Rust (CosmWasm contracts)

**Proposal.** Keep `overflow-checks = true` in release profiles (already true for all three contracts). A failed checked operation propagates as a `ContractError`, unless a reviewed domain rule defines a fallback.

**How.**
1. Write `checked_*()?` mapped to a contract error.
2. Do not use `unwrap_or(default)` on a checked arithmetic result unless the fallback is a protocol-defined rule (P3), documented next to it. Defaults for an absent `Option` (e.g. a missing balance) remain fine.
3. Use `try_from` for narrowing instead of saturating helpers.
4. CI runs Clippy with `arithmetic_side_effects` and the `cast_*` lints.

**Example.** `liquidity-pool/src/state.rs:88`:

```rust
// today: on overflow, keeps the old price and still divides by 1000,
// so the price drops ~1000x instead of failing
price = price.checked_mul(tier_multiplier).unwrap_or(price)
             .checked_div(Uint256::from(1000u128)).unwrap_or(price);

// proposed
price = price.checked_mul(tier_multiplier)?
             .checked_div(Uint256::from(1000u128))?;
```

The function's return type becomes `Result<Uint256, ContractError>`. Contract changes ship as a contract migration, with the deployed code hash matched to source (#1858).

#### P12: Solidity

**Proposal.** Keep pragma ≥ 0.8 (checked arithmetic by default). Every `unchecked {}` block and every narrowing cast carries a justification, because 0.8 does **not** check casts.

**How.** Use OpenZeppelin `SafeCast` for narrowing where representability is not obvious; otherwise add a comment with the bound. Optionally enable the SMTChecker's overflow targets in CI.

**Example.** `BridgeContract.sol` has `uint64(block.timestamp)` (safe for ~585 billion years; annotate) and `uint8` casts in the byte-wise subtraction with borrow that computes `p − y` for G1 negation (lines 675–682). Each result is in [0, 255] by construction (`pi ≥ subtrahend`, or `pi + 256 − subtrahend` with `subtrahend ≤ 256`); annotate that bound next to the casts.

#### P13: Consensus-related Python (PoC compute, validation)

**Proposal.** Any integer tensor that feeds consensus has an explicit dtype, byte order and size bound. Any intended wraparound is documented and pinned by tests, not left to chance.

**How.**
1. Declare dtypes explicitly; use explicit little-endian decoding (`'<u4'`) instead of native order.
2. Assert shape bounds before `int32` `arange`, and mirror them as upper bounds in chain parameters (e.g. `SeqLen`, which today only checks `≥ 0`).
3. Pin seeded **CPU and CUDA** golden vectors and rerun them on every torch/numpy upgrade.

**Example.** `gonka-vllm-plugins/.../poc/gpu_random.py:37` multiplies two values below 2³² in an `int64` tensor; the product (up to ~1.47×10¹⁹) wraps before the 32-bit mask. Tested CPU samples match the modulo-2³² reference; CUDA and other device or version combinations are not yet verified, which is what the golden vectors are for. The proposal: document the intended modulo-2³² behavior, add CPU+CUDA golden vectors, and only then consider rewriting the arithmetic (e.g. splitting into 16-bit halves so it never exceeds int64).

## 5. Shared helper contract

Use `inference-chain/pkg/safemath`, import path `github.com/productscience/inference/pkg/safemath`, as a stdlib-only leaf. All three other core modules already require/replace the chain module. Put SDK adapters in a separate package, e.g. `pkg/safemath/sdkbridge`; the leaf must not import that package or Cosmos modules.

Start with only the operations and widths that callers need:

```go
// Error means no usable result; return zero on failure.
var ErrOverflow = errors.New("integer overflow")
func AddUint64(a, b uint64) (uint64, error)
func SubUint64(a, b uint64) (uint64, error)
func MulUint64(a, b uint64) (uint64, error)
func AddInt64(a, b int64) (int64, error)
func SubInt64(a, b int64) (int64, error)
func MulInt64(a, b int64) (int64, error)
// Checked conversions for the inventoried width/sign pairs.
func Int64FromUint64(v uint64) (int64, error)
func Int32FromInt64(v int64) (int32, error)
// Add other pairs as migration needs them, without unchecked intermediate casts.
```

Saturation helpers stay in their domain code (P3). When an existing helper is switched to `safemath`, its error codes, boolean meaning, caps and logs must not change; for example, token-count overflow returns `ErrTokenCountOutOfRange`, not `ErrArithmeticOverflow`. [Existing errors](https://github.com/gonka-ai/gonka/blob/beb159be59e1b980f63e1caffcb51c89fce7bc23/inference-chain/x/inference/calculations/inference_state.go#L39).

The SDK adapter package provides `IntToInt64`, `IntToUint64`, `DecTruncateInt64` and `DecRoundInt64`, which return an error instead of panicking. Truncation and banker's rounding stay distinct. [SDK implementations](https://github.com/cosmos/cosmos-sdk/blob/math/v1.5.3/math/legacy_dec.go#L687).

The decimal adapter package checks the exponent first, takes the integer part as a `big.Int`, and then checks it against the target type, so nothing is narrowed before the check. Coefficient conversion is a separate function: it reads `Coefficient()`, checks its range, and returns it with the original exponent. It must not go through the integer part: 1.23 has coefficient 123 and exponent −2, but integer part 1, so mixing the two would turn 1.23 into 0.01. Round-trip tests cover fractional values and positive and negative exponents. Code that relies on `Div().IntPart()` rounding keeps its current result unless a change is agreed.

For `(a*b)/d`, a checked 64-bit multiply can fail even when the result fits (e.g. `(MaxUint64*2)/2`). Such callers use a wide intermediate. We add a shared `MulDiv` only once a real caller needs it.

We write `safemath` ourselves rather than adopting [go-safecast](https://github.com/ccoveille/go-safecast): that library covers conversions only, not arithmetic, and would add an external dependency to consensus code for something that fits in a small package. SDK adapters live in `pkg/safemath/sdkbridge`, decimal adapters in `pkg/safemath/decimaladapter`.

## 6. Enforcement

### Initial blocking checks

The existing `panic`/`Must`/`govet` job stays unchanged. The overflow check is a separate config, so that its new-lines-only mode does not weaken the existing job.

The configs below were tested on fixtures with golangci-lint v2.14.0 (the SDK rules also with v2.6.0). v2.14.0 needs Go 1.26 for the CI linter job only; node builds are unaffected. Chain config:

```yaml
version: "2"
run:
  timeout: 15m
  tests: false
linters:
  default: none
  enable: [gosec, nolintlint, forbidigo]
  settings:
    gosec:
      includes: [G115]
    nolintlint:
      require-explanation: true
      require-specific: true
    forbidigo:
      analyze-types: true
      forbid:
        - pattern: '^math\.Int\.(Int64|Uint64)$'
          pkg: '^cosmossdk\.io/math$'
          msg: Use a checked adapter or an explained range guard.
        - pattern: '^math\.LegacyDec\.(TruncateInt64|RoundInt64)$'
          pkg: '^cosmossdk\.io/math$'
          msg: Use a checked adapter preserving rounding semantics.
        - pattern: '^decimal\.Decimal\.(IntPart|CoefficientInt64)$'
          pkg: '^github\.com/shopspring/decimal$'
          msg: Use a checked adapter preserving scale and rounding.
  exclusions:
    generated: strict
    paths: ['^testutil/', '^tests?/', '^cmd/', '\.pb\.go$', '\.pb\.gw\.go$']
issues:
  max-issues-per-linter: 0
  max-same-issues: 0
```

Devshard, decentralized-api and common start with conversions only:

```yaml
version: "2"
run:
  timeout: 15m
  tests: false
linters:
  default: none
  enable: [gosec, nolintlint]
  settings:
    gosec:
      includes: [G115]
    nolintlint:
      require-explanation: true
      require-specific: true
  exclusions:
    generated: strict
    paths: ['^testutil/', '^tests?/', '^cmd/', '\.pb\.go$', '\.pb\.gw\.go$']
issues:
  max-issues-per-linter: 0
  max-same-issues: 0
```

Each config lives at its module root. The CI job checks out full history and compares against the PR's base branch:

```sh
# Repository root; PR jobs only. PR_BASE is the target branch name.
git check-ref-format "refs/heads/$PR_BASE"
git fetch --no-tags origin "+refs/heads/$PR_BASE:refs/remotes/origin/$PR_BASE"
# In each module, using that module's overflow config:
golangci-lint config verify --config .golangci-overflow.yml
golangci-lint run --config .golangci-overflow.yml \
  --new-from-merge-base="origin/$PR_BASE"
```

PRs often target `upgrade-v*` branches, so the base must come from the PR, not be hardcoded to `main`. A scheduled full run (without the new-lines filter) tracks the remaining backlog. Initially the job runs all four modules on every relevant PR.

Limitations:

- The check only looks at changed lines. If a PR changes a bound elsewhere, an old cast can become unsafe without being flagged; reviewers still need to catch that.
- `nolintlint` only checks that a suppression names the linter and has a reason. Whether the reason is true is up to review. Expected format: `//nolint:gosec // G115: value <= 10000, enforced by ValidateX.`
- `forbidigo` flags the six listed methods (also under import aliases) but cannot see a guard; a guarded direct call needs a suppression with a reason. It does not cover other library panics.
- Test files are excluded from the lint, but the tests themselves still run in CI.

### Advisory checks and other repos

Unchecked arithmetic (`+ - *`) is not covered by any existing linter. We start with a non-blocking report for the money and weight packages. After one full release cycle, if no more than 10% of its findings there are false positives and it runs in under 2 minutes, a separate PR proposes making it blocking.

AI review (the existing `ai-reviewer` personas and Copilot instructions) can apply this policy as a reviewer aid, not as a merge gate. Lint rules for Rust, Solidity, Python and JS follow later, each with its own scope.

For the cosmos-sdk fork, lint only Gonka's own changes (`new-from-rev` set to the upstream tag the fork is based on) and update that tag when #1671 rebases. Upstream cosmos-sdk and CometBFT disable G115; limiting the gate to Gonka's changes keeps it focused, and upstream code is covered by dependency review during the rebase. [SDK config](https://github.com/cosmos/cosmos-sdk/blob/main/.golangci.yml), [CometBFT config](https://github.com/cometbft/cometbft/blob/v0.38.21/.golangci.yml#L90).

## 7. Compatibility, testing, recovery and alternatives

A change is a consensus change if it can alter which transactions succeed, what is stored, gas used, validator updates or hooks, even only for extreme inputs. CometBFT hashes each transaction's result code and gas, so a changed error code or gas amount alone changes consensus. [Result encoding](https://github.com/cometbft/cometbft/blob/v0.38.21/types/results.go#L47). Such changes ship in a coordinated chain upgrade; devshard and PoC changes follow their own versioning.

Testing, scaled to each change:

1. **Helpers:** compare against `big.Int` at the boundaries (0, min, max, sign changes, `MinInt × −1`), test nil SDK values and both rounding modes, and fuzz.
2. **Money:** debits equal credits, supply caps hold, nothing is paid twice or lost, and bad input is rejected before any partial write. Check rounding remainders against exact math.
3. **Block hooks:** inject failures at each write, over several blocks, including the H+2 validator activation; check that a failed step is retried and that state stays consistent.
4. **Upgrade:** replay real blocks with old and new binaries and compare app hashes, result hashes and timing; rehearse the upgrade on a mainnet snapshot; check that existing state already satisfies the new bounds. [SDK upgrade workflow](https://docs.cosmos.network/sdk/v0.53/build/building-apps/app-upgrade).
5. **Sequences:** random but seeded sequences of parameter changes, claims and epoch transitions, checking the money invariants after each step (the Cosmos [simulation framework](https://docs.cosmos.network/sdk/v0.53/learn/advanced/simulation) supports this).
6. **Other languages:** deployed contract hashes match source; PoC golden vectors pass on CPU and CUDA; explorer values round-trip exactly.

**If something goes wrong:** a bad CI rule can simply be disabled. A problem found after a chain upgrade is fixed with a follow-up upgrade; validators should not roll back to the old binary on their own.

| Alternative | Assessment |
|---|---|
| Bounds plus review only | Good where a bound is simple; still needs an automated check |
| Full-tree G115 now | 379 findings at once is too much to review |
| Fuzzing or AI review instead of lint | Useful additions, but neither enforces a convention on every PR |
| go-safecast for conversions | Not adopted: conversions only, and an extra dependency in consensus code |
| `sdkmath.Int` / `big.Int` everywhere | Too broad; use them only as intermediates where needed |
| Runtime overflow detection (e.g. go-panikint) | Optional, for test runs only |

## 8. Expected outcomes and decisions

Success is consistent overflow behavior across the codebase, not just fewer lint findings. We track: findings fixed or annotated per release, suppressions without a cited bound (target: none), and test coverage of money paths.

Decisions made in this proposal:

1. **Policy:** rules P1–P13 apply as written. Block hooks never skip silently: invariant-critical failures halt, and recoverable steps retry through an explicit, deterministic mechanism (P2).
2. **Package:** `inference-chain/pkg/safemath`, stdlib-only, written in-house, with adapters in `pkg/safemath/sdkbridge` and `pkg/safemath/decimaladapter`. go-safecast is not adopted.
3. **CI:** G115 and `nolintlint` in all four Go modules on new lines only; the SDK/decimal `forbidigo` rules on the chain; golangci-lint v2.14.0 in a separate job; the existing chain job unchanged.
4. **Release targets:** the policy, helpers and CI have no runtime effect and go to `main`. Changes to chain behavior ship in the next upgrade after v0.2.16, coordinated with #1379 and #1017. The voting-power cap returns an error and lands after the fork rebase (#1671).
5. **Other repositories and the analyzer:** PoC, contract and client changes go through each repository's own review; the contract fix follows #1858. The arithmetic analyzer stays non-blocking until it meets the criteria in the Enforcement section.

*Scan outputs and test fixtures are available on request.*
