# Canonical Dashboard API for Gonka

Status: design proposal
Date: 2026-11-13
Baseline: `v0.2.16`

## 1. Problem

After the `v0.2.16` upgrade, public dashboards started showing incorrect and inconsistent data, mostly around ML node weight and model coefficients. The chain data is correct. The problem is interpretation:

1. Every dashboard reimplements protocol formulas on its own: raw PoC weight × model coefficient, group caps, collateral, cPoC, power cap.
2. Field semantics change between upgrades: `cap_weight`, dynamic coefficients, deprecated `weight_scale_factor`, inference statistics moving to devshard.
3. There is no canonical, machine-readable interpretation.

Today changes are communicated through maintainer memos in `gonka-ai/gonka-docs` (`dashboard-maintainer-memo-v0.2.13` … `-v0.2.16-fee`). The v0.2.16 memo describes the H100-equivalent calculation in 8 steps across 5 endpoints. At epoch 456, six dashboards show six different values for that metric.

Goal: one source of display-ready values for shared, upgrade-sensitive and easy-to-misinterpret metrics, without new chain state.

Out of scope: standardizing all dashboards, covering every metric, historical series, market data.

## 2. How the data works today

### 2.1. Weight formation

`onEndOfPoCValidationStage` → `runWeightPipeline` (`x/inference/module/module.go`):

| # | Step | Code | Stored |
|---|------|------|--------|
| 1 | Raw node weight: `poc_weight = trunc(weight_node × timeNormalizationFactor)` | `module/chainvalidation.go:438-477` | `MLNodeInfo.poc_weight` in model subgroups |
| 2 | Model contribution, eligible models only: `trunc(effective_coeff_m × Σ poc_weight)` | `module/delegation_weight_calculator.go:294-370` | — |
| 3 | Group cap: `cap_factor × confirmed_total(N-1)`; the initial model and a sole eligible group are uncapped | same file, `:279-292` | — |
| 4 | Collateral: `trunc(W·bwr + min(W·(1−bwr), collateral / cpwu))`, skipped during the grace period | `keeper/collateral.go:79-165` | — |
| 5 | Power cap: the genesis parameter only enables it; the applied policy is 30% (50% / 40% for 2 / 3 participants) | `module/power_capping.go:40`, `keeper/bitcoin_rewards.go:424-445` | — |
| 6 | Real weight | — | `ActiveParticipant.weight`, root `ValidationWeight.weight` |
| 7 | Trust weight: `min(weight, P_{N-1})` (0 for newcomers), `applyZeroTrustFallback`, `applyTrustPowerCapping` | `module/previous_epoch_cap.go:98`, `module.go:902` | `ActiveParticipant.cap_weight` |
| 8 | Per-model voting power from `cap_weight` | `module/delegation_pipeline.go` | `ValidationWeight.voting_power` in subgroups |

The pre-collateral weight exists only in memory (`beforeCollateral`, `module.go:738`).

During the inference phase of the effective epoch, random Confirmation PoC events (`confirmation_poc.go:94`) and PoC Challenge (`poc_challenge.go:201`) run:

- `confirmation_weight = min(current, reading)`, where `reading` is the coefficient-weighted sum of measured and preserved nodes (`confirmation_poc.go:410-548`, `types/weight.go:72-98`);
- `ratio = min((reading / expected) / 0.909, 1)`; when `ratio < alpha_threshold` the participant becomes `INACTIVE`;
- `EffectiveConfirmedWeight = weight × confirmation_weight / rawConfirmationTotal` (`types/weight.go:21-41`) is the basis for rewards and next-epoch trust weight.

### 2.2. Same names, different meanings

| Field | Meaning | Includes coefficient |
|-------|---------|----------------------|
| `MLNodeInfo.poc_weight` | raw proven node capacity | no |
| `ValidationWeight.weight` in a subgroup | sum of raw `poc_weight` | no |
| `ValidationWeight.weight` in the root | real weight after caps and collateral | yes |
| `EpochGroupData.total_weight` | sum of the corresponding `weight`; root and subgroup use different units | root yes, subgroup no |
| `ActiveParticipant.weight` | real weight, reward basis | yes |
| `ActiveParticipant.cap_weight` | trust weight: governance, BLS, cPoC voting | yes |
| `ValidationWeight.confirmation_weight` | cPoC-confirmed weight, minimum across events | yes |
| `ValidationWeight.voting_power` | set only in subgroups; 0 in the root | — |
| `Participant.weight` | legacy, -1 at registration | — |
| `PoCModelConfig.weight_scale_factor` | deprecated; retained for migration and legacy compatibility (read when dynamic coefficients are not configured, `coefficients/coefficients.go:35`) | — |
| `ConfirmationWeightScale.effective_coefficient` | coefficient applied in the epoch | — |
| `ConfirmationWeightScale.base_coefficient` | controller state before dilution | — |
| `ParticipantCurrentStats.effective_weight`, `capped_weight` | never populated | — |

Common mistakes that follow:

1. Summing `poc_weight` and comparing with `weight` without the coefficient.
2. Multiplying by `weight_scale_factor`, which is empty since v0.2.16.
3. Using `base_coefficient` instead of `effective_coefficient`; above the target share, dilution applies: `effective = coeff_min + (base − coeff_min) × target / share`.
4. Showing `cap_weight` as reward weight, or `weight` as voting power.
5. Computing model share from raw weight or ignoring model eligibility.
6. Reading the power cap as 25%: the genesis parameter only enables the cap, is not exposed by any query, and the applied policy is hard-coded.
7. Recomputing the epoch reward independently.

