# Protocol Design

## Overview

Tesseracoin implements **POWP-Stake** (Proof-of-Work + locked, slashable Stake), a hybrid consensus combining:

- **PoW mining**: SHA-256 puzzle solving with **LWMA-1 per-block difficulty retargeting** (every era); no fixed-interval step retarget
- **Stake layer**: Miners lock real on-chain balance via `stake` and `unstake` transactions, subject to an unbonding window. Equivocation burns the offender's entire locked and unbonding stake and permanently tombstones the address.
- **Deterministic stake-weighted random reward**: A slice of the block reward goes to a participant other than the miner, selected by `sha256(prev_hash + stake_snapshot)` over all stakers above `min_stake`, weighted by locked stake, excluding the current producer.
- **Demand-responsive base fee**: Each block header commits a `base_fee_rate` derived from the parent header's rate and the block's fill ratio. Half of collected base fees are burned from supply; the rest goes to the miner. Tips go entirely to the miner and are the priority lever for inclusion in full blocks.
- **Fork choice**: Multi-layer tie-breaker — height → cumulative work → active locked stake of divergent producers → tiebreak seed → hash
- **Slashing**: Equivocation evidence in a block burns the offender's entire locked and unbonding stake and tombstones the address; the whistleblower receives 1% of the burned amount.

---

## POWP-Stake: Proof-of-Work + Locked Stake

Pure PoW (e.g., Bitcoin) grants influence proportional to hash rate. Proof-of-Stake requires no work but grants influence proportional to capital.

POWP-Stake bridges both — PoW provides Sybil resistance while locked, slashable stake creates real economic accountability.

### How a block is made

1. **Mine** — find a nonce such that `sha256(header_bytes)` meets the current target.
2. **Lock stake** — maintain an active `stake` transaction on-chain; the locked amount participates in fork-choice tiebreaking.
3. **Earn rewards** — the block reward splits: coinbase goes to the miner; a deterministic stake-weighted random reward goes to a randomly selected active staker (excluding the miner).

### Fork Choice

When two chains have the same PoW difficulty, the protocol breaks ties in this order:

| Priority | Rule |
|----------|------|
| 1 | Chain height (longer chain wins) |
| 2 | Cumulative PoW work |
| 3 | Active locked stake of the divergent block producers |
| 4 | Tiebreak seed (`sha256(prev_hash + nonce)`) |
| 5 | Block hash |

### Stake Mechanics

- Miners lock spendable balance with a `stake` transaction
- `unstake` queues a withdrawal behind an unbonding window (default 1,000 blocks)
- Locked stake participates in fork-choice tiebreaking and random reward eligibility
- Equivocation (signing two blocks at the same height) triggers slashing: all locked and unbonding stake is burned and the address is permanently tombstoned
- A whistleblower who submits equivocation evidence receives 1% of the burned stake as a reward

---

## Random Reward

Each block includes a deterministic stake-weighted random reward:

```
winner = sha256(prev_block_hash + stake_snapshot)
```

over all stakers above `min_stake` (default 25 TESC), excluding the current block producer. Every validator recomputes the winner identically; a block paying the wrong address is rejected. When no eligible staker exists, the slice is not minted.

Reward split per block:

| Recipient | Amount |
|-----------|--------|
| Miner | Coinbase reward (block subsidy + base fee share + tips) |
| Random reward recipient | Up to 10% of block reward, stake-weighted selection |

---

## Base Fee

The base fee is demand-responsive:

- Each block header commits a `base_fee_rate` derived purely from the parent header's rate and the block's fill ratio
- All validators agree on the rate because it derives from the block's ancestry, not from any per-node chain-tip state
- Half of collected base fees are burned from supply
- The other half goes to the miner
- Tips (amounts above the base fee specified by the sender) go entirely to the miner and are the mechanism for priority inclusion in full blocks

---

## Mining Flow

```
1. Build mempool snapshot
   ├── Select transactions (up to MAX_BLOCK_TXS)
   └── Include stake/unstake transactions if valid

2. Assemble BlockHeader
   ├── prev_hash, merkle_root, target, timestamp
   └── base_fee_rate (from parent)

3. Mine PoW
   └── Iterate nonce until sha256(header_bytes) meets target

4. Compute random reward recipient
   └── sha256(prev_hash + stake_snapshot) → stake-weighted selection

5. Assemble and broadcast block
   └── Verify locally before broadcasting
```

---

## Transaction Types

| Type | Purpose | Gossip-relay |
|------|---------|-------------|
| `transfer` | Send TESC between addresses | Yes |
| `coinbase` | Block reward, minted by miner | No |
| `stake` | Lock balance as validator stake | Yes |
| `unstake` | Queue stake withdrawal (unbonding window) | Yes |
| `slash` | Burn equivocating miner's stake | Yes |
| `random_reward` | Protocol-minted reward to stake-weighted winner | No |
| `consensus_activation` | Switch consensus era (multisig) | Yes |

---

## Addresses and Cryptography

All addresses are bech32m-encoded with the human-readable prefix `tesc`. The address encodes both the public key and the signature scheme:

- **Ed25519** — default; fast, compact signatures
- **secp256k1** — Bitcoin-compatible; supported for ecosystem tooling
- **Dilithium2 / FIPS 204 ML-DSA-44** — post-quantum; production-ready at genesis

