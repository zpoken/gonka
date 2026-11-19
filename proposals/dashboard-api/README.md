# Canonical Dashboard API for Gonka

Status: design proposal
Date: 2026-11-19
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

1. **A `/v1/dashboard/*` facade in edge-api** (`common/queryapi`). The single dashboard contract: one observation height, display-ready values, explicit availability for every field. dapi does not serve these routes; the proxy publishes them with the other edge-api read routes.
2. **A `dashboardcalc` package** in `inference-chain/x/inference/dashboardcalc`. Pure Go interpretation functions plus the field registry: contributions, group cap, shares, oversupply, `EffectiveConfirmedWeight`, weight to confirm, collateral formula and bounds, power-cap policy, reward schedule and supply cap, eras and compatibility, epoch state and provenance, and the epoch fee budget. It imports only `types` and `coefficients`. The weight pipeline, settlement, collateral adjustment and the dapi fee estimator call these functions, so the facade and consensus share one implementation; `common/go.mod` already depends on `inference-chain`.
3. **Minimal read-only chain queries**, only for what needs keeper state (section 5.6).

The queries do not change state, are not marked module-query-safe and are not added to the Wasm accept list (`app/legacy.go:192-197, 303-332`), so they ship in a `release/vX.Y.Z-postN` patch release without governance. v1 adds no new state.

### 5.2. Alternatives considered

| Alternative | Why not the primary approach |
|-------------|------------------------------|
| edge-api only | no access to keeper inputs (reward estimate, `ActiveParticipants`) |
| `dashboard/v1` queries directly on chain | duplicates the contract and release surface; deferred |
| A per-epoch display snapshot in state | exact, but costs consensus storage and coordinated activation; a separate decision if exact history becomes a requirement |
| A per-participant node index in state | removes the full-record decode behind node queries, but adds state; deferred together with snapshots |
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

`meta` also carries `compatibility` (`known`, `genesis` or `unknown`) and `support_level` (`full`, `formation` or `stored`).

Epoch milestones:

| Milestone | Source |
|-----------|--------|
| PoC start | root `EpochGroupData.poc_start_block_height` |
| Coefficient config freeze | PoC start (`module.go:515`) |
| Weight formation | `ActiveParticipants.created_at_block_height` (`module.go:932`) |
| Activation | root `EpochGroupData.effective_block_height` (`module.go:1507`) |

`ActiveParticipants.effective_block_height` is not used: it is the formation height + 2 (`module.go:938`), and at epoch 456 it is 118 blocks earlier than the real activation. CometBFT validator-set activation is a separate milestone.

**Upgrade registry.** The facade keeps an ordered registry of upgrade names with the era each one starts, for example `v0.2.16 → v0_2_16` (with transition era `v0_2_16_upgrade_epoch`) and `v0.2.17 → v0_2_17`. Heights are not stored: the facade reads each name with `cosmos.upgrade.v1beta1.Query/AppliedPlan` at the observation height, so one build serves mainnet and every testnet. A registered upgrade that has not been applied yet is ignored, so a newer facade works before its newly supported upgrade activates. Operators can replace the registry without a rebuild (`DASHBOARD_ERA_BOUNDARIES`, a JSON array of `{upgrade_name, era, transition_era}`).

Eras are determined by formation height relative to the applied upgrade heights; field presence is only checked for consistency.

| Era | Epochs | Support level | Returned |
|-----|--------|---------------|----------|
| `v0_2_16`, `v0_2_17` | formed at or after the upgrade height | `full` | all fields |
| `v0_2_16_upgrade_epoch` | formed before the v0.2.16 height and settled after it: the next epoch was formed at or after the upgrade height, or is not formed yet | `formation` | stored values, contributions, shares and eligibility; values that depend on settlement rules are `unavailable` (`era_not_supported`) |
| `legacy` | all other older epochs | `stored` | stored values only; derived values `unavailable` (`era_not_supported`) |
| `unknown` | compatibility at the observation height is unknown, and the epoch was formed after the last boundary known to the build | `stored` | derived values `unavailable` (`unsupported_interpretation`) |

`v0_2_17` keeps the `v0_2_16` interpretation, so it has no separate transition era: an epoch formed under v0.2.16 and settled under v0.2.17 stays `v0_2_16`. The transition-epoch rule also covers an upgrade during PoC: the epoch formed before the upgrade and settled after it is the transition epoch, not the one effective at the upgrade height. On a compatible chain, an epoch formed long after the latest known upgrade is classified by its era interval and stays fully supported.