### 2.3. Inference statistics

Since the move to devshard, statistics have three sources that disagree:

| Source | When written | Retention | One host, epoch 455 |
|--------|--------------|-----------|---------------------|
| `Participant.current_epoch_stats` | escrow settlements of the current epoch before accounts close (`msg_server_settle_devshard_escrow.go:80-96, 232-238`) | reset by account settlement | zero for everyone mid-epoch 456 |
| `EpochPerformanceSummary` | snapshot at account closing; reward basis | permanent | missed 563, invalidated 45, inferences 36,150 |
| `DevshardHostEpochStats` | every escrow settlement of the epoch, including late ones | ~2 epochs (`pruning.go:281, 382`) | missed 4,036, invalid 194, escrows 414 |

`DevshardHostEpochStats` has no completed-inference count, and `EpochPerformanceSummary` has no escrow or validation counters. "Validated inferences" and "validations performed" are different quantities.

`current_epoch_stats` is reset inside the atomic settlement transaction (`accountsettle.go:238-242`) and only for that epoch's participants (`accountsettle.go:49, 271`). Activation does not reset statistics (`module.go:1530`). A new participant gets clean statistics at registration. If settlement fails, formation of the next epoch continues (`module.go:787`) and the participant carries the old counters forward.

`confirmationPoCRatio` is `null` after a reset until the first event in the epoch (`types/current_epoch_stats.go:3`). Both cPoC events and PoC Challenge write it. `EpochPerformanceSummary` does not keep the ratio.

### 2.4. Participant status

`Participant` is a single mutable record keyed by address (`participant.go:43`, `participant.proto:16`), not an epoch snapshot. Activation resets status; successful settlement resets LLRs. SPRT thresholds are separate: `DowntimeHThreshold` for INACTIVE (`calculations/status.go:103`) and `InvalidationHThreshold` for INVALID (`:127`). A decision reason is stored only for terminal transitions, in the epoch's `ExcludedParticipants` (free-form `reason`); everything else is only logged (`participant_status.go:43`).

### 2.5. Rewards

- Schedule: `initial_epoch_reward × exp(decay_rate)^(epoch − genesis_epoch)` (`bitcoin_rewards.go:265-311`). The parameters can change through `UpdateParams`.
- The minted amount is capped by the remaining `StandardRewardAmount − TokenomicsData.total_subsidies` (`bitcoin_rewards.go:98-139`); settlement adds the minted amount to `total_subsidies` (`accountsettle.go:249`).
- Distribution: the gross proportional share is `trunc(w × reward / Σ fullWeight)`, where `w` is the reward weight after confirmation scaling, settlement cap, downtime punishment and delegation adjustments, and the denominator includes inactive participants (`bitcoin_rewards.go:740-998, 884`). A negative participant balance is then deducted from `RewardCoins` (`:952`), and subsidy exhaustion can prorate the result (`:109`). The undistributed remainder goes to governance. Final values always come from the shared settlement function.
- Settlement applies the 30% cap to positive confirmed weights of ACTIVE participants without checking the genesis parameter (`bitcoin_rewards.go:848`), so the percentage can differ from the formation cap.
- The settlement downtime test uses `BinomTestP0`, which a punishment-grace record can override (`accountsettle.go:90`), and `getDynamicP0` (`bitcoin_rewards.go:886`).
- `SettleAccounts` runs at the end of PoC validation, before the effective pointer switches (`module.go:786`); the delegation snapshot is then replaced with the next epoch's (`module.go:962`).

### 2.6. What is stored and for how long

| Data | Retention |
|------|-----------|
| Root and subgroup `EpochGroupData` (`ml_nodes`, `confirmation_weight_scales`) | never removed |
| `ActiveParticipants` (with `cap_weight`, `created_at_block_height`) | never removed; no gRPC query |
| `EpochPerformanceSummary` | never removed |
| `DevshardHostEpochStats` | ~2 epochs |
| `hardware_nodes`, collateral, balances, grants | current state only |
| State at height H on public nodes | `pruning_keep_recent = 20000` (≈1.3 epochs at ≈15.4K blocks per epoch) |

Historical queries must therefore read per-epoch structures at the current height rather than state at height H. `/v1/epochs/455/participants` already fails because its proof needs a pruned height.

### 2.7. Existing queries

- `EpochGroupData`, `CurrentEpochGroupData`, `EpochInfo`, `DynamicCoefficients/{epoch}` (raw controller state, no shares).
- `EstimateBitcoinReward/{participant}` replays settlement without writing state. It takes the epoch from the delegation snapshot (`query_bitcoin_reward_estimate.go:20`).
- `GetAllParticipantCurrentStats` (effective and capped weight fields are empty), `GetParticipantCurrentStats` (only `weight` and `reputation`, `query.proto:882`).
- `EpochPerformanceSummary(epoch)` loads all summaries for all epochs and filters (`query_epoch_performance_summary.go:40`); point lookups by `(participant, epoch)` are possible.
- `DevshardHostEpochStats` is single-participant only.
- `LastUpgradeHeight` is the height of the last upgrade; every handler updates it (`app/upgrade_tracking.go:10-36`); it is a single overwritten value (`partial_upgrade.go:58`).
- HTTP `common/queryapi` (edge-api, dapi): `/v1/epochs/*`, `/v1/participants`, `/v1/models`, `/v1/pricing`. The `coerceProtoJSONNumbers` helper turns 64-bit strings into JSON numbers (`common/queryapi/protojson.go:19`).
- An epoch fee-budget estimator exists in dapi (`decentralized-api/internal/server/admin/epoch_fee_budget.go:95, 245`).

