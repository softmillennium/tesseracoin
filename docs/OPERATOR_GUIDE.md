# Operator guide — Tesseracoin v0.1 turnkey

Audience: the person who actually runs the cluster for a small group.
Goal: a working private Tesseracoin instance for ~3-20 people in ≤30
minutes of focused work, end to end.

For a deeper, deployment-options reference (Docker / K8s / VM / bare
metal sizing) see [`DEPLOYMENT.md`](DEPLOYMENT.md). This guide is the
fast path through the v0.1 **turnkey package**.

---

## 1. What gets deployed

The turnkey package brings up one host running, in Docker:

```
                  ┌──────────────────────────────────────┐
                  │  nginx  (HTTPS, /dashboard /wallet)  │
                  └─────────────────┬────────────────────┘
                                    │ /api/discovery, /api/nodeN
   ┌────────┐         ┌─────────────┴─────────────┐
   │ browser│ ──────▶ │ discovery (Redis-backed)  │
   └────────┘         └─────────────┬─────────────┘
                                    │
                ┌───────────┬───────┼───────┬───────────┐
                │           │       │       │           │
            ┌───┴───┐   ┌───┴───┐ ┌─┴───┐ ┌─┴───┐   ┌───┴───┐
            │ node1 │   │ node2 │ │node3│ │node4│   │ node5 │
            │ miner │   │ miner │ │mine │ │ user│   │ user  │
            └───────┘   └───────┘ └─────┘ └─────┘   └───────┘
                                    │
                  ┌─────────────────┴────────────────────┐
                  │  monitor  (readiness watchdog)       │
                  └──────────────────────────────────────┘
```

Five Tesseracoin nodes (3 miners, 2 users), one discovery service, one
readiness watchdog, all behind a single nginx that serves the explorer
+ wallet from `/dashboard/` and `/wallet/` respectively.

Everything is one git repository, one `.env`, and one `deploy.sh`.

---

## 2. Host requirements

| Resource | Minimum | Recommended |
|---|---|---|
| OS | Linux or macOS with Docker Engine | Ubuntu 22.04 LTS or similar |
| CPU | 2 cores | 4 cores |
| RAM | 4 GB | 8 GB |
| Disk | 5 GB free, persistent | 50 GB SSD |
| Network | Outbound HTTPS for image pulls | Static IPv4 + DNS A record |
| Software | Docker 20.10+, docker compose v2, python3, openssl | + a real TLS certificate |
| Ports | 8080, 8443 (sim defaults) | 80, 443 (production) |

Any host that can run a small web service will run a Tesseracoin
cluster. A €5/month VPS is enough for a ~20-person community.

---

## 3. First-time setup (30 minutes)

### Step 1 — clone the repository

```bash
git clone https://github.com/softmillennium/tesseracoin.git
cd tesseracoin
```

### Step 2 — initialise the deployment

```bash
./deploy.sh init
```

This copies `.env.example` to `.env` and generates a fresh
`DISCOVERY_SECRET` (via `openssl rand -hex 32`). The new `.env`
file lists every other knob with sensible sim-friendly defaults and
inline comments explaining each.

### Step 3 — edit `.env` for the deployment

At minimum, set:

```ini
PUBLIC_HOSTNAME=club.example.com   # the domain or IP browsers reach
NGINX_HTTPS_PORT=443               # 8443 keeps sim defaults
SIM_MODE=0                         # 0 in production, 1 for local dev
GENESIS_ID=1                       # 1=PurePoW (recommended genesis), 2=POWP-Stake, 3=SmallNet; see §6 + CONSENSUS_UPGRADES.md for all eras
```

For a strictly-private (LAN-only) cluster, leaving `PUBLIC_HOSTNAME`
as a local IP is fine.

### Step 4 — run the genesis ceremony

```bash
python genesis/ceremony.py
# or non-interactive:
python genesis/ceremony.py --batch \
    --cluster my-coop \
    --owners alice,bob,carol \
    --threshold 2
```

This generates a Dilithium2 keypair per owner, a public-key bundle,
and a signed ceremony record. See [`genesis/GENESIS_CEREMONY.md`](../genesis/GENESIS_CEREMONY.md)
for the full social ritual — owner key distribution should be done
out of band (Signal, in person, encrypted email), and the operator
should destroy their copies of the private keys after distribution.

### Step 5 — validate the configuration

```bash
./deploy.sh check
```

Reports any hard errors (block startup) and warnings (proceed but
review). A clean `.env` shows the cluster identity summary; warnings
typically indicate a sim-vs-production mismatch worth thinking about.

### Step 6 — start the cluster

```bash
./deploy.sh up
```

This builds the node image (~2 minutes first time), starts every
container, prints a status summary, then shows each node's current
tip height once the chain is running.

No TLS cert ships in the repo. On first `up`, `deploy.sh` generates a
local self-signed pair (`nginx/ssl/selfsigned.{crt,key}`) so nginx can
start — browsers will warn until it is replaced with a real CA cert
(see §7). To generate it ahead of time, or for a specific hostname:

```bash
CERT_CN=club.example.com ./nginx/generate_cert.sh
```

**First-time miners need a few minutes** to bootstrap the chain
before the first block lands. Watch `./deploy.sh status` to see tips
appear.

### Step 7 — verify everything

| Check | URL |
|---|---|
| Explorer (live block-by-block view) | `https://<host>:<port>/dashboard/` |
| Wallet (for end users) | `https://<host>:<port>/wallet/` |
| Per-node info | `https://<host>:<port>/api/node1/info` |
| Cluster health | `./deploy.sh status` |
| Readiness watchdog | `docker logs -f tesseracoin-monitor` |

