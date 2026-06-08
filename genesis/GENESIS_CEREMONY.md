# Genesis ceremony

The genesis ceremony is what you run **once**, before your group ever
turns the cluster on. It produces three things:

1. **A multi-sig owner key set** — Dilithium2 private keys, one per
   designated owner, plus a public-key bundle the chain uses to
   authenticate consensus-changing actions.
2. **A fresh discovery secret** — the random token nodes use to find
   each other privately.
3. **A ceremony record** — a human-readable transcript of who
   participated and what was produced, suitable for the group's records.

## Who needs to participate

- **One operator** — the person who runs `./deploy.sh` and physically
  operates the cluster (you).
- **N owners** — the people who collectively govern the cluster's
  long-term consensus rules. Owners do NOT need to operate the cluster
  day-to-day; they only sign when something material changes.
- A **threshold** is set up-front. Any M-of-N subset of owners is
  enough to approve a change. Recommended: a simple majority
  (e.g. 2-of-3, 3-of-5).

If your group is just you and one or two friends, you can be the
operator AND one of the owners.

## The ritual

The cryptography is just running a script. The **social** parts are
where the trust comes from:

1. **Decide together, out of band.** Before running anything, agree
   by voice, video, or in person on:
   - The cluster name
   - The owner names + threshold
   - Who will be the operator
   - The **genesis era** — which consensus era the chain starts on
     (see "Choosing the genesis era" below)
2. **Run the ceremony script.** As operator:
   ```bash
   python genesis/ceremony.py
   # or non-interactive:
   python genesis/ceremony.py --batch \
       --cluster my-coop \
       --owners alice,bob,carol \
       --threshold 2
   ```
3. **Distribute each owner's private key file securely.** The
   `genesis/<cluster>/owner_<name>.key` files are sensitive — anyone
   holding `threshold` of them collectively controls the cluster's
   future consensus.

   Use a channel both sides trust:
   - In-person handoff (USB stick, QR code)
   - Signal / encrypted messenger
   - Encrypted email (PGP, age)

   **DO NOT** send via plain email, Slack, Discord, paste into a
   shared cloud doc, or commit to git.
4. **Verify the bundle hash.** Each owner opens their copy of
   `owner_pubkeys.json` and checks the `bundle_hash` field matches
   the value printed in `CEREMONY_RECORD.md`. Mismatched hashes mean
   someone tampered with a key.
5. **Operator destroys their copies** of the private keys after
   distribution. The operator keeps only `owner_pubkeys.json` (public,
   safe) and `CEREMONY_RECORD.md`:
   ```bash
   shred -u genesis/<cluster>/owner_*.key       # Linux
   # or on macOS:
   rm -P genesis/<cluster>/owner_*.key
   ```

After the ceremony, `./deploy.sh up` starts the cluster.

## Choosing the genesis era

The `GENESIS_ID` in `.env` sets which consensus era the chain's first
block uses. This is a **one-time decision** — changing it later requires
wiping the chain and re-running the ceremony. Future eras are graduated
into via a single on-chain activation transaction, not a re-genesis.

| GENESIS_ID | Era | Recommended when |
|-----------|-----|-----------------|
| `1` | `PurePoWAuthority` — Pure PoW, 100% coinbase to miner, no stake | **Default.** Start any network here. Any CPU can mine from block 1; no staking infrastructure needed. Activate era 2 later via on-chain tx. |
| `2` | `POWPStakeAuthority` — PoW + slashable stake + random reward | Start here only if the community is ready to stake from block 1 (staking wallets distributed, min\_stake threshold reachable immediately). |
| `6` | `POWPStakeBlockRecallAuthority` — era 2 + Proof-of-Access recall | For networks where storage accountability must be enforced from genesis. Requires at least `recall_min_depth=100` blocks before the first recall challenge engages. |
| `3`, `5`, `7` | SmallNet variants | **Dev and testing only.** Never deploy SmallNet parameters in production — stakes and penalty thresholds are deliberately weak. |

The typical production path is:

```
GENESIS_ID=1 (PurePoW)  →  activate era 2 (POWP-Stake)  →  activate era 6 (block recall)
                                                          →  activate era 4 (PoA)
```

Each `→` is a single `consensus_activation` transaction co-signed by the
founding multisig at a pre-announced block height — no software upgrade,
no hard fork, no coordination beyond the M-of-N signing ceremony.
See `docs/CONSENSUS_UPGRADES.md` for the full activation workflow.

## Trust model

The ceremony establishes that:

- The chain's **everyday operation** (mining, transactions, sync) does
  not require any owner. Anyone with a node can participate.
- The chain's **consensus rules** can only be changed by an activation
  transaction signed by `threshold` of `N` owners. No single owner can
  push a change unilaterally; no missing owner can block the others
  forever (unless they hold more than `N - threshold` keys collectively).
- The **operator** is trusted by the group to physically run the
  cluster honestly, but not to make consensus decisions. If the
  operator ever needs to be replaced, the owners can spin up the
  cluster elsewhere using the same `owner_pubkeys.json` and the same
  discovery secret, and the chain continues seamlessly.

## What this ceremony does NOT do

- It does **not** customise the chain's protocol parameters
  (cadence schedule, halving interval, supply cap, fee rate). Every
  Tesseracoin cluster runs the same consensus rules. What differs
  per cluster is **who governs**, **who runs nodes**, and **what
  identity material is in play** — not the rules themselves.
- It does **not** pre-allocate any coins to members. Members get coins
  through normal mining rewards and treasury transfers after the chain
  is running. (See `docs/TREASURY_GUIDE.md` for how a co-op handles
  shared funds.)
- It does **not** publish anything on-chain. The owner pubkeys take
  effect the first time the operator submits a consensus_activation
  transaction signed by the multi-sig — until then, the cluster runs
  with no special governance, suitable for a small private group.

## Re-running the ceremony

You generally **don't**. The owner key set is meant to be permanent
for the life of the cluster. If you need to add or remove an owner,
that's a consensus_activation transaction (signed by the existing
threshold of owners) — see `docs/OPERATOR_GUIDE.md`.

If you're starting completely over (wiped the chain data, fresh
genesis), pass `--force` to overwrite the existing cluster directory.