## 3. Existing dashboards

As of 2026-11-13, epoch 456. Chain reference:

- `total_weight` = 1,980,056; 28 participants, 25 ACTIVE.
- Raw `poc_weight`: MiniMax 1,542,533, DeepSeek 1,450,396, GLM 513,025.
- `effective_coefficient`: MiniMax 0.3024, DeepSeek 0.4930 (base 0.5877, diluted), GLM 1.5565.
- Eligible contribution shares: MiniMax 23.6%, DeepSeek 36.1%, GLM 40.3%; compute shares: 28.0%, 45.0%, 27.0%; target 33.3%.

| Dashboard | Data source | Shows |
|-----------|-------------|-------|
| gonka.gg | own indexer (`gonkalabs/tx-scanner`), `/chain-api` | network, H100, reward per weight, model shares, ML nodes, GPUs, hosts, rewards, slashing, delegations, governance, supply, fee readiness |
| gonkascan.com | FastAPI + SQLite (`6block/gonkascan`, fork of `gonka-ai/gonka-tracker`) | epoch, hosts, missed/invalid, collateral, cPoC, models, hardware, H100, fee check, blocks |
| tracker.gonka.hyperfusion.io | based on gonka-tracker | hosts (on-chain / devshard / fees), cPoC, collateral, epoch reward |
| gonka.vip | own backend | hosts, "Active Weight", H100, weight per GPU, models (raw, coeff, cap, share, target) |
| tracker.gonka.pro | own backend | on-chain / gateway / network / ML nodes, "Confirmed weight" |
| gonkahub.com | own backend | epoch, H100, weight trend, hosts, cPoC, collateral, supply |
| gnk.space | aggregator | epoch, H100, tokens, models |
| bundled explorer (ping.pub) | node `/chain-api` | "Total Compute Power", epoch rewards, validators |
| monitoring.kaitaku.ai | Prometheus from `/v1/mlnodes/metrics` | ML node operational metrics |

Discrepancies:

- gonka.gg shows model shares of 44.0 / 41.4 / 14.6%, which are raw sums without the coefficient.
- gonkascan reports the raw sum as model weight: 3.5M against 1.98M network weight.
- gonka.vip computes coefficients and shares correctly, but compares the weight share with the target, while the target is defined for the compute share.
- H100 equivalent: 4,251 / 4,500 (gonka.gg), ≈4,710 (gonka.vip), ≈4,454 (hyperfusion, gnk.space), ≈4.25K (gonkahub), 0 (gonkascan), all with the same numerator.
- Current-epoch inference counters are 0 on the five dashboards that read `current_epoch_stats`.
- The cPoC ratio of an inactive host is shown as 7.69%, 6.99% and "—" on different sites.
- No dashboard shows `cap_weight`.

Metric frequency (GG gonka.gg, GS gonkascan, HF hyperfusion, VIP gonka.vip, PRO tracker.gonka.pro, HUB gonkahub, GNKS gnk.space, PING explorer):

| Metric | Dashboards |
|--------|------------|
| Network weight, host weight | GG, GS, HF, VIP, PRO, HUB, PING |
| Epoch, phase, timing | GG, GS, HF, VIP, PRO, HUB, GNKS, PING |
| Missed / invalid | GG, GS, HF, VIP, PRO, HUB |
| cPoC ratio, confirmed weight | GG, GS, HF, VIP, HUB, PRO |
| H100 equivalent | GG, GS, HF, VIP, HUB, GNKS |
| Collateral | GS, HF, HUB, VIP |
| Epoch reward, reward per weight | GG, PING, HF, GS |
| Host reward | GG, PING, HF, VIP |
| Models: coefficients, shares | VIP, GG, GS |
| ML node weight | GS, GG, VIP |
| Fee readiness | GG, GS, HF, HUB |
| GPU inventory, weight per GPU | GG, GS, VIP, HF |
| Supply, price | GG, HUB, GS, VIP |

## 4. Prior art

| Network | Approach | Takeaway for Gonka |
|---------|----------|--------------------|
| Cosmos `x/distribution` | `DelegationRewards` is computed on a branched query store and the write is discarded | derived values can be served without state; Gonka's `EstimateBitcoinReward` already works this way |
| Cosmos ADR-023 / ADR-044 | versioned packages, additive-only fields, new meaning means a new field or message | contract evolution policy |
| Osmosis | `SpotPriceV2` next to v1 instead of a silent replacement; heavy computation in a sidecar (SQS) | new semantics, new name |
| Ethereum Beacon API | rewards computed by replaying spec functions; per-endpoint versions; OpenAPI with releases | display-ready values and a published contract |
| Substrate | runtime APIs run runtime code at any block and are versioned with the runtime | derived values computed by the same code as consensus |
| Filecoin | `StateMinerPower` returns both raw byte power and quality-adjusted power | direct analogy: return raw and coefficient-scaled weight together |
| Bittensor | changing a field's unit without renaming it forced dashboards to rely on documentation | never change a field's meaning under the same name |
| Livepeer, Akash, Injective | statistics from operator-run off-chain indexers | indexers suit history, not canonical definitions |

Common practice: the definition of a value lives next to the protocol, and indexers build history from canonical snapshots.

## 5. Architecture

### 5.1. Components