Derived values are not reproduced for `legacy` because historical rules differ from current ones: for example, v0.2.12 settlement rescaled `confirmation_weight` through a coefficient-adjusted denominator (`release/v0.2.12`, `bitcoin_rewards.go:615-627`). Adapters for specific older eras can be added later, each tested against its own release.

The unknown-upgrade guard has two levels:

1. **Compatibility at the observation height.** The facade reads `LastUpgradeHeight` and compares it with the applied registry upgrades.
   - **Known:** the value equals the height of the newest applied registry upgrade.
   - **Genesis:** no upgrade is recorded and no registry upgrade is applied (a fresh testnet or Testermint cluster); epochs are interpreted with the newest registry era.
   - **Unknown:** the value does not match, is missing while registry upgrades are applied, or a compatibility query fails. Calculations based on current rules are disabled for all epochs: reward estimate, settlement diagnostics, `estimated_mintable`, reward per weight, collateral projection, fee budget, live counters, LLRs, `status_at_observation`, cPoC ratio. Stored facts are still returned, and `availability["meta.compatibility"]` carries the reason.
2. **Historical cutoff, only when compatibility is unknown.** Derived values from stored data are returned only for epochs formed no later than the last known boundary; later epochs are `unknown`. The cutoff follows the registry, not `LastUpgradeHeight`, because after two unknown upgrades that value points only to the second one.

Updating the registry is part of every upgrade release (section 7).

General rules:

- Missing values are never replaced with current parameters, a coefficient of 1, or zero.
- The facade returns `capabilities`: supported sections, includes, eras with their support level, and the registry.
- Module queries of a response are pinned to one height (`x-cosmos-block-height`), returned as `meta.observed_height`. Block metadata (chain id, block time) is read with CometBFT `/header` at that height, with a full-block gRPC fallback that accepts the chain's maximum block size.
- Mutable data is cached by height. Applied upgrade plans and genesis-only parameters can be reused for the same chain and last upgrade height, which is checked at every new observation height; with unknown compatibility genesis-only parameters stay in the height-specific cache. Failed sub-queries are never cached.
- A sub-query failure makes dependent fields `unavailable`; chain error details are logged by edge-api, not returned.

### 5.4. Epoch state and statistics provenance

Epoch state is three independent properties:

| Property | Values | Definition |
|----------|--------|------------|
| `formation` | `pending` / `formed` | `formed` when `ActiveParticipants` is written |
| `activation` | `upcoming` / `effective` / `past` | relative to the effective pointer |
| `accounting` | `not_started` / `open` / `closed` / `unsettled` | `closed`: a summary exists for (participant, epoch); `not_started`: no summary, epoch `upcoming`; `open`: no summary, epoch `effective`; `unsettled`: no summary, epoch `past` |

Accounting is determined per participant; the epoch-level value is `closed` when any participant of the epoch has a summary. Between settlement and activation an epoch can be `effective` with `accounting = closed`. Closure is determined by summary existence, not by the pointer. The loaders a participant page needs are chosen from the accounting state of the participants on that page.

Statistics provenance is determined per participant U in the effective, open epoch, from the most recent previous epoch P in which U was in `ActiveParticipants`:

| `stats_continuity` | Condition |
|--------------------|-----------|
| `fresh` | no P; U registered within the search window, or the search reached the first epoch |
| `clean` | a summary exists for (U, P) |
| `carried_over` | P exists, no summary for (U, P), P is `past` |
| `unknown` | history not established: window limit, unavailable data, error |
| `not_applicable` | the epoch is not effective and open |

The search window is 8 epochs by default (`DASHBOARD_CONTINUITY_WINDOW`). Epoch membership for the search is loaded once per observation height and epoch, and reused across pages.

Availability rules:

| Condition | Counters, LLRs, cPoC ratio | Reward |
|-----------|----------------------------|--------|
| `effective`, `open`, `fresh` / `clean` | from `current_epoch_stats` | estimate from the batch query |
| `effective`, `open`, `carried_over` | `unavailable` (`carried_from_unsettled_epoch`) | estimate with `inputs_carried_over = true` |
| `effective`, `open`, `unknown` | `unavailable` (`provenance_unknown`) | estimate with `inputs_carried_over = null` |
| `closed` | counters from summary; LLRs and ratio `unavailable` (`accounting_closed`) | finalized from summary |
| `unsettled` | `unavailable` (`settlement_missing`) | `unavailable` |
| `not_started` | `unavailable` (`not_started`) | `unavailable` (`not_started`) |

With unknown compatibility, live counters are `unavailable` (`unsupported_interpretation`) and the estimate is not requested. The batch estimate returns its epoch; a mismatch with the requested epoch makes the estimate `unavailable` (`estimate_epoch_mismatch`).

### 5.5. Field availability