All three schemes coexist on one chain. The address encodes which scheme was used, so validators always know which verification function to apply.

Transaction signing is pluggable via `tesseracoin/sigschemes.py`. A `chain_id` (670210) is bound into every signing payload, preventing cross-chain replay.

---

## Swappable Consensus Architecture

Tesseracoin's consensus rules live behind a plugin layer (`tesseracoin/consensus_api.py`):

```
ConsensusAuthority
├── DifficultyPolicy       # next_target (LWMA-1 per-block via expected_target)
├── RewardSchedule         # coinbase / random_reward / base_fee_rate
├── ProposerPolicy         # validate_proposer / active_set
├── StakePolicy            # validate_stake / validate_unstake / slashing
├── MiningPolicy           # hash_for_pow / verify_pow
├── BlockValidator         # verify_block (full validation pipeline)
└── ForkChoicePolicy       # compare_chains
```

Each authority bundle is registered with `@register_consensus(id=N)` and activated via an on-chain `consensus_activation` transaction co-signed by the founding multisig. Swapping consensus requires no hard fork and no coordinated software upgrade.

### Registered eras

Four registered consensus families — the only valid `consensus_id` values:

| id | Authority | Purpose |
|----|-----------|--------|
| 1 | `PurePoWAuthority` | SHA-256 PoW only; full coinbase to miner; no stake layer. Phase 1 genesis era. Genesis-able. |
| 2 | `POWPStakeAuthority` | PoW + locked slashable stake + stake-weighted random reward + demand-responsive base fee. Production era. Genesis-able. |
| 4 | `PoAAuthority` | Owner-signed blocks; no PoW. For governance milestones or consortium deployment. Genesis-able, or reached via activation. |
| 8 | `PoSAuthority` | Pure Proof-of-Stake — slot-based deterministic stake-weighted leader election; no mining. Delegation pools, optional inactivity eviction, top-N active set. Genesis-able. |

### Parameter profiles (SmallNet and block recall)

SmallNet (relaxed gates and fast cadence) and block recall (Proof-of-Access storage accountability) are not distinct eras — they are parameter profiles layered on a family, selected at genesis via the env `CONSENSUS_GENESIS_PROFILE=<name>`:

| Profile | Family + params |
|---------|-----------------|
| `powps-smallnet` | id 2 + SmallNet params |
| `powps-recall-smallnet` | id 2 + recall + SmallNet |
| `powps-recall` | id 2 + recall (production params) |
| `pow-recall-smallnet` | id 1 + recall + SmallNet |

---

## Proof-of-Access Block Recall (recall profiles)

When a recall profile is active, each miner must prove it holds the body of a specific past block:

```
recall_height = f(prev_block_hash, nonce, tip_height, activation_height, recall_min_depth)
```

The miner embeds a `recall_tx_id` into the PoW hash preimage. The winning nonce commits both a valid proof-of-work and a valid recall answer. On winning, a merkle proof of the answer's position in the recalled block is attached.

A node that has discarded old block bodies cannot produce a valid block for that nonce. Its effective hash rate is proportional to the fraction of the eligible range it stores, creating a direct economic incentive to archive.

---

## Peer Discovery and Storage

### Peer sources

| Source | When used |
|--------|----------|
| `PEERS` env var | Static seeds; always loaded at startup |
| `DATA_DIR/peers.json` | Persisted confirmed peer URLs; loaded at `Network.__init__` |
| Discovery service | Optional rendezvous server; peers registered and fetched on startup |

### On-disk peer store

`peers.json` is a flat JSON array of confirmed peer URLs, capped at 100 entries (`_PEER_STORE_CAP`). It is written when the discovery service adds a new confirmed peer or when the gossip crawl discovers and confirms a new peer. It is loaded at `Network.__init__` time and seeds `self.peers` before any network contact is made.

### Gossip crawl

A background thread (`net-gossip-crawl`) wakes after a 15-second initial delay, then every 60 seconds:

1. Snapshot the current peer list.
2. For each peer, fetch `GET /peers`.
3. For each URL in the response that is not already known and not the local node's own URL:
   a. Call `_try_add_peer(url, max_peers)` — handshake via `GET /info`, verify genesis params hash, then insert into `self.peers` if all succeed.
4. If any peers were added, write `peers.json`.

The gossip crawl is the mechanism by which a node seeded with only one known peer can discover the rest of the network without a discovery service.

---

## Open Questions

1. **Sybil resistance in peer discovery** — the peer store and gossip crawl have no subnet diversity enforcement. An adversary controlling the discovery service can eclipse a newly joining node by populating its peer list exclusively with adversary-controlled nodes. Countermeasure: bucket peers by /16 and cap per-bucket entries.

2. **Random reward distribution** — stake-weighted random reward is regressive: larger stakers receive proportionally more. Alternative: uniform random selection over all stakers above the minimum threshold, regardless of stake size.

3. **Era activation governance** — the founding multisig approach is simple but centralised. Stake-weighted voting (where activation requires a threshold of locked stake weight, not just M-of-N key holders) would distribute governance more broadly.

(Resolved since the first draft: **librarian incentives** — archival/librarian nodes now earn a direct reward. In the recall profiles and Pure PoS, the burned base-fee half is redirected to the stake-bonded librarians, ∝ their storage weight, accrued and claimed via `withdraw_rewards`.)
