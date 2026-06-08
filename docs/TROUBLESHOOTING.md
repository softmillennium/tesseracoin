# Troubleshooting

Common failure modes for a Tesseracoin v0.1 turnkey deployment, ordered
by frequency seen in real operation. Each entry: the symptom as it
appears to the operator, the root cause, and the fix.

For operator-side fundamentals see [`OPERATOR_GUIDE.md`](OPERATOR_GUIDE.md);
for end-user wallet questions see [`MEMBER_GUIDE.md`](MEMBER_GUIDE.md).

---

## A. Startup failures

### `./deploy.sh up` fails immediately with port-in-use

```
Error: bind: address already in use
```

Another process is bound to 8080 or 8443 on the host. Either:
- Stop the conflicting process (`lsof -i :8443` to identify), or
- Change `NGINX_HTTP_PORT` / `NGINX_HTTPS_PORT` in `.env` and
  re-run `./deploy.sh up`.

### Docker daemon not running

```
Cannot connect to the Docker daemon
```

Start Docker Desktop (macOS) or `systemctl start docker` (Linux),
then retry.

### Build fails on first `./deploy.sh up`

```
ERROR: failed to solve: ... pip install liboqs-python ...
```

The Dockerfile builds liboqs from source on first build (~2 minutes).
A transient PyPI timeout occasionally aborts it. Retry once:

```bash
./deploy.sh down && ./deploy.sh up
```

If the failure persists, check outbound HTTPS from the host
(`curl -v https://pypi.org/`).

---

## B. Bootstrap problems

### Nodes never finish bootstrapping

`./deploy.sh status` shows all 5 nodes running but tips never appear.
Logs show repeated "discovery rejected: unauthorized".

Root cause: discovery secret mismatch. Every node and the discovery
service must share the same `DISCOVERY_SECRET`.

Fix:
1. Ensure `.env` contains a single `DISCOVERY_SECRET=...` line.
2. `./deploy.sh down && ./deploy.sh up` so all containers pick up
   the same value.

### "discovery secret is the fail-closed default"

```
TESSERACOIN_DISCOVERY_INSECURE: discovery secret is "changeme"
which is the fail-closed default; refusing to start.
```

A production deploy (`SIM_MODE=0`) refuses the `changeme` default.
Either:
- Generate a real secret: `./deploy.sh init` (or `openssl rand -hex 32`
  → paste into `.env`), or
- Set `SIM_MODE=1` if this is a development cluster.

### Nodes start but no blocks mine

If the cluster runs but `./deploy.sh status` shows tip height
stuck at 1:
- Miners are throttled by `MINER_CPU_LIMIT` and the difficulty
  hasn't retargeted yet. Expect the first block within ~5 minutes
  on a fresh chain at the default `0.3` core cap.
- Remove the cap by setting `MINER_CPU_LIMIT=` (blank) in `.env`
  and recreating the miner containers:
  `docker compose up -d --force-recreate node1 node2 node3`.

---

## C. Sync and consensus

### One node's tip lags forever

A specific node (say node3) stays 10+ blocks behind the others.

- The node was offline longer than the discovery TTL (default 300s)
  and lost its peer list. Restart:
  `docker compose restart node3`. Headers-first sync catches up
  within a minute or two.
- If the node still lags after a restart and its peer count is 0,
  it can't find the others. Check that discovery is healthy:
  `docker logs tesseracoin-discovery --tail 50`.

### Monitor logs `SOAK [FAILED]` and stays failed

The `violations_total` counter is **cumulative per monitor process**
— once a violation is logged, the verdict stays FAILED until the
monitor restarts, even if the underlying issue resolves.

Fix: `docker restart tesseracoin-monitor`. The first 5 minutes of a
fresh run produce zero violations on a healthy chain; verify with:

```bash
docker logs -f tesseracoin-monitor | grep SOAK
```

### `PERSISTENT (N polls): /headers/from/1 probe returned nothing`

The monitor's headers probe timed out N polls in a row on a node.
On a CPU-capped sim (default `MINER_CPU_LIMIT=0.3`), the API thread
shares CPU with mining and occasionally takes 5-10 seconds to
respond. The default `REQUEST_TIMEOUT=20` in compose handles this.