| Value | Current epoch | Closed epochs |
|-------|---------------|---------------|
| Raw weight, `effective_coefficient`, `weight`, `cap_weight`, `confirmation_weight` | yes | yes |
| Contributions, shares, eligibility, `adjustment`, formation and trust cap policy | yes | `full` and `formation` support levels |
| `confirmed_effective_weight`, `weight_to_confirm` | yes | `full` support level |
| cPoC ratio | per 5.4 | no |
| Inference counters (accounting) | per 5.4 | yes, from summary |
| Devshard counters | yes | until pruned |
| Finalized participant reward | — | yes, when `closed` |
| Scheduled epoch reward | yes | only with current parameters (`params_basis = observation_height`) |
| `estimated_mintable`, reward per weight, reward estimate, settlement cap diagnostics | `effective` + `open` only | no |
| Actually minted epoch reward | no | no |
| Formation and trust cap outcomes | no | no |
| Pre-collateral weight at formation, `adjustment` breakdown | no | no |
| Full-backing collateral threshold, projection | yes | no |
| Status, LLRs, thresholds | as `status_at_observation` only, `effective` + `open` | no |
| Exclusion reason in the epoch | yes | yes |
| Fee readiness | yes | no |

Values computed from stored inputs follow these rules:

- A subgroup listed by the epoch that is absent, or an active model without subgroup membership, makes `weight_to_confirm` and `confirmed_effective_weight` `unavailable` (`confirmation_inputs_missing`). A non-excluded scale without a stored coefficient makes them `unavailable` (`coefficient_missing`); the protocol helper's fallback coefficient of 1 is not used.
- When dynamic configs are in use, a configured model without a valid relative difficulty, or a scale without a config, makes every `compute_share_bps` and `oversupplied` `unavailable` (`difficulty_missing`), because the denominator is incomplete.
- Arithmetic is overflow-checked. A value that does not fit is `unavailable` (`overflow`) on the affected fields only; an excluded model whose capacity overflows keeps its known zero contribution, and weight shares use a wide denominator so representable shares survive large totals.

### 5.6. Chain queries

| RPC | REST path | Content and bounds |
|-----|-----------|--------------------|
| `ActiveParticipants` | `/productscience/inference/inference/active_participants/{epoch_index}` | the stored record; `found = false` when the epoch is not formed |
| `ActiveParticipantsSummary` | `/productscience/inference/inference/active_participants_summary/{epoch_index}` | compact per-participant, per-model aggregates: raw weight, positive weight, node count, subgroup membership and subgroup raw weight, trust weight; formation height, `cap_weight_applied`, `missing_subgroups`; optional address filter; `members_only` mode without subgroup processing; key pagination by address, at most 1000 per page |
| `ActiveParticipantNodes` | `/productscience/inference/inference/active_participant_nodes/{epoch_index}/{participant}` | one participant's ML nodes in stored order (model order, then node order), paginated by linear offset with the total count, at most 1000 per page; the unused `timeslot_allocation` is not returned |
| `ParticipantsBatch` | `/productscience/inference/inference/participants_batch` | compact records (address, status, join height, current-epoch statistics) for up to 1000 addresses; unbounded fields such as the inference URL are not returned |
| `EstimateBitcoinRewards` | `/productscience/inference/inference/estimate_bitcoin_rewards` | settlement replay for all participants or up to 1000 addresses: `epoch_index`, `scheduled_reward`, `mintable_reward`, `total_full_weight`, `effective_p0`, `punishment_grace_applied`, `punishment_disabled`, settlement power-cap percentage, population and `reduced_any`, and per participant `reward_coins`, `work_coins`, `full_weight`, `confirmed_weight`, `settlement_capped_weight`, `reward_weight`, `downtime_passed`, `status` |
| `EpochPerformanceSummaryBatch` | `/productscience/inference/inference/epoch_performance_summary_batch/{epoch_index}` | point lookups by `(participant, epoch)` for up to 1000 addresses, duplicates removed; store errors other than not-found are returned |
| `DevshardHostEpochStatsAll` | `/productscience/inference/inference/devshard_host_epoch_stats/{epoch_index}` | prefix scan of one epoch, at most 1000 per page |
| `GenesisOnlyParams` | `/productscience/inference/inference/genesis_only_params` | genesis-only parameters, including `max_individual_power_percentage` |

`GetAllParticipantCurrentStats` fills the never-populated `effective_weight` (the confirmed weight settlement starts from: `weight × confirmation_weight / weight_to_confirm`, clamped to `weight`, before the settlement power cap, downtime test and delegation adjustments, regardless of status) and `capped_weight` (`cap_weight`); `GetParticipantCurrentStats` returns both.