1. **A `/v1/dashboard/*` facade in edge-api** (`common/queryapi`). The single dashboard contract: one observation height, display-ready values, explicit availability for every field.
2. **A `dashboardcalc` package** in `inference-chain`. Pure Go interpretation functions (contributions, shares, `EffectiveConfirmedWeight`, weight to confirm, collateral bounds, H100, fee budget) plus the field registry. The consensus pipeline and the facade call the same code; `common/go.mod` already depends on `inference-chain`.
3. **Minimal chain queries**, only for what needs keeper-internal state:
   - `ActiveParticipants(epoch_index)`;
   - a batch reward estimate with settlement diagnostics;
   - batch point lookups of `EpochPerformanceSummary` by address list;
   - `DevshardHostEpochStats` for all participants, paginated;
   - `GenesisOnlyParams`.

These queries do not change state and are not added to the Wasm accept list (`app/legacy.go:192-197, 303-332`), so they ship in a `release/vX.Y.Z-postN` patch release without governance. v1 adds no new state.

### 5.2. Alternatives considered

| Alternative | Why not the primary approach |
|-------------|------------------------------|
| edge-api only | no access to keeper inputs (reward estimate, `ActiveParticipants`) |
| `dashboard/v1` queries directly on chain | duplicates the contract and release surface; deferred |
| A per-epoch display snapshot in state | exact, but costs consensus storage and coordinated activation; a separate decision if exact history becomes a requirement |
| Spec and conformance tests only | keeps duplicated calculations |
| Fixing existing queries only | maintainers still assemble the interpretation themselves |

Indexers consume the facade for history; they must re-read late devshard settlements before the data is pruned.

### 5.3. Versioning and compatibility

Every response carries three separate values:

| Field | Meaning |
|-------|---------|
| `api_schema_version` | response shape; fields are only added |
| `interpretation_era` | protocol rules under which the requested epoch was formed |
| `build` | edge-api and chain binary versions, for diagnostics |

Epoch milestones:

| Milestone | Source |
|-----------|--------|
| PoC start | root `EpochGroupData.poc_start_block_height` |
| Coefficient config freeze | PoC start (`module.go:515`) |
| Weight formation | `ActiveParticipants.created_at_block_height` (`module.go:932`) |
| Activation | root `EpochGroupData.effective_block_height` (`module.go:1507`) |

`ActiveParticipants.effective_block_height` is not used: it is the formation height + 2 (`module.go:938`), and at epoch 456 it is 118 blocks earlier than the real activation. CometBFT validator-set activation is a separate milestone.

Eras are determined by formation height relative to upgrade heights in the facade registry; field presence is only checked for consistency.

| Era | Boundary | Returned |
|-----|----------|----------|
| `v0_2_16` | formed after v0.2.16 activation, with known compatibility at the observation height | all fields |
| `v0_2_16_upgrade_epoch` | formed before activation and effective at the upgrade height | stored values and derived values that do not depend on settlement rules |
| `legacy` | all other older epochs | stored values only; derived values `unavailable` (`era_not_supported`) |
| `unknown` | compatibility at the observation height is unknown, and the epoch was formed after the last boundary known to the build | derived values `unavailable` (`unsupported_interpretation`) |

On a compatible chain, an epoch formed long after the latest known upgrade is classified by its era interval and stays fully supported; `unknown` applies only when the compatibility check below fails.

Derived values are not reproduced for `legacy` because historical rules differ from current ones: for example, v0.2.12 settlement rescaled `confirmation_weight` through a coefficient-adjusted denominator (`release/v0.2.12`, `bitcoin_rewards.go:615-627`). Adapters for specific older eras can be added later, each tested against its own release.

The unknown-upgrade guard has two levels:

1. **Compatibility at the observation height.** The facade reads `LastUpgradeHeight` and checks it against the chain-specific boundary registry.
   - **Known:** the value matches a known applied boundary in the registry. Registry entries for upgrades that have not activated yet are ignored, so a newer facade also works before its newly supported upgrade activates. Every epoch is classified by its formation height against the era intervals; epochs formed after the latest applied boundary belong to that era and are fully supported.
   - **Unknown:** the value is not in the registry, is missing, or the query fails. Calculations based on current rules are disabled for all epochs: reward estimate, settlement diagnostics, `estimated_mintable`, reward per weight, collateral projection, fee budget, `status_at_observation`, cPoC ratio. Stored facts are still returned.
2. **Historical cutoff, only when compatibility is unknown.** Derived values from stored data are returned only for epochs formed no later than the last known boundary; later epochs are `unknown`. The cutoff follows the registry, not `LastUpgradeHeight`, because after two unknown upgrades that value points only to the second one.

Updating the boundary registry is part of every upgrade release.

General rules:

- Missing values are never replaced with current parameters, a coefficient of 1, or zero.
- The facade returns `capabilities`: supported sections and eras.
- All sub-queries of a response are pinned to one height (`x-cosmos-block-height`), returned as `meta.observed_height`; the cache is keyed by it.
- A sub-query failure makes dependent fields `unavailable`.

### 5.4. Epoch state and statistics provenance

Epoch state is three independent properties:

| Property | Values | Definition |
|----------|--------|------------|
| `formation` | `pending` / `formed` | `formed` when `ActiveParticipants` is written |
| `activation` | `upcoming` / `effective` / `past` | relative to the effective pointer |
| `accounting` | `not_started` / `open` / `closed` / `unsettled` | `closed`: a summary exists for (participant, epoch); `not_started`: no summary, epoch `upcoming`; `open`: no summary, epoch `effective`; `unsettled`: no summary, epoch `past` |