If the message persists despite 20s timeouts, the affected node is
genuinely unresponsive: check `docker logs <that-node>` for crashes
or out-of-memory kills.

---

## D. Wallet and dashboard

### Wallet shows "unreachable"

The wallet's configured node URL doesn't resolve from the browser.
Possibilities:
- `PUBLIC_HOSTNAME` in `.env` is set to a hostname that doesn't
  exist in DNS or the local hosts file from the user's machine.
- The browser is on a different network than the cluster (e.g.
  cluster on a LAN, user on cellular).
- The TLS certificate isn't trusted (self-signed dev cert). Modern
  browsers refuse `https://` to a self-signed host by default.

Fix one of:
- Use the dashboard's URL bar to point at a reachable node
  (`?node=https://...`).
- Install a real TLS certificate (Let's Encrypt) and update
  `nginx/ssl/`.
- For a closed LAN deploy, ship the self-signed CA cert to each
  member's device.

### "POST /tx/local — 403 wrong secret"

The dashboard's send-tx panel requires `X-Debug-Secret` to match
`TESSERACOIN_DEBUG_SECRET`. The devnet-sim default is `devnet-sim-secret`.

Fix: type the secret into the Debug-secret field in the wallet
panel (it persists in sessionStorage for this browser tab).

### Tx sent but never appears in wallet history

- Transaction is in the mempool waiting for inclusion. Check the
  block explorer for the tx_id; if it appears in the mempool but
  not in a block, the fee may be below the cutoff. Re-send with a
  higher fee.
- If the tx is not in either: the API submission may have failed
  silently (rare). Check the node's logs:
  `docker compose logs node1 | grep tx_received`.

---

## E. Data and recovery

### Disk fills up

Chain data grows ~10-50 MB per day at sim cadence; faster on a
high-throughput cluster. The node5 sim instance is configured to
prune body data older than 50 blocks (a "librarian" mode); other
nodes retain everything.

To prune another node:

```ini
# in .env, before bringing up
KEEP_BLOCKS_NODE3=1000        # retain last 1000 blocks of bodies
```

Then `docker compose up -d --force-recreate node3`. Headers are
always retained — light clients can still verify history via the
peers that keep full bodies.

### Corrupted chain database

After an unclean shutdown (host crash, force-kill), a node may fail
to start with `database is locked` or `malformed`. SQLite's WAL mode
usually recovers, but if a checksum fails:

1. Stop the affected node: `docker compose stop nodeN`.
2. Back up the broken DB:
   `cp data/nodeN/chain.db data/nodeN/chain.db.broken`.
3. Restore from a healthy peer: delete `data/nodeN/chain.db*`,
   restart the node. Headers-first sync rebuilds from genesis.
   This takes a few minutes for a small chain, longer for a big one.

If no healthy peer is reachable, restore from the most recent backup
(`tar xzf tesseracoin-backup-YYYYMMDD.tar.gz`).

---

## F. Operator overrides for diagnostics

| Need | How |
|---|---|
| See live consensus state | `curl http://localhost:8001/consensus/eras` |
| Inspect mempool depth | `./deploy.sh status` shows `mp_max` in the monitor; per-node: `curl http://localhost:8001/info \| jq .mempool` |
| Force a re-sync from genesis | Stop a node, delete its `chain.db*`, restart |
| Trigger an immediate readiness re-check | `docker restart tesseracoin-monitor` |
| See per-peer scoring | `curl http://localhost:8001/peer-scores` |
| List all registered consensus authorities | `curl http://localhost:8001/consensus/registered` |

---

## G. When to seek help

A persistent failure that doesn't match any pattern above usually
points at:
- An incompatibility with the host OS (run the deployment in a
  Docker-on-Ubuntu-22.04 VM as the known-good baseline).
- A consensus bug surfaced by an edge case the sim hasn't hit.

Capture the following before reporting:

```bash
./deploy.sh status > diag.txt
docker compose logs --no-color --tail 500 > logs.txt
docker logs tesseracoin-monitor --no-color --tail 200 > monitor.txt
ls -la data/                                              >> diag.txt
git rev-parse HEAD                                        >> diag.txt
```

File an issue at https://github.com/softmillennium/tesseracoin/issues
attaching `diag.txt`, `logs.txt`, `monitor.txt` and a description of
what was being done when the failure surfaced.