Paginated queries reject offset and reverse pagination and ignore `count_total`. The batch estimate and settlement load their inputs through the same function; settlement keeps its existing code path, and settlement diagnostics are collected only for the estimate query. Query handlers do not log. Node queries and filtered summaries share a query-only cache of the two most recent decoded `ActiveParticipants` records with an address index; an entry is reused only when the stored bytes are identical, concurrent misses decode once, and consensus code never reads it.

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
| 1 | Network weight | protocol | `network.total_weight`, `.participant_count` |
| 2 | Host weight | protocol | `participant.weight`, `.confirmed_effective_weight`, `.cap_weight`, `.confirmation_weight` |
| 3 | Model | protocol | `models[].{eligible, effective_coefficient, base_coefficient, coeff_min, coeff_max, relative_difficulty, pinned, target_share_bps, compute_share_bps, pre_adjustment_weight_share_bps, raw_poc_weight, scaled_poc_capacity, eligible_contribution, ineligible_capacity, excluded_from_confirmation, oversupplied, coefficient_reduced, participant_count, ml_node_count}` |
| 4 | Host models and ML nodes | protocol | `participant.models[]`, `ml_nodes[]` |
| 5 | Confirmation | protocol | `participant.weight_to_confirm`, `.confirmation_poc_ratio`, `.confirmation_poc_ratio_state` |
| 6 | Inference statistics | protocol | `participant.inference.{accounting, devshard}`, `.accounting`, `.stats_continuity` |
| 7 | Status | protocol | `participant.status_at_observation.*`, `.epoch_exclusion.*`, `.settlement_downtime.*` |
| 8 | Epoch reward | protocol | `network.epoch_reward.{scheduled_ngonka, params_basis, estimated_mintable_ngonka, total_reward_weight}`, `.reward_per_weight_unit_ngonka` |
| 9 | Host reward | protocol | `participant.reward.{epoch_index, mode, estimated_reward_ngonka, estimated_work_ngonka, estimated_reward_weight, finalized_reward_ngonka, finalized_work_ngonka, inputs_carried_over}` |
| 10 | Collateral | protocol | `participant.collateral.{deposited_ngonka, full_backing_threshold_weight, grace_period_active, projection.*}` |
| 11 | Power cap | protocol | `network.power_cap.{formation, trust, settlement}.*` |
| 12 | Fee readiness | convention | `/v1/dashboard/participants/{address}/fee` |

Not in v1:

| Metric | Reason |
|--------|--------|
| H100 equivalent | a display convention built on self-reported hardware; it stays in the v0.2.16 maintainer memo |
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
| `GET /v1/dashboard/epochs/{epoch}` | epoch state, network, models (1, 3, 8, 11) | — |
| `GET /v1/dashboard/epochs/{epoch}/participants?include=reward,collateral,status` | participants (2, 5–10) | yes |
| `GET /v1/dashboard/epochs/{epoch}/participants/{address}` | participant with every section and the per-model breakdown | — |
| `GET /v1/dashboard/epochs/{epoch}/participants/{address}/ml-nodes` | participant ML nodes | yes |
| `GET /v1/dashboard/participants/{address}/fee` | fee readiness | — |
| `GET /v1/dashboard/schema` | field registry, `capabilities`, availability reasons | — |

`{epoch}` is an epoch index, `current` (the effective epoch) or `latest` (the newest epoch, which is the upcoming one during PoC). Addresses are accepted in any bech32 case and normalized.

Sections: the participant page returns every section; the list returns `reward`, `collateral` and `status_at_observation` only when requested with `include`. A section not requested is omitted; a requested section that is unavailable is returned with `null` fields and a reason.

Pagination: `limit` (default 100, max 500) and `next_token`. Participants are ordered by address; ML nodes are returned in the participant's model order and stored node order, so the chain can seek a page by position. A token is HMAC-signed, expires after one hour and is bound to the route, participant, epoch, includes and height; continuation reuses the token's height. A tampered token or a token from another scope is rejected (`400 invalid_token`, `400 token_scope_mismatch`); an expired token or one whose height has been pruned returns `410 token_expired`. Replicas behind one balancer share the signing secret (`DASHBOARD_TOKEN_SECRET`).

Envelope:

```json
{
  "meta": {
    "observed_height": "7037893",
    "observed_time": "2026-11-13T12:00:00Z",
    "chain_id": "gonka-mainnet",
    "epoch_index": "456",
    "effective_epoch_index": "456",
    "latest_epoch_index": "456",
    "phase": "Inference",
    "interpretation_era": "v0_2_16",
    "support_level": "full",
    "compatibility": "known",
    "api_schema_version": "1",
    "build": "edge-api v0.2.17 / inferenced v0.2.17"
  },
  "data": {},
  "availability": {
    "network.epoch_reward.estimated_mintable_ngonka": "accounting_closed"
  },
  "next_token": null
}
```

