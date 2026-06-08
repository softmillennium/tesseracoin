# Tesseracoin — Deployment & Operations Guide

Practical guidance for running Tesseracoin nodes in the real world: machine
sizing, and how to deploy via Docker, Kubernetes, a VM, or bare metal.

> **Status:** Tesseracoin is an advanced, chaos-tested **prototype**, not a
> formally audited production system. Start with a small, trusted community
> deployment, watch it, and grow with confidence. Never expose a node's HTTP
> API or the debug port to the public internet (see [Security](#8-security-hardening)).

---

## 1. What you are deploying

One OS process = one **node**. The simplest deployment is a set of nodes plus one
shared **discovery** service; the discovery service is optional once nodes know each
other (see §1.1).

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

### 1.1 Deployment models at a glance

| Model | Proxy | Discovery | When to use |
|-------|-------|-----------|-------------|
| **Docker Compose + nginx + discovery** | nginx (per-host) | yes | Reference. Single-host community deployment; nginx handles TLS and DoS at the HTTP boundary. |
| **k8s StatefulSet + ingress + discovery** | shared ingress | yes | Multi-host production; standard k8s path with Deployment for discovery + Redis. |
| **k8s headless Service (no discovery)** | shared ingress | **no** | k8s-native; pod DNS resolves peers directly via headless Service — set `PEERS` from stable pod hostnames; retire the discovery Deployment. |
| **Direct / pod sidecar (no proxy)** | **none** | optional | Node binds to a public port or a service mesh handles mTLS at the sidecar layer; the built-in rate limiter is the primary DoS gate (see §8.1). |
| **Static peers (no discovery)** | operator choice | **no** | Small fixed-topology or air-gapped networks; set `PEERS=http://node1:8000,http://node2:8000,…` on each node. |

Regardless of the chosen model, consensus traffic (gossip, sync) is **never** rate-limited —
the node's built-in rate limiter exempts peer subnets. Only the public read surface
(`/economics`, `/address/*/transactions`, `/chain`) gets the tight token bucket.

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
- Set `CONSENSUS_GENESIS_ID=2` for production POWP-Stake (the dev compose uses `3`,
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
            - { name: CONSENSUS_GENESIS_ID, value: "2" }
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
            CONSENSUS_GENESIS_ID=2
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
| `CONSENSUS_GENESIS_ID` | genesis era: `1`=PurePoW (default genesis), `2`=POWP-Stake, `3`=SmallNet, `5`=POWP-Stake+Recall SmallNet, `6`=POWP-Stake+Recall production, `7`=PoW+Recall SmallNet |
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

### 8.1 DoS hardening — layered defence

The node is hardened against flooding at three independent layers:

**Layer 1 — Reverse proxy / ingress (outside the node)**

nginx, Caddy, or a cloud load-balancer sits in front and handles:
- TLS termination (nodes' self-signed certs are invisible to clients)
- Coarse per-IP connection rate limits (`limit_req_zone` in nginx) before
  requests reach the process
- IP allowlisting for admin endpoints (`/tx/local`, `/debug-*`)
- DoS surface reduction — only the intended paths are forwarded; the node
  port is firewalled from the public internet

This layer is skipped in the **direct / pod-sidecar** model (§1.1), in which
case layer 2 becomes the first gate.

**Layer 2 — Node token-bucket rate limiter (built in, `rate_limit.py`)**

A per-IP token-bucket runs as ASGI middleware on the node. Design goals:

- **Consensus-safe** — loopback and a configurable CIDR allowlist (peer
  subnet + discovery) are unconditionally exempt. Peer gossip (`POST /block`,
  `POST /tx`, `GET /headers`, `GET /chain/from`) and sync endpoints are
  never included in the rate-limit path — throttling a peer would stall
  consensus.
- **Two buckets per IP** — a generous general bucket (normal polling is well
  below it) and a tight *expensive* bucket for the compute-heavy read
  endpoints (`/economics`, `/address/*/transactions`, `/chain`, `/supply`).
  Peers do not call the expensive endpoints during sync.
- **Temporary ban** — a caller that repeatedly triggers the tight bucket is
  banned for a configurable period (`ban_duration`, default 300 s). The ban
  is in-memory; a restart clears it.
- **Memory-safe under spoofed-source floods** — idle client entries are
  reaped periodically; the table cannot grow unbounded.

Configure via the `rate_limit` section of `node_config.json`:

```json
"rate_limit": {
  "requests_per_second": 20,
  "burst": 40,
  "expensive_requests_per_second": 2,
  "expensive_burst": 5,
  "ban_duration": 300,
  "exempt_cidrs": ["10.0.0.0/8", "172.16.0.0/12"]
}
```

`exempt_cidrs` should include the peer-to-peer subnet and the discovery
service so consensus traffic is never throttled.

**Layer 3 — Peer misbehaviour scoring (`peer_scoring.py`)**

Separate from the rate limiter, the peer scorer penalises gossip peers that
send invalid blocks or transactions (not rate-limited peers — this is the
consensus layer). Repeat offenders are banned for `ban_duration` seconds and
skipped in gossip broadcasts.

**Load test harness**

A JMeter harness in `tests/load/` can replay realistic workloads against a
running node. The signing sidecar generates authentic request payloads —
important because a load-generator IP cannot both flood the node and measure
real throughput (it ends up in its own ban bucket). Use a separate measurement
probe IP.

### 8.2 Running without a proxy

When a proxy is not in the picture (direct-bind model or pod-sidecar):

1. Bind the node to a non-root port (`8000` or higher) and ensure the firewall
   only allows the intended client subnets.
2. Set `exempt_cidrs` in `rate_limit` to your peer subnet so nodes can gossip
   freely.
3. If TLS is required, terminate it at the service-mesh sidecar (e.g. Envoy or
   Istio with mTLS) — the node itself does not handle TLS natively.
4. Without a proxy, the expensive-endpoint bucket is the primary DoS gate for
   public read traffic; tune `expensive_requests_per_second` conservatively.

### 8.3 Running without the discovery service

Discovery is *peer introductions only* — it has no role in consensus. To
retire it:

**Option A — Static peers (`PEERS` env var)**

```bash
PEERS=http://node2:8000,http://node3:8000,http://node4:8000
```

Set on each node. Suitable for small fixed-topology networks and air-gapped
deployments. Every peer needs to list every other peer (or at least enough to
be connected); the gossip fan-out handles the rest once connected.

**Option B — k8s headless Service DNS**

Create a headless `Service` (`clusterIP: None`) for the `StatefulSet`.
k8s assigns stable DNS names (`node-0.tesseracoin.default.svc.cluster.local`,
`node-1.tesseracoin…`, …). Set `PEERS` to the stable hostnames:

```yaml
env:
  - name: PEERS
    value: "http://node-0.tesseracoin:8000,http://node-1.tesseracoin:8000"
```

No discovery `Deployment` or Redis is needed. This is the recommended k8s
path at scale — discovery becomes an unnecessary single-point dependency once
the pod topology is stable.

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