A healthy cluster shows `SOAK [HEALTHY]` lines every 5 minutes in
the monitor log.

---

## 4. Day-to-day operations

### Check status

```bash
./deploy.sh status
```

Lists running containers + the tip height each node currently sees.
Heights should match (allow a 1-block lag during active mining).

### View logs

```bash
./deploy.sh logs              # all services, follow
./deploy.sh logs node1        # one service
./deploy.sh logs monitor      # readiness watchdog only
```

### Restart a single node

```bash
docker compose restart node1
```

Headers-first sync re-catches up automatically. The other nodes keep
serving while one restarts.

### Stop the cluster

```bash
./deploy.sh down
```

Data is preserved in `./data/`. Bring it back with `./deploy.sh up`.

### Upgrade the node software

```bash
git pull
./deploy.sh down
./deploy.sh up           # rebuilds the image, restarts nodes
```

Version skew within a few commits is normally tolerated. For any
upgrade that changes consensus, see §6.

---

## 5. Storage and backups

Chain data lives under `./data/node{1..5}/`. The most-important file
per node is `chain.db` (SQLite, WAL mode). Discovery state lives under
`./data/discovery/` and is reconstructible from the nodes if lost.

A clean backup is taken while the cluster is stopped:

```bash
./deploy.sh down
tar czf tesseracoin-backup-$(date +%Y%m%d).tar.gz data/ genesis/ .env
./deploy.sh up
```

Snapshot a running cluster with `sqlite3 chain.db .dump` if downtime
is unacceptable; restoring from a hot snapshot requires more care
(see [`DEPLOYMENT.md`](DEPLOYMENT.md) §7 for the canonical procedure).

---

## 6. Consensus upgrades

The chain's consensus rules can be changed via a signed activation
transaction at a future height. The owner multisig (set up during the
genesis ceremony) is what authorises this.

The current authority-class set:

| ID | Class | What it brings |
|---|---|---|
| 1 | `PurePoWAuthority` | Pure PoW genesis — SHA-256, 100% coinbase to miner, no stake. Recommended starting era for new networks. |
| 2 | `POWPStakeAuthority` | PoW + locked/slashable stake + stake-weighted random reward + demand-responsive base fee with burn. Standard production era. |
| 3 | `POWPStakeSmallNetAuthority` | Same as id=2 with relaxed params (min\_stake=1 TESC, unbonding=50 blocks). Dev/testing only — never production. |
| 4 | `PoAAuthority` | Owner-signed blocks, PoW disabled. Activation-only, not genesis-able. For governance milestones or consortium use. |
| 5 | `POWPStakeBlockRecallSmallNetAuthority` | id=3 + Proof-of-Access block recall (depth 30). Dev/testing only. |
| 6 | `POWPStakeBlockRecallAuthority` | id=2 + Proof-of-Access block recall (depth 100). Production recall era — miners must prove they hold historical block bodies. |
| 7 | `POWBlockRecallSmallNetAuthority` | SmallNet PoW + recall, no stake. For testing recall mechanics in isolation from staking. |

The activation tooling itself (signing + submitting an activation tx
with M-of-N owner signatures) is part of v0.2 — the v0.1 path is to
leave the cluster on its initial GENESIS_ID and only revisit when
the activation CLI ships. See [`CONSENSUS_UPGRADES.md`](CONSENSUS_UPGRADES.md)
for the underlying mechanism.

---

## 7. Security hardening before going public

The sim defaults are intentionally permissive. Before exposing the
cluster to the public internet:

| Setting | Sim default | Production target |
|---|---|---|
| `DISCOVERY_SECRET` | `changeme` (refused unless `SIM_MODE=1`) | `openssl rand -hex 32` |
| `TESSERACOIN_ALLOW_INSECURE_DISCOVERY` | `1` (allows `changeme`) | unset / `0` |
| `DEBUG_MODE` | `true` (debugpy on 568x) | `false` (ports not exposed) |
| `DEBUG_SECRET` | `sim-secret` | unset OR a real secret OR remove the port |
| `MINER_CPU_LIMIT` | `0.3` (dev power saver) | blank (uncapped) |
| `GENESIS_ID` | `3` (SmallNet, relaxed) | `1` (PurePoW genesis), then activate `2` via on-chain tx when staking is ready |
| nginx SSL certs | Self-signed `nginx/ssl/selfsigned.*` (auto-generated) | Let's Encrypt / real CA |
| `TESSERACOIN_CORS_ORIGINS` | `*` (any origin reads the API) | explicit allow-list |

`./deploy.sh check` warns on each of these.

For external exposure, put nginx behind a real TLS terminator
(Cloudflare proxy, Caddy, Traefik) and never expose the debug ports
(5681-5685) to the internet.

---

## 8. Common failures

See [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md) for the full list. The
short version:

| Symptom | Likely cause |
|---|---|
| `./deploy.sh up` fails immediately | Docker not running / port collision on 8080/8443 |
| Nodes never finish bootstrapping | Discovery secret mismatch between nodes |
| One node's tip lags forever | That node was offline > `peer_validation_interval`; restart it |
| Wallet shows `unreachable` | `PUBLIC_HOSTNAME` in `.env` doesn't resolve from the browser |
| Monitor logs `SOAK [FAILED]` once and stays failed | `violations_total` is cumulative for the monitor process; restart `tesseracoin-monitor` to reset |

---

## 9. Decommissioning

To shut down and remove everything (including chain data):

```bash
./deploy.sh wipe        # requires typing the cluster name to confirm
```

To keep the chain data but stop the containers indefinitely:

```bash
./deploy.sh down
```