Errors are `{"error": "<code>", "message": "..."}` with HTTP 400, 404, 410, 501, 502, 503 or 504.

### 6.4. JSON contract

- Fractional values are decimal strings with 18 fractional digits: `"0.493042733571314169"`. `Decimal{value, exponent}` is not exposed: the mantissa exceeds 2^53.
- 64-bit integers (weights, heights, ngonka) are strings.
- `*_bps` values and small counters are numbers.
- A missing value is `null` with a reason in the nearest `availability` map: the envelope's for network and model fields, the participant object's for participant fields. Per-item reasons use the item key, for example `models[<model_id>].compute_share_bps`, `ml_nodes[<model_id>].scaled_poc_capacity` or `keys[<warm_address>].usable_ngonka`. `0` means zero.
- The facade does not use `coerceProtoJSONNumbers`.
- `openapi.yaml` describes every response; the schemas are checked against the Go response types and the field registry.

Availability reasons:

| Reason | Meaning |
|--------|---------|
| `current_epoch_only` | the value exists only for the effective epoch |
| `accounting_open`, `accounting_closed`, `accounting_not_open` | the epoch is not settled yet, already settled, or not open |
| `not_started` | the epoch has not become effective |
| `settlement_missing` | the epoch is past but its summary does not exist |
| `carried_from_unsettled_epoch`, `provenance_unknown` | live counters still include an unsettled epoch, or their origin is not established |
| `era_not_supported`, `unsupported_interpretation` | the era does not support the value, or the chain applied an upgrade this build does not know |
| `epoch_not_formed`, `coefficient_not_computed` | formation has not happened yet |
| `confirmation_inputs_missing`, `coefficient_missing`, `difficulty_missing` | a stored input the calculation needs is absent |
| `no_dynamic_config`, `no_positive_capacity`, `not_applicable` | the model has no frozen dynamic config, has no positive raw capacity, or the field does not apply |
| `no_event_yet` | no confirmation PoC event in this epoch yet |
| `pruned` | the chain no longer keeps the data |
| `overflow` | the value does not fit in 64 bits |
| `query_failed`, `estimate_epoch_mismatch` | a sub-query failed or answered for another epoch |
| `grace_period`, `base_weight_ratio_full`, `unbounded`, `invalid_params` | the collateral threshold cannot be expressed |
| `genesis_params_missing`, `power_cap_disabled` | the power-cap policy cannot be determined or does not apply |
| `budget_unknown`, `unsupported_allowance`, `truncated` | fee inputs are incomplete, use an allowance type the facade does not evaluate, or exceed the route's budget |
| `zero_denominator` | a ratio has no denominator |

### 6.5. Field definitions

The field registry in `dashboardcalc` stores, for each field, its route, path, source (`meta`, `stored`, `formation`, `settlement`, `current_rules`, `convention`), type, unit, rounding, eras, availability, kind and calculation function. `/v1/dashboard/schema`, documentation and part of the tests are generated from it; every emitted path is in the registry and every registered path is emitted.

**Weight and models**

| Field | Definition |
|-------|------------|
| `ml_nodes[].raw_poc_weight` | `MLNodeInfo.poc_weight` |
| `ml_nodes[].scaled_poc_capacity` | `trunc(effective_coeff_m × raw_poc_weight)`; capacity, not contribution |
| `participant.models[].scaled_poc_capacity` | `trunc(effective_coeff_m × Σ raw_poc_weight)` for any model |
| `participant.models[].eligible_contribution` | the same for eligible models only (`delegation_weight_calculator.go:294, 324`); 0 for an ineligible model |
| `models[].effective_coefficient` | the stored `effective_coefficient`, or `weight_scale_factor` for epochs formed before v0.2.16 |
| `models[].pinned` | `coeff_min = coeff_max` on the epoch's frozen config |
| `models[].eligible` | model eligibility for the epoch, decided per model. `exclude_from_confirmation` is set when the model is ineligible **or** has no positive-weight PoC nodes (`confirmation_weight_scales.go:23, 67`; `confirmation_weight_scales_test.go:42`). With positive raw capacity, `eligible = !exclude_from_confirmation`; a model without a scale is ineligible; with zero raw capacity, `eligible = null`, and contributions are zero either way |
| `models[].raw_poc_weight` | sum of positive `poc_weight`, as the coefficient controller counts it |
| `models[].scaled_poc_capacity`, `.eligible_contribution` | sums of per-participant truncated values |
| `models[].ineligible_capacity` | `scaled_poc_capacity − eligible_contribution` |
| `adjustment` | `weight − Σ eligible_contribution`: group cap, collateral, power cap |
| `weight` | `ActiveParticipant.weight` |
| `cap_weight` | `ActiveParticipant.cap_weight`; equals `weight` for epochs before v0.2.16 |
| `confirmed_effective_weight` | the settlement confirmed weight: `types.EffectiveConfirmedWeight` over the subgroup nodes, clamped to `weight`; `weight` when the epoch has no scales |
| `weight_to_confirm` | `types.ConfirmationWeightOfModelNodes` over the participant's subgroup nodes, excluding models with `exclude_from_confirmation` |
| `compute_share_bps` | `floor(10000 × D_m × raw_m / Σ D × raw)` across configured models; `null` when the denominator is zero or incomplete |
| `pre_adjustment_weight_share_bps` | `floor(10000 × eligible_contribution_m / Σ eligible_contribution)`; not a share of rewards |
| `oversupplied` | the controller's unrounded cross-product comparison of the compute share with the target (`coefficients.go:425`) |
| `coefficient_reduced` | `effective_coefficient < base_coefficient`; always `false` when `pinned` |

