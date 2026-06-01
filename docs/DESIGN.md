# Tesseracoin Protocol Design

Prepared by SoftMillennium — April 2025

> **Feedback & corrections welcome.** Spotted an error or want to weigh in on the
> design? Open an [issue](https://github.com/softmillennium/tesseracoin/issues) or
> [discussion](https://github.com/softmillennium/tesseracoin/discussions), or send
> a PR. Tesseracoin is community-first — this design is meant to be argued with.

---

## Vision

Tesseracoin implements **POWP** (Proof-of-Work + Pledge), a hybrid consensus that combines PoW's
Sybil resistance with an economic skin-in-the-game requirement. The reward distribution layer
is **PoCRR** (Proof of Contribution + Random Reward), which splits block rewards to incentivise
broad participation beyond just mining.

**Core thesis**: Pure PoW wastes energy at scale; pure PoS creates "rich-get-richer" dynamics.
POWP bridges both — PoW provides Sybil resistance while pledged economic commitment becomes a
first-class fork-choice input, reducing power concentration. PoCRR ensures even non-miners
benefit from active participation in the network.

---

## POWP: Proof-of-Work + Pledge

### Block Production

To produce a valid block, a miner must:

1. Solve a PoW puzzle (Bitcoin-style, difficulty-adjusted SHA-256)
2. Submit a **signed pledge** — an economic commitment from their wallet balance, locked against
   winning the block

The PoW hash commitment includes the pledge root, so the pledge amount is bound to the work.
A miner cannot substitute a larger pledge after finding a valid nonce.

### Fork Choice

When competing chains are of equal height, the winner is determined by a deterministic
multi-layer tie-breaker (all comparisons are applied in order; the first difference wins):

| Priority | Rule | Direction |
|---|---|---|
| 1 | Tip height | Higher wins |
| 2 | Nonce | Lower wins (more work at same difficulty) |
| 3 | Cumulative pledge | Higher total pledged stake wins |
| 4 | VRF seed | Lexicographically smaller wins |
| 5 | Block hash | Lexicographically smaller wins |

This layering prevents short-range selfish mining and grinding attacks — a miner cannot
improve their position by repeatedly trying different nonces once a valid hash is found,
because a valid lower nonce from a competing miner wins outright.

### Pledge Mechanics

- Pledge is a wallet-signed commitment from the miner's balance (the signature scheme is pluggable — see *Addresses and Cryptography*)
- Maximum pledge at height `h` = 10% of the block reward at that height
- The pledge hash (`pledge_root`) is committed into the block header
- A `pledge_donation` transaction inside the block records the pledge amount on-chain
- Pledge constraints are verified by all nodes at block validation time

---

## PoCRR: Proof of Contribution + Random Reward

### Motivation

Traditional consensus models concentrate rewards to those with the most hash power (PoW) or
the most capital (PoS). Small participants are structurally dis-incentivised, leading to
centralisation over time.

PoCRR adds a random reward layer: a portion of each block reward is distributed to a
randomly selected **active** network participant chosen via VRF — not just the miner.
This incentivises genuine engagement from a broad participant base.

### Active Window

Only participants active within the last **1,000 blocks** are eligible for the random reward.
"Active" means:

- Sent or received at least one transaction, **or**
- Submitted a pledge in any block within the window

The window prevents sybil gaming by requiring recent, genuine on-chain activity.

### VRF Selection

The VRF seed is computed deterministically:

```
vrf_seed = sha256(prev_block_hash + nonce + pledge_root)
```

Eligible participants are ranked by `sha256(seed + participant_pubkey)`. The participant with
the lowest resulting hash wins. This is provably fair — no miner can predict or bias the
selection before committing to a nonce, and any observer can independently verify the winner.

### Reward Split

The intended split per block:

| Recipient | Amount |
|---|---|
| Miner | Coinbase reward − pledge amount |
| VRF-selected active participant | Pledge amount |
| Transaction fees | Miner (full) |

VRF selection and activity tracking (`is_active`) live in the POWP consensus authority (`tesseracoin/consensus_powp.py`).
The reward split is enforced at block construction time: `_mine_block` builds the coinbase and
`pledge_donation` transactions before PoW begins, committing the split into the merkle root.

---

## Economic Model

### Supply Parameters

| Parameter | Value |
|---|---|
| Maximum supply | 21,000,000 TSR |
| Initial block reward | 5 TSR |
| Halving interval | 2,000,000 blocks |
| Target block time | 60 seconds |
| Difficulty adjustment | Every 2,016 blocks |
| Divisibility | 8 decimal places (2.1 quadrillion base units) |
| Fee rate (floor) | 0.00001 TSR per kB (1 sat/B, Bitcoin minrelay parity) |
| Max transactions per block | 100 |

The 8-decimal divisibility (giving 2.1 × 10¹⁵ base units) ensures sufficient granularity
for micropayments, IoT settlement, and cross-border remittance without impractical per-unit
costs at higher valuations.

### Fee and Pledge Evolution

Miner economics are designed to stay viable as the block subsidy halves — the
pledge requirement falls and fee revenue rises as the network matures. The
**default** consensus achieves this with *static, deterministic* rules, not a
per-block controller:

- **Pledge auto-tapers with the subsidy.** Maximum pledge is 10% of the block
  reward, and the reward halves every 2,000,000 blocks, so the pledge ceiling
  falls in step with the subsidy automatically — no tuning required.
- **The fee floor is a static anti-spam minimum** (1 sat/B, Bitcoin minrelay
  parity). Actual miner fee revenue comes from the **market tips** users pay
  above the floor, which grow with adoption and congestion — this is what
  supplements the declining subsidy, the same mechanism Bitcoin relies on.

So the "pledge-heavy → fee-heavy" transition is driven by the halving schedule
plus a tip market, not by a dynamic floor.

> **Experimental — gated, not active.** A per-block *dynamic* fee/pledge
> controller was prototyped as the `POWPv3` consensus authority
> (`fee_rate ×= 1 + α·(utilisation − 0.5)`, `pledge_cap ×= 1 + β·(initial −
> miners)/initial`; modelled in `docs/fee_pledge_fluctuation_graph.py`,
> α = 0.0001, β = 0.0002). It is **refused by the activation validator**: its
> values derive from the validator's chain tip rather than the block's ancestry,
> so honest nodes on different forks can disagree on a block's validity — a
> consensus split. See `CONSENSUS_UPGRADES.md` (authority id 4, marked GATED)
> and `tests/test_powp_v3_determinism.py`.

---

## Protocol Flow

```
1. Build block
   ├── Select transactions from mempool (max 100, ordered by fee then type)
   ├── Create coinbase transaction (miner → miner, amount = block_reward + fees)
   └── Create pledge (≤ 10% of current block reward), sign it

2. Mine
   ├── Assemble BlockHeader (prev_hash, merkle_root, pledge_root, target, timestamp, …)
   ├── Increment nonce until sha256(header) < target
   └── Abort cleanly if chain tip changes under us (bootstrap reorg)

3. Finalise
   ├── Sign the block header (miner's wallet signature)
   ├── Verify tip hasn't moved (race guard)
   └── Verify block locally (PoW, pledge constraints, VRF seed, merkle root, signatures)

4. Gossip
   ├── Add block to local chain
   ├── Broadcast to all peers (X-Origin header prevents echo)
   └── Peers: validate → add or reject

5. Fork resolution (on receiving a competing chain)
   ├── Apply multi-layer tie-breaker
   ├── If better: find common ancestor, orphan divergent blocks, re-add fork blocks
   └── Return orphaned transactions to mempool
```

---

## Addresses and Cryptography

- **Signature schemes** (pluggable, `sigschemes.py`): `secp256k1` (legacy ECDSA), `ed25519`
  (the default — compact signatures), and post-quantum `dilithium2` (FIPS 204 ML-DSA-44, via
  liboqs). Each wallet records its scheme and the address encodes which one it uses, so all
  three can coexist on one chain.
- **Address format**: bech32, HRP `tsr` — e.g. `tsr1q...`, derived from `sha256(pubkey)[:20]`
  with witness version 0
- **Chain ID**: 670210 — included in every transaction signing payload to prevent cross-chain replay
- **Block signature**: the miner signs `hash_for_pow(header)` (not the full header hash, so the
  signature field itself is not part of the signed data)

---

## Transaction Types

| Type | Purpose | Implemented |
|---|---|---|
| `currency` | Standard value transfer | Yes |
| `coinbase` | Block reward payment to miner | Yes |
| `pledge_donation` | On-chain record of miner's pledge | Yes |
| `nft` | Non-fungible token | Scaffolded only |
| `smart_contract` | Contract call | Scaffolded only |

---

## Implementation Status

### Done

- Bitcoin-style PoW with Bitcoin-style difficulty adjustment (every 2,016 blocks, ±4× cap)
- Pledge creation, signing, verification, and fork-choice weighting
- VRF seed committed in block header, verified at validation time
- Active participant tracking (`is_active`, `eligible_proposers`)
- Multi-layer deterministic fork choice
- FastAPI HTTP gossip with X-Origin deduplication
- Background bootstrap sync (15s poll, reorg to common ancestor)
- SQLite persistence with WAL mode and connection pooling (32+16)
- Orphaned-block pruning on startup
- Per-node log files at `DATA_DIR/logs/node.log`
- Bech32 addresses with pluggable signing — `secp256k1` / `ed25519` (default)
- **Post-quantum signatures available** — Dilithium2 / ML-DSA-44 (FIPS 204) via `sigschemes.py`
- **Chain-derived balance validation** — sender balance checked against confirmed on-chain history (`get_confirmed_balance`) at block validation
- **PoCRR reward disbursement** — VRF winner selected from the previous block's `vrf_seed`; the `pledge_donation` is credited on block receipt
- **Mempool expiry** — unconfirmed transactions evicted after `mempool_ttl_blocks` (`_evict_expired_mempool`)
- **Equivocation slashing + pledge-renewal-window fork-choice decay** (`pledge_renewal_window_blocks`, `slashing_lookback_blocks`)
- **Pluggable consensus authority interface** — swappable difficulty / reward / proposer / pledge / mining / validation / fork-choice policies (`consensus_api.py`, `@register_consensus`; see *Swappable Consensus* below)

### Not Yet Implemented

- **NFT / smart-contract execution** — the `nft` and `smart_contract` transaction types are accepted and stored, but there is no execution logic (scaffolded only)
- **Formal UTXO / account chainstate layer** — balances are derived from confirmed on-chain history at validation time (above), but there is no separate account-model layer; tightening PoCRR recipient enforcement inside `verify_block` waits on it

---

## Swappable Consensus (implemented)

Tesseracoin's consensus rules — difficulty, reward, proposer selection, pledge, mining, block validation, and fork choice — live behind a plugin layer (`tesseracoin/consensus_api.py`). The default `POWPAuthority` (consensus_id=1) implements the base POWP rules natively; legacy POWP blocks (`consensus_id == 1`, empty `consensus_proof`) hash byte-for-byte as before, so existing chains stay compatible. Other authorities are registered alongside it — `PoAAuthority` (id=2, owner-signed blocks), `POWP-SmallNet` (id=5), `POWP-Recall` (id=6), and the activation-gated `POWPv3` (id=4).

### Architecture

```
ConsensusAuthority
├── ConsensusParams           # frozen dataclass: max_supply, target time, …
├── DifficultyPolicy          # next_target(node, params)
├── RewardSchedule            # block_reward(height, params), fee_for(tx, params)
├── ProposerSelectionPolicy   # vrf_seed / eligible_proposers
├── PledgePolicy              # validate_pledge / active_set
├── MiningPolicy              # produce_proof / verify_proof
├── BlockValidator            # verify_block(block, node, params)
└── ForkChoicePolicy          # compare_tips(a, b, ctx)
```

Authorities self-register at import time via `@register_consensus`. The Node loads plugin modules listed in `config.consensus.plugins` (dotted paths) at startup; this triggers the decorator and populates `AUTHORITY_CLASSES`.

### Era-tagged history

The `consensus_eras` table records every activation as `(activation_height, consensus_id, params_json, activation_txid, activated_by)`. `Node.consensus_at(h)` resolves the active authority via `bisect_right` over the era list in O(log n). Genesis inserts a row `(0, 1, default_params, NULL, 'genesis')` on first startup.

`BlockHeader` gained `consensus_id: int = 1` and `consensus_proof: bytes = b""` (and, later, the Proof-of-Access recall fields). For legacy POWP blocks (`consensus_id == 1 and consensus_proof == b""`) the hashing path serializes the legacy JSON dict verbatim, preserving byte-equality with existing chains. Non-default headers bind `consensus_id` into the PoW preimage; the `consensus_proof` itself is *excluded* from `hash_for_pow` — it is the search artifact (like the nonce), so including it would be circular.

### Owner-signed activation transactions

A new transaction type `consensus_activation` carries `{consensus_id, activation_height, params_json, owner_sig | owner_sigs, nonce}` in `tx.data`. `POWPBlockValidator` enforces:

1. The owner signature(s) verify under the configured owner key(s) — single-sig (`owner_sig`) or an M-of-N threshold (`owner_sigs` + `owner_threshold`).
2. `consensus_id` is registered in `AUTHORITY_CLASSES`.
3. `activation_height > block.height + 100` (SAFETY_WINDOW).
4. No other era row already has `activation_height > current_height` (no concurrent pending activation).
5. `params_json` parses into `ConsensusParams`.
6. `(owner_pubkey, nonce)` has not been used before — replay protection via `consensus_activation_nonces` table.

When a block containing such a tx commits, `Node._register_eras_from_block` inserts the era row and records the nonce. From `activation_height` onward, `consensus_at()` returns the new authority.

### Cross-era fork choice

The authority of the *higher* tip governs comparison. If two competing tips live in different eras, the later-era tip wins regardless of any per-authority tie-breakers. This is enforced as a short-circuit at the top of `Node._is_better_chain` before delegating to `authority.fork_choice.compare_tips`. The rule prevents an old-era fork from reorganizing across an activation height — once an era activates, the chain is committed.

### Activation flow

```
1. Owner constructs consensus_activation tx with N blocks of safety window.
2. Tx propagates to mempool.
3. Some miner includes it in a block; POWPBlockValidator runs all 6 rules.
4. Block commits → Node._register_eras_from_block writes consensus_eras row.
5. From activation_height onward, consensus_at() returns the new authority.
6. /status endpoint reports current consensus_id.
```

### Risks

- **Activation with no compatible miners** — chain halts at activation_height. Mitigation: SAFETY_WINDOW + the owner-signed `consensus_revert` tx (implemented; `scripts/sign_revert.py`).
- **Tx semantics across boundaries** — txs valid under POWP may behave differently under a new authority. Mitigation: keep txs consensus-agnostic except for `consensus_activation` itself; or include `consensus_id` in the tx sighash (future work).
- **Light clients** — out of scope; any future light-client format must include the era table.
- **Difficulty cadence change across an era boundary** — if a new era changes `target_block_time` or `difficulty_adjust_interval` (e.g. from 60s to 30s), the FIRST difficulty adjustment after the boundary computes `expected = new_target_block_time × adjust_interval` while `actual` spans blocks mined under the OLD target. This produces a one-shot step in difficulty bounded by the existing 0.25..4× cap. A 2× change in `target_block_time` will produce a roughly 2× step. Subsequent adjustments converge normally. Activation operators should expect a single transient difficulty jolt at the boundary; this is a known and bounded behavior rather than a bug. Mitigation if undesired: set `extra["skip_first_adjust"]=true` in the activating params and the authority skips the first post-boundary window — **implemented** (M3.1).

---

## Open Design Questions

1. **Pledge-weighted mining odds** — ~~should a larger pledge give slightly lower effective difficulty?~~ **Resolved: No.** Pledge affects fork-choice tiebreaker only; PoW odds remain purely compute-based. Coupling pledge to mining odds would reintroduce capital-favours-capital dynamics that POWP is designed to avoid.

2. **Opt-out for random reward** — some participants may not want to receive rewards
   (regulatory, privacy). Should eligibility be opt-in rather than automatic?

3. **Protocol treasury** — ~~a small percentage of each block reward to a governed fund for dev, node ops, and protocol upgrades.~~ **Resolved: No treasury.** All block rewards go to miners and VRF-selected participants. Can be revisited once the core protocol is stable.

4. **Pledge update policy** — can a miner change their pledge commitment after broadcasting it
   to peers but before the block is confirmed? Current design: no (pledge is committed into the
   block being mined).

5. **Initial distribution** — how the genesis supply is introduced into circulation is an
   open question and is **not yet designed**. No token is being offered, sold, or distributed,
   and none is available for purchase. Any future distribution scheme would be specified here,
   would be a utility allocation within a community rather than an investment offering, and
   would be subject to applicable law in the relevant jurisdictions.

6. **Dynamic pledge and fee adjustment** — the `fee_pledge_fluctuation_graph.py` simulation uses fixed sensitivity parameters (α = 0.0001, β = 0.0002). **Resolved: not adopted as the default.** It was prototyped as the `POWPv3` authority, but its per-block values derive from the validator's chain tip rather than the block's ancestry, which can split consensus — so `POWPv3` is **gated against activation**. The default keeps static, deterministic rules (see *Fee and Pledge Evolution*). A corrected version would commit the rate in the header (EIP-1559 style) or derive it from the block's ancestry.


---

## Protocol Notes

### Sync scope

By default every node is full-archival — it stores and serves the entire chain in SQLite, and
`node_type` controls only whether a node produces blocks, not whether it serves history.

A node can also run **light**: with `keep_blocks > 0` it prunes old block bodies (keeping all
headers) and fetches missing history on demand from archival "librarian" peers — part of the
Proof-of-Access / shared-history storage work. Full SPV-style clients (headers + Merkle proofs
only, no librarian fetch) remain a future optimisation.

### Sync verification paths

All three ingestion paths now verify blocks before writing them:

```
Gossip:      receive block → verify_block → _handle_fork → verify_chain → _switch_to_fork ✓
Bootstrap:   _bootstrap_sync_once → _paginated_sync → _attempt_chain_reorg → verify_chain → _switch_to_fork ✓
Gap repair:  _fill_gap / _force_repair_segment → verify_block per block → add/replace ✓
```

### Comparison with Bitcoin sync

Bitcoin uses several robustness techniques; Tesseracoin has adopted some and not others:

1. **Headers-first** — download headers and verify PoW before fetching bodies. *Partially
   adopted*: a `/headers/from` endpoint and headers-first light-node sync exist; verifying the
   full header chain before any body is fetched is not yet the default path for archival sync.
2. **Cumulative work** — canonical chain = most total accumulated PoW, not greatest height.
   *Not adopted*: Tesseracoin uses height as the primary tie-breaker, which is approximately
   correct but not cryptographically sound (two chains at the same height can have very
   different total work).
3. **Parallel block download** — fetch ranges from multiple peers simultaneously. *Not adopted.*
4. **Checkpoints** — hardcoded `(height, hash)` pairs that prevent reorgs past them. *Not adopted.*
5. **Peer scoring / banning** — misbehaviour accrues a score; crossing the threshold bans the
   peer for a cooldown. *Adopted* (`peer_scoring.py`, `PeerScoreTracker`).

### Cumulative work + cumulative pledge as fork choice

Replacing height-first fork choice with cumulative work + cumulative pledge (Step 5 on the
roadmap) would close the cryptographic soundness gap identified above. A tie-breaker is still
needed for chains with equal cumulative work and equal pledge — the current VRF seed →
block hash ladder suffices for that residual case.