Between settlement and activation an epoch can be `effective` with `accounting = closed`. Closure is determined by summary existence, not by the pointer.

Statistics provenance is determined per participant U in the effective epoch, from the most recent previous epoch P in which U was in `ActiveParticipants`:

| `stats_continuity` | Condition |
|--------------------|-----------|
| `fresh` | no P; U registered within the search window |
| `clean` | a summary exists for (U, P) |
| `carried_over` | P exists, no summary for (U, P), P is `past` |
| `unknown` | history not established: window limit, unavailable data, error |

The search window is bounded to N epochs; N is calibrated by benchmark.

Availability rules:

| Condition | Counters, LLRs, cPoC ratio | Reward |
|-----------|----------------------------|--------|
| `effective`, `open`, `fresh` / `clean` | from `current_epoch_stats` | estimate from the batch query |
| `effective`, `open`, `carried_over` | `unavailable` (`carried_from_unsettled_epoch`) | estimate with `inputs_carried_over = true` |
| `effective`, `open`, `unknown` | `unavailable` (`provenance_unknown`) | estimate with `inputs_carried_over = null` |
| `closed` | counters from summary; LLRs and ratio `unavailable` | finalized from summary |
| `unsettled` | `unavailable` (`settlement_missing`) | `unavailable` |
| `not_started` | `null` | `null` |

The batch estimate returns its epoch; a mismatch with the requested epoch is an error.

### 5.5. Field availability

| Value | Current epoch | Closed epochs |
|-------|---------------|---------------|
| Raw weight, `effective_coefficient`, `weight`, `cap_weight`, `confirmation_weight` | yes | yes |
| Contributions, shares, `adjustment`, `confirmed_effective_weight`, `weight_to_confirm` | yes | `v0_2_16` era only |
| cPoC ratio | per 5.4 | no |
| Inference counters (accounting) | per 5.4 | yes, from summary |
| Devshard counters | yes | until pruned |
| Finalized participant reward | — | yes, when `closed` |
| Scheduled epoch reward | yes | only with current parameters (`params_basis = observation_height`) |
| `estimated_mintable`, reward per weight, reward estimate | `effective` + `open` only | no |
| Actually minted epoch reward | no | no |
| Formation and trust cap outcomes | no | no |
| Pre-collateral weight at formation, `adjustment` breakdown | no | no |
| Full-backing collateral threshold, projection | yes | no |
| Status, LLRs, thresholds | as `status_at_observation` only | no |
| Exclusion reason in the epoch | yes | yes |
| H100, fee readiness | yes | no |

## 6. API

### 6.1. Metric selection rule

A metric is in v1 if it is:

- **shared**: shown by at least 2 dashboards;
- **risky**: its semantics changed between v0.2.13 and v0.2.16, or it is already shown incorrectly or inconsistently;
- **feasible**: it can be computed deterministically from chain data. Protocol values use the same code as the protocol; display conventions (`kind = convention`) follow a published memo.

A field without which another field is ambiguous is included as a clarifying field.

### 6.2. v1 metrics

| # | Metric | Kind | Fields |
|---|--------|------|--------|
| 1 | Network weight | protocol | `network.total_weight` |
| 2 | Host weight | protocol | `participant.weight`, `.confirmed_effective_weight`, `.cap_weight` |
| 3 | Model | protocol | `models[].{eligible, effective_coefficient, base_coefficient, coeff_min, coeff_max, pinned, target_share_bps, compute_share_bps, pre_adjustment_weight_share_bps, raw_poc_weight, scaled_poc_capacity, eligible_contribution, ineligible_capacity, excluded_from_confirmation, oversupplied, coefficient_reduced}` |
| 4 | Host models and ML nodes | protocol | `participant.models[]`, `ml_nodes[]` |
| 5 | Confirmation | protocol | `participant.weight_to_confirm`, `.confirmation_poc_ratio`, `.confirmation_poc_ratio_state` |
| 6 | Inference statistics | protocol | `participant.inference.{accounting, devshard}`, `.stats_continuity` |
| 7 | Status | protocol | `participant.status_at_observation.*`, `.epoch_exclusion.*`, `.settlement_downtime.*` |
| 8 | Epoch reward | protocol | `network.epoch_reward.{scheduled_ngonka, estimated_mintable_ngonka}`, `.reward_per_weight_unit_ngonka` |
| 9 | Host reward | protocol | `participant.reward.{epoch_index, estimated_reward_ngonka, estimated_work_ngonka, finalized_reward_ngonka, finalized_work_ngonka, inputs_carried_over}` |
| 10 | Collateral | protocol | `participant.collateral.{deposited_ngonka, full_backing_threshold_weight, grace_period_active, projection.*}` |
| 11 | Power cap | protocol | `network.power_cap.{formation, trust, settlement}.*` |
| 12 | H100 equivalent | convention | `/v1/dashboard/h100` |
| 13 | Fee readiness | convention | `/v1/dashboard/participants/{address}/fee` |

Not in v1:

| Metric | Reason |
|--------|--------|
| Network trust weight, per-model voting power | no dashboard shows them |
| Universal missed / invalid limits | status is decided by SPRT over stored LLRs; no single per-volume number exists |
| Model share of actual rewards | depends on status, cPoC, settlement cap, downtime, delegations |
| Weight per GPU type, GPU inventory | dashboard-specific views, self-reported hardware |
| Epoch timing | available in `/v1/epochs/latest`; `meta` carries only epoch and phase |
| Circulating supply, price, holders, telemetry | no canonical definition or off-chain |
| Historical series | indexer responsibility |
| Governance, validators, blocks | standard Cosmos queries |