**Confirmation, statistics, status**

| Field | Definition |
|-------|------------|
| `confirmation_poc_ratio` | `current_epoch_stats.confirmationPoCRatio`, per the rules in 5.4 |
| `confirmation_poc_ratio_state` | `measured` / `no_event_yet` / `unavailable` |
| `accounting` | the participant's accounting state (5.4) |
| `inference.accounting` | `{source, final, inference_count, missed, validated_inferences, invalidated_inferences}` from the summary (`final = true`) or `current_epoch_stats` |
| `inference.devshard` | `{missed, invalid, required_validations, completed_validations, cost_ngonka, escrow_count}` from `DevshardHostEpochStats`; a missing record is zero while the epoch is within the two-epoch retention or another participant has a record for it, otherwise `null` (`pruned`) |
| `stats_continuity` | `fresh` / `clean` / `carried_over` / `unknown` / `not_applicable` |
| `status_at_observation` | `{status, invalid_llr, invalid_threshold, inactive_llr, inactive_threshold, observed_height}`; the participant record at the observation height, not epoch history |
| `epoch_exclusion` | `{excluded, reason, exclusion_block_height}` from the epoch's `ExcludedParticipants`; without a record, `excluded = false`, `reason = null` |
| `settlement_downtime` | `{effective_p0, punishment_grace_applied, punishment_disabled, eligible_at_observation}` from the batch reward query; an assessment at the observation height |

**Rewards**

| Field | Definition |
|-------|------------|
| `network.epoch_reward.scheduled_ngonka` | schedule function with observation-height parameters; `params_basis = "observation_height"` |
| `network.epoch_reward.estimated_mintable_ngonka` | `min(scheduled, StandardRewardAmount − total_subsidies)`, only when `effective` + `open` |
| `network.epoch_reward.total_reward_weight` | the settlement denominator: full weights including inactive participants |
| `network.reward_per_weight_unit_ngonka` | `estimated_mintable / total_reward_weight`, only when `effective` + `open`; nominal rate |
| `reward.epoch_index`, `reward.mode` | epoch of the estimate or summary; `estimate` or `finalized` |
| `reward.estimated_reward_ngonka`, `estimated_work_ngonka`, `estimated_reward_weight` | `SettleAmount.RewardCoins`, `WorkCoins`, and the reward weight after confirmation, settlement cap, downtime and delegation adjustments |
| `reward.finalized_reward_ngonka`, `finalized_work_ngonka` | `EpochPerformanceSummary.rewarded_coins`, `earned_coins` |

**Collateral**

| Field | Definition |
|-------|------------|
| `collateral.grace_period_active` | `latest_epoch ≤ grace_period_end_epoch` |
| `collateral.full_backing_threshold_weight` | the largest weight the deposit fully backs: grace → `null`; `bwr = 1` → `null`; `cpwu = 0` → `null` with a positive deposit, `0` with a zero deposit; otherwise `floor(deposited / cpwu / (1 − bwr))`, corrected against the collateral formula so that the returned weight is fully backed and the next one is not |
| `collateral.projection.input_weight` | the participant's `Σ eligible_contribution` in the current epoch |
| `collateral.projection.projected_post_collateral_weight` | weight after collateral, before the power cap: grace (for the target epoch) → `input`; `input = 0` → 0; `bwr = 1` → `input`; `cpwu = 0` → `input` with a positive deposit, `trunc(input × bwr)` with a zero deposit; otherwise `trunc(input × bwr + min(input × (1 − bwr), deposited / cpwu))` |
| `collateral.projection.{target_epoch, assumptions}` | the next epoch; current parameters and deposit, no group caps, no delegation adjustments, no power cap, no hardware changes |

