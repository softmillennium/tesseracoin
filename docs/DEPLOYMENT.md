# Tesseracoin — Deployment & Operations Guide

Practical guidance for running Tesseracoin nodes in the real world: machine
sizing, and how to deploy via Docker, Kubernetes, a VM, or bare metal.

> **Status:** Tesseracoin is an advanced, chaos-tested **prototype**, not a
> formally audited production system. Start with a small, trusted community
> deployment, watch it, and grow with confidence. Never expose a node's HTTP
> API or the debug port to the public internet (see [Security](#8-security-hardening)).

---

## 1. What you are deploying

One OS process = one **node**. A deployment is a set of nodes plus one
shared **discovery** service:

| Component | What it is | Image base | Listens on |
|---|---|---|---|
| **Node** | wallet + chain DB + mempool + mining/sync loops + HTTP API | `python:3.12-slim` (builds liboqs 0.14.0 for post-quantum sigs) | `8000` HTTP API (`5678` debugpy — dev only) |
| **Discovery** | peer rendezvous (register / heartbeat / list); FastAPI | `python:3.13-slim` (+ Redis) | `8000` HTTP (auth via `X-Auth-Token`) |
| **Redis** | discovery's registry store (TTL-pruned) | — | `6379` (bundled in the discovery image, or external) |

Nodes find each other through discovery: each registers on startup,
heartbeats to stay live, waits for `min_peers`, then syncs and runs. There
is **no single point of consensus** — discovery only does introductions; if
it goes down, existing nodes keep validating, only new joins are affected.

Each node holds its own SQLite database (WAL mode) under `DATA_DIR` —
**this must be on persistent storage.**

---

## 2. Machine sizing

Community scale is light: 60-second blocks, ≤10 transactions/block. The only
CPU-intensive role is mining (SHA-256 proof-of-work, ~1 core busy). Disk is
the variable that grows over time — see [§7 Storage](#7-storage--volumes).

| Role | vCPU | RAM | Disk | Notes |
|---|---|---|---|---|
| **Discovery + Redis** | 1 | 0.5–1 GB | 1 GB | one per network; tiny, stateless-ish (Redis TTL-pruned) |
| **Miner** | 2 (≥1 dedicated) | 2–4 GB | SSD, sized for growth | ~1 core continuously on PoW |
| **Full archival / librarian** | 2 | 4 GB | larger SSD (full history) | retains all bodies; serves them to pruning peers |
| **Sharded miner** | 2 | 2–4 GB | ~1/N of full history | mines at a rate ∝ stored fraction |
| **Light / SPV (user)** | 1 | 1–2 GB | small (headers only) | no mining; verifies via Merkle proofs |

- **Disk growth** (full node, ~5 tx/block): ≈ **1.3 GB/year** with classical
  signatures, ≈ **14.5 GB/year** under post-quantum (Dilithium). Control it
  with `TESSERACOIN_KEEP_BLOCKS` (suffix pruning) or shard retention — see
  `docs/Tesseracoin_proof_of_access_storage.docx` for the full model.
- **Bandwidth** is modest at community scale (one small block per minute plus
  gossip); any commodity link suffices.

---

## 3. Docker / Docker Compose (recommended starting point)

The repo ships a working multi-node `docker-compose.yaml` (4 miners/users +
a passive node + simulator + discovery). For a real deployment, treat it as
a template and harden it:

```bash
docker compose up --build          # bring up the local cluster
curl -s localhost:8001/status | python -m json.tool
```

**Production checklist when adapting the compose:**
- Set a real `DISCOVERY_SECRET` (not `changeme`) on discovery **and** every node.
- Set `CONSENSUS_GENESIS_ID=1` for production POWP (the dev compose uses `5`,
  the relaxed small-net gate, for convenience). Wipe data dirs to re-genesis.
- **Remove the debug port** (`5678` / `DEBUG=true` / `DEBUG_PORT`) — never
  expose debugpy.
- Use **named volumes** or mounted managed disks for each node's `/data`.
- Run **Redis as its own managed service** for the discovery registry and
  point discovery at it via `REDIS_HOST` / `REDIS_PORT` / `REDIS_DB` (the
  bundled in-container Redis is fine for dev, not for durable production).
- Put a TLS-terminating reverse proxy in front of any API you must expose.

---

## 4. Kubernetes

Map each role to the right primitive — nodes are **stateful, identity-bound**
(each has its own wallet + chain DB), so they are NOT interchangeable
replicas.

- **Nodes → `StatefulSet`** with a `volumeClaimTemplate` (one
  `ReadWriteOnce` PVC per pod). The stable pod identity gives each node a
  durable wallet/DB. One node per pod.
- **Discovery → `Deployment` + `Service`** (it's stateless given an external
  Redis). Run **Redis** as its own `StatefulSet`/PVC or a managed cloud Redis,
  and set `REDIS_HOST`/`REDIS_PORT` on discovery.
- **Service DNS:** a headless `Service` lets nodes resolve each other;
  `DISCOVERY_URL` points at the discovery `Service`.
- **Config & secrets:** `node_config.json` via `ConfigMap`; `DISCOVERY_SECRET`
  and any consensus owner keys via `Secret`.
- **Probes:** liveness/readiness on `GET /status` (the image already exposes it).
- **Resources:** set CPU `requests`≈`limits` for miners (PoW is steady-state
  CPU) so they aren't throttled; size PVCs per §2/§5.
- **Scaling:** do NOT use an HPA on nodes — they aren't stateless. To add
  capacity, add `StatefulSet` members (each gets a distinct identity) or
  separate miner/librarian/light StatefulSets. SQLite ⇒ RWO volumes only;
  never share one PVC between pods.

Sketch (one miner StatefulSet member):

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: tesseracoin-miner }
spec:
  serviceName: tesseracoin
  replicas: 3
  template:
    spec:
      containers:
        - name: node
          image: tesseracoin-node:latest
          env:
            - { name: NODE_TYPE, value: miner }
            - { name: DATA_DIR, value: /data }
            - { name: DISCOVERY_URL, value: http://tesseracoin-discovery:8000 }
            - { name: CONSENSUS_GENESIS_ID, value: "1" }
            - name: DISCOVERY_SECRET
              valueFrom: { secretKeyRef: { name: tesseracoin, key: discovery-secret } }
          ports: [ { containerPort: 8000 } ]
          readinessProbe: { httpGet: { path: /status, port: 8000 } }
          volumeMounts: [ { name: data, mountPath: /data } ]
  volumeClaimTemplates:
    - metadata: { name: data }
      spec: { accessModes: [ReadWriteOnce], resources: { requests: { storage: 50Gi } } }
```

---

## 5. VM or bare metal

Good for a dedicated miner (give it a full core for PoW) or a long-lived
archival/librarian node.

1. **OS deps + liboqs.** `liboqs-python` is a hard dependency, so the C
   library must be present. Build liboqs **0.14.0 with shared libs** (the
   exact steps are in the repo `Dockerfile`), then `ldconfig`.
2. **Python.** 3.10+ (the image uses 3.12). Create a venv and
   `pip install -r requirements.txt`.
3. **Data dir.** Point `DATA_DIR` at a dedicated persistent disk/SSD.
4. **Run under systemd** so it restarts on crash/reboot:

```ini
# /etc/systemd/system/tesseracoin-node.service
[Unit]
Description=Tesseracoin node
After=network-online.target
[Service]
User=tesseracoin
WorkingDirectory=/opt/tesseracoin
Environment=PYTHONPATH=/opt/tesseracoin DATA_DIR=/var/lib/tesseracoin \
            DISCOVERY_URL=https://discovery.example:8000 NODE_TYPE=miner \
            CONSENSUS_GENESIS_ID=1
EnvironmentFile=/etc/tesseracoin/secrets.env   # DISCOVERY_SECRET=...
ExecStart=/opt/tesseracoin/.venv/bin/python scripts/run_node.py
Restart=always
RestartSec=5
[Install]
WantedBy=multi-user.target
```

5. **Front with a reverse proxy** (nginx/caddy) for TLS if the API must be
   reachable; otherwise bind it to localhost / a private network only.

---

## 6. Configuration

Settings come from `node_config.json` (copied to `NODE_CONFIG_PATH`);
**environment variables override the file**, which is how you parameterise
per-node deployments. Key knobs:

| Env var | Purpose |
|---|---|
| `NODE_TYPE` | `miner` or `user` |
| `DATA_DIR` / `NODE_CONFIG_PATH` | persistent data dir / config file location |
| `DISCOVERY_URL` / `DISCOVERY_SECRET` | discovery endpoint + auth token |
| `CONSENSUS_GENESIS_ID` | genesis era (`1` = production POWP; `5`/`7` = relaxed small-net / recall) |
| `CONSENSUS_PLUGINS` | JSON list of consensus plugin modules to load |
| `TESSERACOIN_SIGNATURE_SCHEME` | new-wallet signature scheme (ed25519 / secp256k1 / dilithium2) |
| `TESSERACOIN_KEEP_BLOCKS` | light-node body pruning depth (0 = full archival) |
| `TESSERACOIN_SHARD_MODULUS` / `_RESIDUE` | modular shard retention |
| `TESSERACOIN_ROLE` | advertised archival role (`full` / `librarian` / `light`) |
| `CONSENSUS_OWNER_PUBKEY(S)` / `_THRESHOLD` | consensus-governance owner / M-of-N set |
| `LOG_LEVEL` | log verbosity |

---

## 7. Storage & volumes

- Each node's SQLite DB (WAL) lives under `DATA_DIR` — **mount persistent
  storage** (a PVC, managed disk, or dedicated SSD). Losing it loses that
  node's identity + local chain (it can re-sync, but a miner loses its wallet).
- WAL means you'll see `*.db`, `*.db-wal`, `*.db-shm` — back up consistently
  (snapshot the volume, or copy with the node stopped / via SQLite backup API).
- **Control growth** with `TESSERACOIN_KEEP_BLOCKS` (suffix pruning) or shard
  retention; both only take effect once the §7.1 storage gate opens. Headers
  are always retained.
- SQLite needs a **local filesystem** — do not put `DATA_DIR` on a shared
  network mount, and never point two processes at one DB.

---

## 8. Security hardening

- **Never expose** the node HTTP API (`8000`) or debugpy (`5678`) to the
  public internet. Bind to a private network; if external access is needed,
  put it behind a TLS reverse proxy with auth.
- Set a strong, unique `DISCOVERY_SECRET`; rotate it across the fleet.
- Protect **consensus owner keys** (the `CONSENSUS_OWNER_PUBKEY(S)` signing
  keys govern era upgrades) and **wallet files** in `DATA_DIR` — treat them
  like any signing key material (Secrets, restricted file perms, backups).
- Run the discovery Redis on a private network; never expose `6379`.
- Run nodes as a non-root user.

---

## 9. Monitoring & operations

- **Health:** `GET /status` (mining flag, tip height, peers, consensus_id,
  prune-gate + recall state, pruned_below_height) and `GET /tip`. Use
  `/status` as the liveness/readiness probe.
- **Chain comparison:** `python tools/chain_comparison_tool.py <urls…>` to
  confirm nodes agree.
- **Logs:** stdout + a rotating file under `DATA_DIR` (`LOG_LEVEL`,
  `max_file_size_mb`, `max_backup_count`).
- **Upgrades:** node software updates are rolling (one node at a time;
  headers/bodies re-sync). Consensus-RULE changes ship as a governed
  `consensus_activation` at a future height — make sure every node runs the
  code for the target era before its `activation_height` (check
  `/authorities` and `/pending-activations`).

---

## 10. Reference topology (minimum viable network)

- 1 × discovery (+ Redis)
- ≥ 3 × miner nodes (the default `min_peers` is 3; more miners ⇒ more
  decentralisation and sooner storage-gate opening)
- optional: 1+ librarian (archival) for history serving; light/SPV nodes for
  low-resource participants

See also: `README.md` (quick start), `docs/Tesseracoin_overview.docx` (architecture),
`docs/Tesseracoin_proof_of_access_storage.docx` (disk-growth model), and
`docs/Tesseracoin_security_model.docx` (threat model).