### 6.3. Routes

| Route | Content | Paginated |
|-------|---------|-----------|
| `GET /v1/dashboard/epochs/{epoch\|current}` | network, models (1, 3, 8, 11) | — |
| `GET /v1/dashboard/epochs/{epoch\|current}/participants?include=reward,collateral,status` | participants (2, 5–10) | yes |
| `GET /v1/dashboard/epochs/{epoch\|current}/participants/{address}` | participant with per-model breakdown | — |
| `GET /v1/dashboard/epochs/{epoch\|current}/participants/{address}/ml-nodes` | participant ML nodes | yes |
| `GET /v1/dashboard/h100` | H100, current epoch | — |
| `GET /v1/dashboard/participants/{address}/fee` | fee readiness | — |
| `GET /v1/dashboard/schema` | field registry, `capabilities` | — |

Pagination: `limit` (default 100, max 500) and `next_token`. Participants are ordered by address, ML nodes by `(model_id, node_id)`. A token is bound to the route, participant, epoch, filters and height; continuation reuses the token's height. A token from another scope is rejected; a token whose height has been pruned returns `410 token_expired`.

Envelope:

```json
{
  "meta": {
    "observed_height": "7037893",
    "epoch_index": "456",
    "effective_epoch_index": "456",
    "phase": "INFERENCE",
    "interpretation_era": "v0_2_16",
    "api_schema_version": "1",
    "build": "edge-api v0.2.17 / inferenced v0.2.16-post3"
  },
  "data": {},
  "availability": {
    "participants[].confirmation_poc_ratio": "current_epoch_only"
  },
  "next_token": null
}
```

### 6.4. JSON contract

- Fractional values are decimal strings: `"0.493042733571314169"`. `Decimal{value, exponent}` is not exposed: the mantissa exceeds 2^53.
- 64-bit integers (weights, heights, ngonka) are strings.
- `*_bps` values are numbers.
- A missing value is `null` with an explanation in `availability`; `0` means zero.
- The facade does not use `coerceProtoJSONNumbers`.

### 6.5. Field definitions

The field registry in `dashboardcalc` stores, for each field, its source, scope, units, rounding, availability, eras, kind and calculation function. `/v1/dashboard/schema`, documentation and part of the tests are generated from it.

**Weight and models**

| Field | Definition |
|-------|------------|
| `ml_nodes[].raw_poc_weight` | `MLNodeInfo.poc_weight` |
| `ml_nodes[].scaled_poc_capacity` | `trunc(effective_coeff_m × raw_poc_weight)`; capacity, not contribution |
| `participant.models[].scaled_poc_capacity` | `trunc(effective_coeff_m × Σ raw_poc_weight)` for any model |
| `participant.models[].eligible_contribution` | the same for eligible models only (`delegation_weight_calculator.go:294, 324`); 0 for an ineligible model |
| `models[].eligible` | model eligibility for the epoch, decided per model. `exclude_from_confirmation` is set when the model is ineligible **or** has no positive-weight PoC nodes (`confirmation_weight_scales.go:23, 67`; `confirmation_weight_scales_test.go:42`). With positive raw capacity, `eligible = !exclude_from_confirmation`; with zero raw capacity, `eligible = null`, and contributions are zero either way |
| `models[].scaled_poc_capacity`, `.eligible_contribution` | sums of per-participant truncated values |
| `models[].ineligible_capacity` | `scaled_poc_capacity − eligible_contribution` |
| `adjustment` | `weight − Σ eligible_contribution`: group cap, collateral, power cap |
| `weight` | `ActiveParticipant.weight` |
| `cap_weight` | `ActiveParticipant.cap_weight`; equals `weight` for epochs before v0.2.16 |
| `confirmed_effective_weight` | `types.EffectiveConfirmedWeight` |
| `weight_to_confirm` | `types.ConfirmationWeightOfParticipant`, excluding models with `exclude_from_confirmation` |
| `compute_share_bps` | `floor(10000 × D_m × raw_m / Σ D × raw)` across all models; `null` when the denominator is zero |
| `pre_adjustment_weight_share_bps` | `floor(10000 × eligible_contribution_m / Σ eligible_contribution)`; not a share of rewards |
| `oversupplied` | the controller's unrounded cross-product comparison (`coefficients.go:425`) |
| `coefficient_reduced` | `effective_coefficient < base_coefficient`; always `false` when `pinned` |

**Confirmation, statistics, status**

| Field | Definition |
|-------|------------|
| `confirmation_poc_ratio` | `current_epoch_stats.confirmationPoCRatio`, per the rules in 5.4 |
| `confirmation_poc_ratio_state` | `measured` / `no_event_yet` / `unavailable` |
| `inference.accounting` | `{inference_count, missed, validated_inferences, invalidated_inferences, final}` from the summary or `current_epoch_stats` |
| `inference.devshard` | `{missed, invalid, required_validations, completed_validations, cost_ngonka, escrow_count}` from `DevshardHostEpochStats`; `null` after pruning |
| `stats_continuity` | `fresh` / `clean` / `carried_over` / `unknown` |
| `status_at_observation` | `{status, invalid_llr, invalid_threshold, inactive_llr, inactive_threshold, observed_height}`; the participant record at the observation height, not epoch history |
| `epoch_exclusion` | `{excluded, reason}` from the epoch's `ExcludedParticipants`; without a record, `excluded = false`, `reason = null` |
| `settlement_downtime` | `{effective_p0, punishment_grace_applied, punishment_disabled, eligible_at_observation}` from the batch reward query; an assessment at the observation height |

