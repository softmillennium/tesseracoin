# Consensus Upgrades — Operator Guide

How to roll out a new consensus version (e.g. tuning a parameter, adding a
new authority) across a running Tesseracoin network.

Audience: node administrators and the network's owner-key holder.

---

## What "a consensus version" is

Two parts:

| Part | Where it lives | Distribution mechanism |
|------|----------------|------------------------|
| **Authority code** (`ConsensusAuthority` subclass) | Python module on disk, e.g. `tesseracoin/consensus_powp_v2.py` | Out-of-band: bundle into node binary / package, deploy to every operator |
| **Authority params** (`ConsensusParams` instance) | `consensus_eras.params_json` row in each node's SQLite | On-chain via an owner-signed `consensus_activation` tx |

Think of it as: **code by deploy, params by transaction**. The chain has no
mechanism to ship Python code — every node must already have the authority
class installed before its `consensus_id` activates.

### Why classes, not JSON?

A reasonable question — *couldn't the authority itself be data?* The short
answer is no, and there's no good precedent for doing it that way. Concretely:

- **Expressiveness.** Consensus has to verify PoW (big-int arithmetic), check
  signatures across schemes (PQ + classical), iterate header chains, evaluate
  Merkle proofs. A JSON DSL strong enough to express all that is a programming
  language — designing one from scratch is a research project, not a feature.
- **Determinism.** Every node must compute the *exact same* answer bit-for-bit.
  Native typed code is easier to keep deterministic than an interpreter, and
  easier to audit (one layer instead of "spec + interpreter").
- **The trust gap is already closed cryptographically.** Activations carry the
  canonical `code_hash` of the authority module (see Step 2b). Validators
  reject the activation if their locally-loaded class doesn't match — so
  "every operator must trust the binary" isn't trust-by-faith; it's
  verifiable per-activation.

The trade-off of going code-based: a new authority needs a node software
upgrade before it can be activated. The off-chain announcement (Step 0) +
the code_hash attestation (Step 2b) make this a coordinated, verifiable
process rather than a leap of faith.

If on-chain code distribution ever becomes a goal, the Substrate-style
WASM-runtime model is the path — ship authority classes as bytecode in the
chain itself. That's a substantial engineering project; flag it as a future
direction, not today's surface.

### `BRINGS` — the plugin author's release-notes contract

A new authority subclass should declare what it adds, replaces, or consumes
vs the empty `ConsensusAuthority` base, as a `BRINGS` class attribute:

```python
@register_consensus
class MyShinyAuthority(POWPAuthority):
    """One-paragraph human description of this authority."""
    id = 8
    BRINGS = [
        "Replaces POWP's static reward with a stochastic reward draw",
        "Consumes params.extra['rng_seed_source']",
        "Inherits PoW, pledge, mempool from base POWP",
    ]
```

`BRINGS` is **release-notes prose, not a runtime contract** — nothing in
the validator checks it. The framework surfaces it via `/consensus/registered`
so explorers and operators can see at a glance what plugging in this
authority changes. Keep entries short; one bullet per behavioral change.