**Power cap**

| Field | Definition |
|-------|------------|
| `power_cap.formation` | `enabled = max_individual_power_percentage ≠ 0`; population: `weight` of the epoch's participants; 30%, or 50% / 40% for 2 / 3 participants with positive weight; `positive_count` and `max_percentage` need formation support |
| `power_cap.trust` | the same policy applied to the `cap_weight` vector |
| `power_cap.settlement` | no genesis-parameter check; population: positive confirmed weights of ACTIVE participants; `positive_count`, `max_percentage` and `estimated_reduced_any` from the batch estimate |

**Display conventions**

| Field | Definition |
|-------|------------|
| `fee` | `{cold_address, epoch_index, epoch_member, keys[], cold_spendable_ngonka, warm_spendable_ngonka, has_funded_operational_key, epoch_budget, issued_grants_mismatch, issued_grantees}` |
| `fee.keys[]` | per warm key with a valid authz grant for `MsgPoCV2StoreCommit` or `MsgSubmitHardwareDiff`: `{warm_address, message_permissions: {store_commit, hardware_diff}, allowance: {type, spend_limit_ngonka, expiration, expired, supported}, authorized, usable_ngonka, warm_spendable_ngonka}`. A missing or expired grant gives `authorized = false`, `usable = 0`. A valid BasicAllowance with an empty limit gives `usable = cold_spendable`, otherwise `min(cold_spendable, spend_limit)`. `PeriodicAllowance` and `AllowedMsgAllowance` give `supported = false`, `usable = null` (`unsupported_allowance`). authz and feegrant expirations are checked separately against block time |
| `fee.has_funded_operational_key` | `true` when a key has both permissions, `authorized = true` and `usable > 0`; `null` (`unsupported_allowance`) when such a key's usable amount cannot be evaluated; `null` (`truncated`) when the listings were incomplete; `false` otherwise |
| `fee.warm_spendable_ngonka` | sum over displayed keys; never counted as usable; `null` when keys were truncated |
| `fee.issued_grants_mismatch` | the cold account issued an allowance to a non-operational grantee while an operational key has no valid allowance; `null` when the listings were incomplete |
| `fee.epoch_budget` | `{fees_enabled, budget_known, budget_ngonka, covers_budget, count_source, count, serving_models, assumptions}` from the fee-budget function shared with the dapi estimator. The count is the participant's own StoreCommits in the latest PoC stage, or the top participant's when it has none. Fees disabled → `covers_budget = true`; budget unknown → `null`; `covers_budget` is `true` when an evaluated allowance covers the budget, `null` when an operational key's allowance cannot be evaluated or the listings were incomplete, otherwise `false` |

### 6.6. Performance and resource bounds

- Every chain loader is memoized by observation height and shared between concurrent requests. Each shared operation owns a bounded, height-pinned work context, so a disconnecting caller only stops its own wait; a panic inside shared work is contained and returned as an error. Each request has a 30-second deadline.
- The epoch view is built from paginated compact summaries, never from the nested `ActiveParticipants` record, so no single response depends on the epoch's size.
- Participant records, provenance and devshard statistics are loaded for the requested page only; records are trimmed to the fields the dashboard reads before caching.
- ML nodes are loaded in fixed 500-node chunks on demand, into a separate node cache bounded by size (32 MB) with least-recently-used eviction, so `limit=1` costs one chunk and old heights age out.
- The fee route reads one page (1000) of grants and allowances, displays at most 16 operational keys and looks up allowances of displayed keys missing from a truncated listing; the stage's StoreCommit snapshot is loaded once per height and shared by all fee requests.
- Hardware data is not read.
- Gas limits and timeouts are set from a benchmark on a mainnet state snapshot.
- `module_query_safe` and Wasm access are not part of v1.

## 7. Upgrade process

1. Behavioral fixtures for each era at upgrade and epoch boundaries, not only schema checks.
2. Updating the facade's upgrade registry is part of every upgrade release. A test fails while a tracked upgrade handler is neither in the registry nor listed as pending.
3. A "Dashboard changes" section in each upgrade README: changed fields, the new era, activation height, cache instructions.
4. A named owner keeps the maintainer list and collects acknowledgements before the upgrade height.
5. The facade runs on the rehearsal testnet before mainnet for dashboard smoke tests.
6. A deprecated field stays for at least one upgrade, marked `deprecated`.

## 8. Implementation plan

**Stage 1. Shared package and contract**