**Rewards**

| Field | Definition |
|-------|------------|
| `network.epoch_reward.scheduled_ngonka` | schedule function with observation-height parameters; `params_basis = "observation_height"` |
| `network.epoch_reward.estimated_mintable_ngonka` | `min(scheduled, StandardRewardAmount − total_subsidies)`, only when `effective` + `open` |
| `network.reward_per_weight_unit_ngonka` | `estimated_mintable / Σ fullWeight`, only when `effective` + `open`; nominal rate |
| `reward.epoch_index` | epoch of the batch estimate |
| `reward.estimated_reward_ngonka`, `estimated_work_ngonka` | `SettleAmount.RewardCoins`, `WorkCoins` |
| `reward.finalized_reward_ngonka`, `finalized_work_ngonka` | `EpochPerformanceSummary.rewarded_coins`, `earned_coins` |

**Collateral**

| Field | Definition |
|-------|------------|
| `collateral.grace_period_active` | `latest_epoch ≤ grace_period_end_epoch` |
| `collateral.full_backing_threshold_weight` | the largest weight the deposit fully backs: grace → `null`; `bwr = 1` → `null`; `cpwu = 0` → `null` with a positive deposit, `0` with a zero deposit; otherwise `floor(deposited / cpwu / (1 − bwr))` |
| `collateral.projection.input_weight` | the participant's `Σ eligible_contribution` in the current epoch |
| `collateral.projection.projected_post_collateral_weight` | weight after collateral, before the power cap: grace → `input`; `input = 0` → 0; `bwr = 1` → `input`; `cpwu = 0` → `input` with a positive deposit, `trunc(input × bwr)` with a zero deposit; otherwise `trunc(input × bwr + min(input × (1 − bwr), deposited / cpwu))` |
| `collateral.projection.{target_epoch, assumptions}` | the next epoch; current parameters and deposit, no group caps, no hardware changes |

**Power cap**

| Field | Definition |
|-------|------------|
| `power_cap.formation` | `enabled = max_individual_power_percentage ≠ 0`; population: `weight` of the epoch's participants; 30%, or 50% / 40% for 2 / 3 participants |
| `power_cap.trust` | the same policy applied to the `cap_weight` vector |
| `power_cap.settlement` | no genesis-parameter check; population: positive confirmed weights of ACTIVE participants |
| `power_cap.settlement.estimated_reduced_any` | whether the settlement cap would reduce any weight with the current inputs |

**Display conventions**

| Field | Definition |
|-------|------------|
| `h100` | `{epoch_index, equivalent, weight_per_h100, reference_card, algorithm_version, sample: {nodes, cards, models[], mixed_hosts_included}, self_reported_hardware: true}`. v0.2.16 memo algorithm: median of `poc_weight × effective_coefficient / cards` over `NVIDIA H100 80GB HBM3` nodes in `INFERENCE` status with `poc_weight > 0` on active validators; `equivalent = total_weight / weight_per_h100`; an empty sample gives `null` |
| `fee` | `{cold_address, keys[], cold_spendable_ngonka, warm_spendable_ngonka, has_funded_operational_key, epoch_budget, issued_grants_mismatch}` |
| `fee.keys[]` | per warm key: `{warm_address, message_permissions: {store_commit, hardware_diff}, allowance: {type, spend_limit_ngonka, expiration, expired, supported}, authorized, usable_ngonka}`. A missing or expired grant gives `authorized = false`, `usable = 0`. A valid BasicAllowance with an empty limit gives `usable = cold_spendable`, otherwise `min(cold_spendable, spend_limit)`. `PeriodicAllowance` and `AllowedMsgAllowance` give `supported = false`, `usable = null`. authz and feegrant expirations are checked separately against block time |
| `fee.has_funded_operational_key` | a key exists with both valid permissions, `authorized = true`, `usable > 0` |
| `fee.epoch_budget` | `{fees_enabled, budget_known, budget_ngonka, covers_budget, assumptions}` from a shared function extracted from the dapi estimator. Fees disabled → `covers_budget = true`; budget unknown → `null` |

### 6.6. Performance

- The batch reward estimate reads only the requested epoch's subgroups by `(epoch, model)` key instead of `GetAllEpochGroupData`, and builds address maps once.
- Summaries are read by point lookup on `(participant, epoch)` for the participant page.
- Hardware is read only for the relevant participants.
- The reward estimate is enabled with `include=reward` and cached by height.
- ML nodes are served from a separate paginated route.
- Gas limits and timeouts are set from a benchmark on a mainnet state snapshot.
- `module_query_safe` and Wasm access are not part of v1.

## 7. Upgrade process

1. Behavioral fixtures for each era at upgrade and epoch boundaries, not only schema checks.
2. Separate H100 fixtures for each sampling-algorithm version.
3. Updating the facade's upgrade boundary registry is part of every upgrade release.
4. A "Dashboard changes" section in each upgrade README: changed fields, the new era, activation height, cache instructions.
5. A named owner keeps the maintainer list and collects acknowledgements before the upgrade height.
6. The facade runs on the rehearsal testnet before mainnet for dashboard smoke tests.
7. A deprecated field stays for at least one upgrade, marked `deprecated`.

## 8. Implementation plan

**Stage 1. Registry and contract**