For machine-checkable claims (e.g. "this authority *actually* reads
`params.extra['rng_seed_source']`"), a separate, narrower contract is the
natural next step — `CONSUMES_EXTRA_FIELDS = [...]` with a startup-time
assertion. Not in place today.

---

## Authorities available today

| consensus_id | Class | What it does |
|--------------|-------|--------------|
| 1 | `POWPAuthority` (POWP) | Base Proof-of-Work + Pledge. Canonical owner of the shared `ConsensusState` (supply, difficulty target, etc.). |
| 2 | `PoAAuthority` (PoA) | Replaces PoW with owner-signed blocks. Requires `params.extra["owner_pubkey"]`. Disables PoW difficulty retargeting. |
| 3 | `POWPv2Authority` | Same code as POWP — kept registered so eras that activated under cid=3 remain validatable. Differences vs cid=1 live in the era's params. |
| 4 | `POWPv3Authority` ⚠️ **GATED — not activatable** | POWP with **time-evolving** fee + pledge dynamics (`fee_rate` compounds per block by `1 + α × (utilisation − 0.5)`; pledge ceiling by `1 + β × (initial − active)/initial`). **Refused by the activation validator** (`ACTIVATABLE = False`): its dynamic values read `node._recent_window` (the validator's tip) instead of the block's ancestry, so honest validators on different forks disagree on the same block's validity — a consensus split. The class stays registered + unit-tested; re-enable only after the value derives from the block's ancestry or is committed in the header (EIP-1559 style). See `tests/test_powp_v3_determinism.py`. |
| 5 | `POWPSmallNetAuthority` | Same code as POWP — relaxed gates live in the era's params (low `min_miners`, eager prune thresholds, tighter `difficulty_adjust_interval`). Used by the development sim. Genesis-able via `CONSENSUS_GENESIS_ID=5`. |
| 6 | `POWPRecallAuthority` | POWP + Proof-of-Access recall: miners must prove they hold older block bodies. Consumes `params.recall_enabled`, `params.recall_min_depth`. |
| 7 | `POWPRecallSmallNetAuthority` | Recall (as cid=6) plus the small-net params tuning (as cid=5). Genesis-able for chaos / sim runs that need both. |

Plus any authority shipped in an operator-supplied plugin module.

---

## Workflow: activating a new consensus version

### Step 0 — Announce intent

Before any signing happens, the owner(s) publish a public
announcement on the operator community's primary channel
(mailing list, forum, Discord, Slack — operator-community
choice; **what matters is the channel is the well-known one
the operators already follow**). The announcement is the
audit trail every node operator can read to decide whether
to upgrade their binary in time. It MUST contain:

- **Target `consensus_id`** — which authority is being activated.
- **Planned `activation_height`** — minimum `current_tip +
  SAFETY_WINDOW` (100); recommended is `current_tip + 200` or
  more so operators have buffer time.
- **Target date** — `activation_height − current_tip` blocks
  × the network's target block time, in human-readable form.
- **Binary release URL + `code_hash`** — the SHA-256 of the
  authority module that will run after activation. Operators
  verify their installed binary matches via `GET
  /authority-attestations` before signing (see Step 2b).
- **Signing schedule** — who's signing, when, M-of-N
  threshold (for multi-sig).
- **Review window** — how many days/blocks the announcement
  should remain public before the owner(s) start signing.
  Recommended: ≥ 7 days for production networks; shorter is
  fine for testnets.
- **Why** — the technical rationale, links to the change,
  any backward-compat notes.

**The chain itself can carry a pointer back to this
announcement.** The activation tx supports two optional
fields in `params.extra`:

```json
{
  "extra": {
    "intent_url": "https://forum.example.com/t/upgrade-discussion/42",
    "intent_summary": "POWPv2 — doubled fee_rate; 14-day review window",
    ...
  }
}
```

Both are validated for shape only — `intent_url` must start
with `http://` or `https://` and fit in 2 KB; `intent_summary`
≤ 200 chars. We never fetch the URL; the value is that
chain history is self-describing — future readers can follow
the link to read the rationale without grepping years of
forum archives. Both fields are OPTIONAL: activations
without them validate exactly as before.

Recommended announcement template:

```
Subject: [<network>] Intent to activate consensus_id=<N> at height <H>

Hi all,

I'm planning to sign an activation tx for consensus_id=<N>
(<authority class name>) at activation_height=<H>. Estimated
date: <YYYY-MM-DD> based on current tip <T> and target block
time <S>s.

Binary: <release URL>
code_hash: <64 hex chars>
Verify locally:  curl <node>/authority-attestations | jq .

Rationale: <link to change / forum thread / spec>
Multi-sig signers: <list> (threshold M of N)
Review window: <D> days; signing starts <YYYY-MM-DD>.

Operators: binaries must be upgraded BEFORE signing starts;
confirm with:  curl <node>/peers/consensus-ids
        # all peers should include <N> in their consensus_ids
        # and none should appear in missing_for_pending.

— <owner>
```

After the review window closes the owner proceeds to Step 1.

### Step 1 — Develop or pick the authority

For tuning POWP params (fee_rate, target_block_time, etc.), `POWPv2Authority`
is the canonical template:

```python
# tesseracoin/consensus_powp_v2.py — the whole class is 3 lines
from dataclasses import replace as _replace
from .consensus_api import register_consensus
from .consensus_powp import POWPAuthority, POWP_DEFAULT_PARAMS

@register_consensus
class POWPv2Authority(POWPAuthority):
    id = 3
```

For new mining or fork-choice logic, subclass `ConsensusAuthority` directly
and provide the relevant sub-policies (see `consensus_poa.py` as a worked
example).

`id` must be a unique integer across all authorities loaded on every node.
Each network reserves a range and documents it.

### Step 2 — Ship the code to every node

Add the dotted-path import to each node's `node_config.json`:

```json
"consensus": {
    "plugins": ["tesseracoin.consensus_poa", "tesseracoin.consensus_powp_v2"]
}
```

Then update + restart every node so the module is loaded at startup. The
`@register_consensus` decorator populates `AUTHORITY_CLASSES` on import.

**Verify every node has the code installed** before signing the activation.
With discovery, one curl answers cluster-wide:

```bash
curl -s http://localhost:8001/peers/consensus-ids | python -m json.tool
```

Response (excerpt):
```json
{
  "peers": {
    "tsr1...alice": [1, 2, 3, 4],
    "tsr1...bob":   [1, 2, 3, 4],
    "tsr1...carol": [1, 2],            // outdated — missing 3 and 4
    "tsr1...dan":   [1, 2, 3, 4]
  },
  "pending_consensus_ids": [4],
  "missing_for_pending": {
    "tsr1...carol": [4]                // ABORT: do not sign yet
  },
  "discovery_available": true
}
```

`missing_for_pending` is non-empty → any peer listed there will REJECT
the activation tx's block at `activation_height` and fork off silently.
Install the missing plugin + restart those nodes, re-check, then proceed.

Without a discovery service, per-node probes are the fallback:

```bash
for port in 8001 8002 8003 8004; do
    echo "node-$port: $(curl -s http://localhost:$port/authorities)"
done
```

Empty `consensus_ids` from a peer (in `/peers/consensus-ids`) means the
peer registered before this field was introduced — treat as "unknown",
not "empty"; confirm via the per-node `/authorities` fallback during
the upgrade window.

### Step 2b — (Optional but recommended) Collect the canonical code hash

Confirm every node carries identical bytes for the target authority,
then capture the hash to bake into the activation tx:

```bash
for port in 8001 8002 8003 8004; do
    echo "node-$port: $(curl -s http://localhost:$port/authority-attestations \
        | python -c "import sys, json; d=json.load(sys.stdin); print(d['attestations'].get('4', {}))")"
done
```

Every node must report the **same** `code_hash` for the target
`consensus_id`. Divergence means at least one node has a tampered or
stale binary — fix it before continuing. The canonical hash should
be recorded; it is set as `params.extra["code_hash"]` in Step 3 so
any node whose hash diverges later (or is tampered between now and
activation_height) rejects the activation tx automatically. Omitting
`code_hash` skips this defense — eras predating the attestation
feature continue to validate.

### Step 3 — Owner(s) sign the activation transaction

The activation can be signed in **single-sig** or **multi-sig** mode
depending on what the active era's params declare:

| Mode | `params.extra` fields | Tx wire format |
|------|------------------------|----------------|
| Single-sig (legacy) | `owner_pubkey: str` | `owner_sig: str` |
| Multi-sig (M-of-N)  | `owner_pubkeys: list[str]`, `owner_threshold: int` | `owner_sigs: list[str]` |

**Single-sig**:

```python
tx = node.build_consensus_activation_tx(
    consensus_id=3,
    activation_height=node._get_tip_height() + 200,
    params_json=new_params.to_json(),
    owner_wallet=node.wallet,
    nonce=1,
)
```

**Multi-sig (offline signing — recommended for production)**:

Each signer computes their signature on their **own machine** using
`scripts/sign_activation.py`. No private key ever leaves the signer's
host. They send the resulting hex string to the coordinator via any
channel they trust (PGP-encrypted email, Slack, etc.). The coordinator
assembles the final tx without ever holding anyone else's key.

On each signer's machine:

```bash
# Each signer runs this with THEIR OWN private key. Output is one
# hex line — the signature. Use the @PATH form so the key never
# touches shell history.
python scripts/sign_activation.py \
    --consensus-id 3 \
    --activation-height 250 \
    --params-json /path/to/new_params.json \
    --nonce 1 \
    --privkey-hex @/secure/signer_A.hex
4d3e2c1b...
```

`scripts/sign_revert.py` is the companion for revert txs.

All signers MUST agree on the EXACT byte string of `params.json` (the
canonical preimage is signed verbatim — no whitespace
normalisation). The coordinator distributes the params file ahead of
time and confirms its SHA-256 with each signer before they sign.

On the coordinator's node (Python REPL or script):

```python
# Collected from each signer via the chosen channel:
sig_a = "4d3e2c1b..."  # from signer A
sig_b = "9f8e7d6c..."  # from signer B
# The coordinator's own wallet wraps the outer tx; its pubkey must be
# in the era's owner set (any owner can be the originator).
tx = node.build_consensus_activation_tx(
    consensus_id=3,
    activation_height=node._get_tip_height() + 200,
    params_json=new_params_string,    # byte-identical to what signers used
    nonce=1,
    owner_sigs=[sig_a, sig_b],
    sender_wallet=node.wallet,        # coordinator's own wallet
)
# Submit via POST /tx on this or any peer node.
```

**Multi-sig (all keys on one node — dev / single-operator)**:

If a single operator legitimately holds all the keys (e.g. a dev
cluster), they can skip the offline-signing dance and pass `Wallet`
objects directly:

```python
tx = node.build_consensus_activation_tx(
    consensus_id=3,
    activation_height=node._get_tip_height() + 200,
    params_json=new_params.to_json(),
    owner_wallets=[wallet_a, wallet_b],   # ≥ threshold signers
    nonce=1,
)
```

This form should not appear in production deployments where signers
are distinct people.

Constraints enforced by the block validator:

- `activation_height >= current_tip + SAFETY_WINDOW (100)` — gives the
  cluster lead time to react.
- `consensus_id` must be a registered authority on the validating node
  (mempool gate rejects unknown ids immediately).
- `(representative_owner_pubkey, nonce)` must be unused — replay
  protection. The representative pubkey is the lexicographically
  lowest member of the era's owner set, so the slot is stable
  regardless of which signers contributed.
- At least `owner_threshold` signatures verify against DISTINCT
  pubkeys in the owner set. Same-signer-twice doesn't double-count;
  out-of-set signatures don't contribute.
- The tx's `sender_pubkey` must be IN the owner set (defence-in-depth
  spam gate — caught before signature verification).
- At most one consensus-* tx per block (atomic rule 4).

### Step 4 — Monitor propagation

The tx propagates through gossip, gets mined into a block (typically the
next one), commits to `consensus_eras`. Check from any node:

```bash
curl -s http://localhost:8001/pending-activations | python -m json.tool
```

```json
{
  "pending": [
    {
      "activation_height": 305,
      "consensus_id": 3,
      "params_json": "...",
      "activation_txid": "fc_a1b2c3...",
      "activated_by": "owner"
    }
  ]
}
```

Run the same against every node and confirm they ALL see the row before
`activation_height`. If a node's `/pending-activations` is empty, the tx
hasn't reached it yet (peering issue) — fix before activation lands.

### Step 5 — Tip crosses activation_height

When the chain reaches `activation_height`, the registry's
`consensus_at(activation_height)` switches to the new authority. Mining at
that height onward uses the new params. Verify on every node:

```bash
for port in 8001 8002 8003 8004; do
    curl -s http://localhost:$port/status | python -c \
        'import sys,json; d=json.load(sys.stdin); print(f"node-{d[\"name\"]}: id={d[\"consensus_id\"]} h={d[\"height\"]}")'
done
```

All four should report the new `consensus_id` once their tip is at or past
`activation_height`. If one diverges, see the rollback section below.

### Step 6 — `/pending-activations` empties out

Post-activation, the row is no longer "pending" (its `activation_height`
is now `≤ tip`). `/pending-activations` should return `{"pending": []}`
again.

---

## Cancelling a pending activation

If the activation was a mistake (wrong params, bad timing) and
`activation_height` hasn't yet been reached, the owner can revert it:

```python
revert_tx = node.build_consensus_revert_tx(
    target_txid=activation_tx_id,
    owner_wallet=node.wallet,
    nonce=2,  # next free nonce for this owner key
)
```

Submit like any other tx. Once mined, the registry drops the era row and
`/pending-activations` no longer lists it. The revert tx must commit
before `activation_height` arrives.

---

## Pre-activation operator checklist

Before signing an activation tx, the owner should confirm:

- [ ] Every node has the authority Python module installed
      (`/authorities` includes the new `consensus_id`).
- [ ] Every node is on the same chain
      (`/info`'s `genesis_params_hash` matches across all nodes).
- [ ] Every node is at a recent tip
      (`/tip` heights within a few blocks of each other).
- [ ] `activation_height` is at least `current_tip + 100` AND well in the
      future given current block cadence (don't activate during a known
      maintenance window).
- [ ] The new `params_json` has been reviewed for sanity (max_supply,
      halving, fee_rate, etc.) — once activated, only another owner-signed
      activation can change them.
- [ ] When the network has off-chain ops (monitoring, exchanges, indexers),
      coordinate the activation date with them. The `consensus_id` change
      is observable in every new block's header.

---

## Hardcoded checkpoints (deep-reorg defense)

Each operator's node maintains a list of `(height, hash)` checkpoints in
`node_config.json` under `consensus.checkpoints`. The validator refuses
any block at a checkpointed height whose hash doesn't match. This is
**not** on-chain — each operator maintains their own list — matching
Bitcoin's defensive-checkpoint model.

```json
"consensus": {
  "owner_pubkey": "...",
  "plugins": [...],
  "checkpoints": [
    {"height": 10000, "hash": "abc123..."},
    {"height": 50000, "hash": "def456..."}
  ]
}
```

**When to add a checkpoint**: after a milestone height (release boundary,
governance decision, large supply event) at which chain identity should be
locked past that point. New operators joining the network sync against the
canonical chain, then bake the checkpoint after the sync.

**When NOT to add**: don't checkpoint recent heights. Reorgs of a few
blocks are normal under PoW racing; checkpointing too aggressively causes
the node to reject legitimate forks.

**Operator workflow**:

1. Each operator picks heights and reads the canonical hash from `/block/height/{H}`:
   ```bash
   curl -s http://localhost:8001/block/height/10000 | python -c \
       'import sys, json; d = json.load(sys.stdin); print(d["header"]["hash"])'
   ```
2. Update `node_config.json` on each node and restart.
3. Cross-check by hitting `/checkpoints` on every node — they should all
   return the same list. A mismatch means one operator hasn't updated yet.

**Cross-check command**:

```bash
for port in 8001 8002 8003 8004; do
    echo "node-$port:"
    curl -s http://localhost:$port/checkpoints | python -m json.tool
done
```

**Failure mode**: a block at a checkpointed height with the wrong hash
raises `Block at checkpointed height N has hash X but operator
checkpoint requires Y` during validation. The block is rejected; if it
came from a peer's gossip, the peer is effectively trying to fork the
chain — operators should investigate.

## Peer misbehaviour scoring + banning

Each node independently scores peers that send invalid blocks/txs/
pledges and temporarily bans repeat offenders. Complement to on-chain
slashing: slashing punishes byzantine MINERS (valid-but-equivocating
blocks); peer scoring catches the simpler attack of flooding peers
with garbage that fails verification.

**Defaults** (configurable per-node via constructor, currently not
exposed in `node_config.json`):

| Event | Points |
|---|---|
| invalid_block | 20 |
| invalid_tx | 1 |
| invalid_pledge | 1 |

Ban threshold: 100 points (≈ 5 invalid blocks before ban).
Ban duration: 300 seconds (5 minutes).
Decay: every 60 seconds, all scores halve; entries below 1.0 are dropped.

**Operator visibility**: `GET /peer-scores` returns:

```json
{
  "scores": {"tsr1q...": 25.0, "tsr1q...": 5.5},
  "bans":   {"tsr1qbad...": 1716843200.5},
  "threshold": 100
}
```

Cross-check across nodes to confirm the same peer is being banned
cluster-wide. A peer high-scored on one node but unscored on others
suggests a targeted attack against that node.

**Banned peer behaviour**:

- Inbound: `/block`, `/tx`, `/pledge` POSTs from a banned peer's
  X-Origin return HTTP 403.
- Outbound: `Network.broadcast_*` skips banned peers as targets.
- Recovery: after `ban_duration` seconds the peer is automatically
  unbanned with a fresh zero score. Ephemeral mistakes don't
  accumulate forever.

**Direct user submissions** (no X-Origin header) bypass scoring —
we don't penalize our own clients for sending malformed data.

## How this compares to other chains

Useful framing for operators, reviewers, and people writing public material
about Tesseracoin. Two axes capture every chain's activation model:

1. **Who chooses the activation height?** — miners, client teams,
   token-holder governance, or a designated owner / governed set.
2. **Does the new code already live on every node, or is it distributed
   on-chain?** — installed out-of-band, or fetched via consensus.

| Chain | Who chooses | Code distribution | Activation trigger |
|-------|-------------|-------------------|--------------------|
| **Bitcoin** | Miners (BIP9/BIP8 signalling within a 2016-block retarget period; BIP8 adds a "Must Signal" deadline height) | Out-of-band (software release) | **Block height** (retarget boundary) |
| **Ethereum (pre-Merge)** | Client teams + community, hard-coded in chainconfig | Out-of-band | **Block number** |
| **Ethereum (post-Merge)** | Same; recent forks use a `*Time` field interpreted as "first block whose timestamp ≥ T" | Out-of-band | **Slot or first block with timestamp ≥ T** |
| **Cosmos SDK** | On-chain governance vote, target `upgrade_height` | Out-of-band (Cosmovisor automates binary swap) | **Block height** |
| **Substrate / Polkadot** | On-chain governance vote approves a new runtime WASM blob | **On-chain** (next block applies it) | **Block (immediately after governance passes)** |
| **Tezos** | Five-phase cycle-based on-chain governance | **On-chain** (Michelson-compiled OCaml) | **End of Adoption period (cycle boundary)** |
| **Tesseracoin** | Owner key(s) — single or M-of-N set, signed off-chain announcement + on-chain `consensus_activation` tx | Out-of-band, with on-chain `code_hash` attestation | **Block height** |

A common misconception: "some chains activate by *time*." Time is **never** the
real trigger. Even when a chain uses a `*Time` field, the underlying rule is
"the first block whose timestamp ≥ T applies the new rules" — which is still
block-conditioned. Block timestamps are advisory (miners can fudge them within
a tolerance window); block height is monotonic and deterministic, so consensus
hinges on it.

### Where Tesseracoin's activation plumbing is concretely better than Bitcoin's

You're in the same quadrant as Bitcoin (installed out-of-band,
activation-by-block-height) and that quadrant is the operationally mature
one — claiming to "beat Bitcoin at BIP9" isn't the right framing. The real
differentiators live elsewhere (PoCRR rewards, pledge, post-quantum sigs,
sound-money supply). But the activation plumbing **does** have concrete
improvements worth knowing:

| Concern | Bitcoin's answer | Tesseracoin (in repo) | Why it's better |
|---------|------------------|------------------------|-----------------|
| "Is the binary I run the canonical code?" | Trust the signed release; informal review. No on-chain claim. | Activation tx carries `code_hash` of the authority module. Validators reject if their local class hash differs. `/authority-attestations` exposes per-authority hashes for cluster-wide parity checks. | **Cryptographic, on-chain. A tampered binary fails at the network level, not in post-mortem.** |
| "What rules govern this block?" | Read the C++ client + the BIP list. No queryable answer. | `/consensus/eras` returns the full activation history with params and authority class metadata. `consensus_at(h)` is the canonical answer per height. | **Chain is self-describing.** |
| Parameter tuning (block size, fee rate, retarget interval, mempool TTL, …) | Compile-time constants. Any change = soft/hard fork + software release. | Per-era `ConsensusParams` data. Tunable via a `consensus_activation` tx — no new code, no software upgrade. | **Same model as Cosmos module params: data, not code.** |
| What's actually being activated | One rule at a time (segwit, taproot, …) via soft fork. Soft-fork-only because hard forks are politically toxic. | A complete authority bundle (validator + pledge + reward + mempool sub-policies) swapped atomically. | **Honest about the unit of change. Permits a model switch (POWP → PoA for sim, POWPv3 for dynamic fees) — Bitcoin can't.** |
| Operator warning | Informal (mailing lists, Twitter, "when miners start signalling") | Step 0: signed off-chain announcement ahead of the on-chain tx, with `code_hash` and target height. Operators have time to install + verify. | **Formal, signed, machine-readable.** |
| Cluster trust verification | Each operator audits independently. | `/peers/consensus-ids` + `/authority-attestations` let one operator confirm every node has matching hashes for every registered authority. | **Detects "one node is running a tampered authority" actively, not after divergence.** |

### Where Tesseracoin explicitly trades away vs Bitcoin

These are deliberate design choices, not omissions — call them out when
talking publicly so readers understand the model:

- **Activation chooser**: owner-signed vs miner-signalled. Faster + clearer,
  but more centralised at the activation step. Defensible for a
  community-first chain; not defensible for "trustless permissionless money."
  This is the choice baked in.
- **No on-chain governance vote**: Cosmos, Tezos, Substrate all have
  token-holder governance; Tesseracoin doesn't (yet). The off-chain
  announcement + M-of-N owner sigs are the substitute.
- **No rollback path**: same as Bitcoin — only roll-forward is supported,
  via another activation.
- **No on-chain code distribution**: a node missing the Python class for the
  active `consensus_id` is dead until its operator installs + restarts. The
  Substrate-style WASM-runtime model is the future answer if this becomes a
  pain point.

### The marketing-style framing (one sentence)

> Tesseracoin's activation model is Bitcoin's block-height-based,
> install-out-of-band approach **plus** on-chain code attestation, queryable
> era history, and consensus params as data — so the chain is self-describing,
> parameter tuning doesn't need a software release, and a tampered node is
> caught by the network instead of in a post-mortem. The trade-off is that
> activations are owner-driven, not miner-signalled — appropriate for a
> community-first chain.

---

## Things that don't exist (yet)

These are NOT in place today; operators should plan around them or wait
for future milestones:

- **Pledge-weighted activation governance.** M-of-N multi-sig works,
  but voting weight is flat per pubkey. A future milestone could
  weight signatures by recent pledge contribution so larger pledgers
  have proportionally more say.
- ~~**Peer-advertised supported_consensus_ids in discovery.**~~ ✅ Shipped.
  `GET /peers/consensus-ids` on any node returns the cluster-wide map
  sourced from discovery + flags any peer missing a pending activation's
  target id. See Step 2 above.
- ~~**Off-chain announcement channel.**~~ ✅ Shipped (Step 0
  above). The channel itself is operator-community choice
  (mailing list / Discord / forum), but the chain now carries
  optional `intent_url` + `intent_summary` fields in the
  activation tx's `params.extra` — chain history points back
  to the announcement, so the off-chain context is auditable
  without grepping forum archives.
- **On-chain code distribution.** A node missing the Python module for
  the active `consensus_id` is dead — operator must install + restart.
- ~~**Code attestation.**~~ ✅ Shipped. Owners can bake the canonical
  SHA-256 of the target authority module into the activation tx's
  `params.extra["code_hash"]`; every validator rejects the tx if its
  local module's hash doesn't match. `GET /authority-attestations`
  exposes each node's per-authority hash for pre-signing parity
  checks. Missing `code_hash` in params = no enforcement (back-compat
  with eras that predate the feature).

---

## Failure modes & remediation

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| One node's `/authorities` doesn't include `consensus_id=N` | Missing plugin entry in `node_config.json`, or restart not yet done | Add plugin path, restart, verify |
| `/peers/consensus-ids` shows a peer with empty `consensus_ids` | Peer registered before this field existed (legacy binary), or discovery payload was rejected | Restart the lagging peer against an up-to-date binary; verify via per-node `/authorities` fallback |
| `missing_for_pending` non-empty in `/peers/consensus-ids` | A pending activation targets an id one or more peers lack | Install the plugin on listed peers + restart; do NOT sign until the gap is empty |
| `/authority-attestations` shows divergent `code_hash` for the same `consensus_id` across nodes | At least one node was deployed with a tampered or stale authority module | Re-deploy the canonical binary on the divergent node; verify hash parity before signing any activation |
| Activation tx rejected with `code_hash MISMATCH` | Owner's published `params.extra.code_hash` doesn't match the local module's SHA-256 | Either the owner published the wrong hash (re-sign with the correct hash) OR the local binary is tampered (re-install the canonical module) |
| `/pending-activations` empty after activation tx submission | Tx didn't reach this node's mempool or didn't get mined yet | Check peering; wait for next block; resubmit if mempool TTL expired |
| Node's tip falls behind after `activation_height` | Node missing the authority code → rejects every post-activation block | Install missing module + restart; node will resync from peers |
| Cluster splits — some nodes on old id, some on new | Activation propagated unevenly + a partition; or a node ran ahead with stale code | Heal partitions; ensure all nodes have the new authority; lagging nodes will re-sync |
| Owner key compromised | No revocation primitive | Roll out a new genesis on a new chain (current network's identity is the genesis_params_hash, tied to that owner key) |

---

## Where to look for more

- **Code**: `tesseracoin/consensus_api.py` for the abstract layer; concrete
  authorities are `tesseracoin/consensus_{powp,poa,powp_v2}.py`.
- **Activation lifecycle internals**: `tesseracoin/consensus_registry.py`
  (ConsensusRegistry), `tesseracoin/persistence.py` (consensus_eras table),
  `tesseracoin/node.py` (`build_consensus_activation_tx`,
  `_register_eras_from_block`).
- **End-to-end test references**:
  - `tests/chaos/test_powp_v2_activation.py` — single-sig activation
    end-to-end against a 4-node cluster.
  - `tests/chaos/test_multisig_activation.py` — multi-sig activation
    end-to-end: 3 owner wallets with threshold 2, A and B sign
    offline, C wraps and submits, cluster-wide adoption verified.
    Reference for the production multi-sig flow.
  - `tests/chaos/test_pledge_renewal.py` — pledge-renewal decay
    under a miner outage. Cluster keeps mining when a miner goes
    dark for renewal_window+ blocks.
  - `tests/chaos/test_slashing_no_false_positives.py` — natural PoW
    fork racing (different-parent siblings, different-miner siblings)
    does NOT trigger equivocation slashing. Negative-test guard.
- **Replay protection & atomicity guarantees**: see `docs/DESIGN.md`
  §POWP and the M3 milestone notes in `TODO.md`.