- `dashboardcalc`: field registry, eras, availability rules, epoch view.
- Extract contributions, sums, shares, `oversupplied`, `adjustment`, collateral, power cap, reward schedule, supply cap and the fee budget into pure functions; the weight pipeline, settlement, collateral adjustment and the dapi estimator switch to them without behavior change.
- JSON contract and behavioral fixtures.
- This design document in `proposals/dashboard-api/`.

**Stage 2. Chain queries (patch release)**

- The queries in section 5.6.
- Populate `effective_weight` and `capped_weight` in `GetAllParticipantCurrentStats`; add these fields to `GetParticipantCurrentStats`.

**Stage 3. Facade**

- Routes, envelope, `capabilities`, single height, caching, unknown-upgrade guard, signed pagination.
- Fee readiness.
- `openapi.yaml` and proxy routes.

**Stage 4. Documentation and rollout**

- `docs/dashboard-api.md` for maintainers, including the meaning of `current_epoch_stats`, and a memo in `gonka-docs`.
- A benchmark on a mainnet state snapshot.
- Testermint runs on a fresh cluster and before and after an upgrade.
- The process in section 7, from the next upgrade.

**Later**: formation and settlement snapshots in state if exact history is needed; a per-participant node index in state; on-chain queries for direct and Wasm access.

**Tests**

- Equivalence: the estimate equals `RewardCoins`, `WorkCoins` and the minted amount of the actual settlement; the pipeline and the facade produce identical contributions; randomized comparison of the reward calculation before and after the extraction.
- Eras: upgrade during PoC, upcoming epoch without `effective_coefficient`, pinned model, genesis chain, and the normal path: a compatible chain with an epoch formed long after the latest known upgrade stays fully supported.
- Unknown upgrade: a new upgrade with unchanged answers for known names; an upgrade during an effective epoch; two upgrades in a row.
- Epoch state: settlement–activation overlap; failed settlement followed by activation; mixed accounting on one page.
- Statistics provenance: `fresh`, `clean`, `carried_over` after an absence, `unknown` beyond the window.
- Historical status: a closed epoch does not return the current status and LLRs.
- Model eligibility: ineligible model with positive raw weight; eligible model without positive-weight nodes (`eligible = null`, zero contribution).
- Missing inputs and overflow: missing subgroup, missing coefficient, missing difficulty or config, overflowing capacities and shares.
- Rewards: subsidy exhaustion, schedule parameter change.
- Collateral: grace, `bwr = 1`, `input = 0`, `cpwu = 0` with zero and positive deposits; the threshold against the collateral formula.
- Fee: expired and multiple grants, disabled fees, unknown budget, expired authz with a valid feegrant, unsupported allowance types, truncated listings.
- Pagination: tampered, foreign-scope, expired and pruned-height tokens; node chunks across varying limits.
- Resources: concurrent requests, caller cancellation, large epochs, large participants, large blocks.
- JSON: values above 2^53, `null`, omitted and zero fields; responses validated against `openapi.yaml`.

## 9. Open questions

1. Whether formation and settlement snapshots in state are needed, and for which fields.
2. The owner of maintainer communication.
3. Which nodes are guaranteed to serve the facade during release rollout.
4. Whether an APR estimate is needed.
5. The continuity search window and gas limits, to be calibrated on a mainnet snapshot.

## 10. References

Code (`inference-chain/`):

- `x/inference/module/`: `module.go`, `chainvalidation.go`, `delegation_weight_calculator.go`, `delegation_pipeline.go`, `previous_epoch_cap.go`, `power_capping.go`, `confirmation_poc.go`, `confirmation_weight_scales.go`, `poc_challenge.go`
- `x/inference/keeper/`: `collateral.go`, `bitcoin_rewards.go`, `accountsettle.go`, `msg_server_settle_devshard_escrow.go`, `devshard_host_stats.go`, `pruning.go`, `participant_status.go`, `query_bitcoin_reward_estimate.go`, `query_epoch_performance_summary.go`, `partial_upgrade.go`
- `x/inference/types/weight.go`, `x/inference/coefficients/coefficients.go`, `x/inference/calculations/status.go`
- `proto/inference/inference/`: `epoch_group_data.proto`, `activeparticipants.proto`, `params.proto`, `query.proto`, `participant.proto`, `epoch_performance_summary.proto`, `excluded_participant.proto`
- `app/upgrade_tracking.go`, `app/legacy.go`, `app/upgrades/v0_2_16/upgrades.go`, `app/upgrades/v0_2_17/upgrades.go`
- `common/queryapi/`, `common/chain/`, `proxy/entrypoint.sh`, `decentralized-api/internal/server/admin/epoch_fee_budget.go`

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