- `dashboardcalc` package: field registry, eras, availability rules.
- Extract contributions, sums, shares, `oversupplied` and `adjustment` into pure functions; the pipeline switches to them without behavior change.
- JSON contract and fixtures.

**Stage 2. Chain queries (patch release)**

- `ActiveParticipants(epoch_index)`.
- Batch reward estimate: epoch, `RewardCoins`, `WorkCoins`, settlement diagnostics from the same inputs.
- Batch point lookups of `EpochPerformanceSummary`.
- `DevshardHostEpochStats` for all participants of an epoch.
- `GenesisOnlyParams`.
- Populate `effective_weight` and `capped_weight` in `GetAllParticipantCurrentStats`; add these fields to `GetParticipantCurrentStats`.
- Document the meaning of `current_epoch_stats` in the proto.

**Stage 3. Facade**

- Routes, envelope, `capabilities`, single height, cache, unknown-upgrade guard.
- H100 and fee readiness.
- `openapi.yaml`, `docs/dashboard-api.md`, a memo in `gonka-docs`.

**Stage 4. Process**: section 7, from the next upgrade.

**Later**: formation and settlement snapshots in state if exact history is needed; on-chain queries for direct and Wasm access.

**Tests**

- Equivalence: the estimate equals `RewardCoins` and `WorkCoins` of the actual settlement; the pipeline and the facade produce identical contributions.
- Eras: upgrade during PoC, upcoming epoch without `effective_coefficient`, pinned model, and the normal path: a compatible chain with an epoch formed long after the latest known upgrade stays `v0_2_16`.
- Unknown upgrade: a new upgrade with unchanged answers for known names; an upgrade during an effective epoch; two upgrades in a row.
- Epoch state: settlement–activation overlap; failed settlement followed by activation.
- Statistics provenance: `fresh`, `clean`, `carried_over` after an absence, `unknown` beyond the window.
- Historical status: a closed epoch does not return the current status and LLRs.
- Model eligibility: ineligible model with positive raw weight; eligible model without positive-weight nodes (`eligible = null`, zero contribution).
- Rewards: subsidy exhaustion, schedule parameter change.
- Collateral: grace, `bwr = 1`, `input = 0`, `cpwu = 0` with zero and positive deposits.
- Fee: expired and multiple grants, disabled fees, unknown budget, expired authz with a valid feegrant.
- Pagination: a token with a pruned height.
- JSON: values above 2^53, `null`, omitted and zero fields.
- Benchmark of batch queries; Testermint before and after an upgrade.

## 9. Open questions

1. Verify the `full_backing_threshold_weight` formula against code and fixtures.
2. The value of N for the previous-participation search window.
3. Location of `dashboardcalc`: `x/inference/types/dashboard` or a separate package.
4. Whether formation and settlement snapshots in state are needed, and for which fields.
5. The owner of maintainer communication.
6. Which nodes are guaranteed to serve the facade during release rollout.
7. Whether an APR estimate is needed.

## 10. References

Code (`inference-chain/`):

- `x/inference/module/`: `module.go`, `chainvalidation.go`, `delegation_weight_calculator.go`, `delegation_pipeline.go`, `previous_epoch_cap.go`, `power_capping.go`, `confirmation_poc.go`, `poc_challenge.go`
- `x/inference/keeper/`: `collateral.go`, `bitcoin_rewards.go`, `accountsettle.go`, `msg_server_settle_devshard_escrow.go`, `devshard_host_stats.go`, `pruning.go`, `participant_status.go`, `query_bitcoin_reward_estimate.go`, `query_epoch_performance_summary.go`, `partial_upgrade.go`
- `x/inference/types/weight.go`, `x/inference/coefficients/coefficients.go`, `x/inference/calculations/status.go`
- `proto/inference/inference/`: `epoch_group_data.proto`, `activeparticipants.proto`, `params.proto`, `query.proto`, `participant.proto`, `epoch_performance_summary.proto`, `excluded_participant.proto`
- `app/upgrade_tracking.go`, `app/legacy.go`, `app/upgrades/v0_2_16/upgrades.go`
- `common/queryapi/`, `decentralized-api/internal/server/admin/epoch_fee_budget.go`

Documents:

- `proposals/governance-artifacts/update-v0.2.16/README.md`
- `proposals/previous-epoch-weight-cap/README.md`
- `proposals/multi-model-poc/dynamic-coeff.md`, `dynamic-coeff-impl.md`
- `gonka-ai/gonka-docs`: `dashboard-maintainer-memo-v0.2.13.md` … `-v0.2.16-h100.md`, `-v0.2.16-fee.md`, `network-updates.md`

External:

- Cosmos SDK `x/distribution`: https://github.com/cosmos/cosmos-sdk/blob/main/x/distribution/keeper/grpc_query.go
- ADR-023: https://docs.cosmos.network/main/build/architecture/adr-023-protobuf-naming
- ADR-044: https://docs.cosmos.network/main/build/architecture/adr-044-protobuf-updates-guidelines
- Osmosis poolmanager v2: https://github.com/osmosis-labs/osmosis/blob/main/proto/osmosis/poolmanager/v2/query.proto
- Ethereum Beacon APIs: https://github.com/ethereum/beacon-APIs
- Substrate runtime API: https://paritytech.github.io/polkadot-sdk/master/polkadot_sdk_docs/reference_docs/custom_runtime_api_rpc/index.html
- Lotus API: https://github.com/filecoin-project/lotus/blob/master/documentation/en/api-methods-v1-stable.md
- Bittensor emissions: https://www.bittensor.com/docs/concepts/emissions
