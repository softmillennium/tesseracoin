# Tesseracoin documentation

Start here. These are the operator- and protocol-facing docs.

## Run a cluster
| Doc | What it covers |
|---|---|
| [OPERATOR_GUIDE.md](OPERATOR_GUIDE.md) | Turnkey path — stand up a private cluster for a small group in ~30 minutes |
| [DEPLOYMENT.md](DEPLOYMENT.md) | Deployment reference — Docker / Kubernetes / VM / bare-metal, sizing, storage, backups, security hardening |
| [TROUBLESHOOTING.md](TROUBLESHOOTING.md) | Common failures and their fixes |
| [MEMBER_GUIDE.md](MEMBER_GUIDE.md) | For people *using* a cluster — wallet, sending and receiving, participation |

## Understand the protocol
| Doc | What it covers |
|---|---|
| [DESIGN.md](DESIGN.md) | Full protocol design — POWP consensus, random rewards, fork choice, economic model, addresses & cryptography |
| [CONSENSUS_UPGRADES.md](CONSENSUS_UPGRADES.md) | How the rules change — owner-signed activations, the per-era authority registry, and which authorities are active vs gated |

## Set up a new network
The one-time genesis ceremony (owner keys, multisig threshold, signed genesis
record) is documented separately in
[`../genesis/GENESIS_CEREMONY.md`](../genesis/GENESIS_CEREMONY.md).

## Feedback & corrections

These docs are public on purpose — review, corrections, and design critique are
welcome. Tesseracoin is community-first; the design is meant to be argued with.

- **Found something wrong** — a stale parameter, a broken link, an out-of-date
  claim: open an [issue](https://github.com/softmillennium/tesseracoin/issues).
- **Design discussion or questions**: start a
  [discussion](https://github.com/softmillennium/tesseracoin/discussions) (or an
  issue, if Discussions isn't enabled on the repo).
- **A concrete fix**: a pull request with the change is the fastest path in —
  line-level review comments on the diff are welcome there too.
