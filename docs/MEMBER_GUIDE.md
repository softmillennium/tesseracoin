# Member guide — using a Tesseracoin wallet

Audience: someone who has been given access to a Tesseracoin cluster
by its operator (the person who runs it for the group). This guide
explains how to receive, hold, and send funds on the chain — no
prior crypto experience assumed.

For the underlying technology, see the [Tesseracoin landing page](../website/index.html).
For running a cluster, see [`OPERATOR_GUIDE.md`](OPERATOR_GUIDE.md).

---

## 1. What is the wallet

The wallet is a single web page. It holds the cryptographic key
that controls one address on the chain.

- The wallet runs **entirely in the browser**. There is no wallet
  server. The private key never leaves the device.
- The wallet talks to a **Tesseracoin node** (a small server the
  operator runs) over plain HTTP/S to read balances and submit
  transactions.
- Locking the wallet, closing the tab, or refreshing the page wipes
  the unlocked key from memory. The encrypted copy remains in the
  browser's local storage until explicitly forgotten.

The wallet URL looks like:

```
https://<operator-host>/wallet/
```

The operator provides the exact URL.

---

## 2. First-time setup

### Step 1 — open the wallet

Navigate to the wallet URL the operator provided. The page either
opens in the **empty state** (first time on this browser) or the
**unlock state** (a wallet has been created here before).

### Step 2 — create a wallet

From the empty state, click **Create new wallet**, then:

- **Name** (optional) — only stored locally; helps if multiple
  wallets are kept on the same browser.
- **Signature scheme** — choose between:
  - **Dilithium2** (default, recommended): post-quantum.
    Wallet stays usable even after large quantum computers exist.
    Pubkey ~1.3 KB, signatures ~2.4 KB.
  - **Ed25519**: classical. Smaller keys/signatures (~32 bytes).
    Vulnerable to a future quantum attacker but otherwise fine for
    everyday use.
- **Passphrase** — a strong password. The wallet on disk is
  AES-GCM-encrypted with a key derived from this passphrase via
  250,000 PBKDF2 iterations. **Write it down.**

Confirm. A new keypair is generated locally; the wallet enters the
**main view** with a zero balance.

### Step 3 — share the address

The **Receive** card shows a `tsr1q…` address. Anyone on the same
cluster can send funds to it. Copy with the **Copy address** button
or just select-and-copy.

---

## 3. Receiving funds

There is nothing to do beyond sharing the address. Once a sender
broadcasts a transaction to the chain, the wallet shows the new
balance within ~10 seconds (one mining round, plus the wallet's
own refresh cycle).

---

## 4. Sending funds

From the main view, the **Send** card has three fields:

- **Recipient address** — paste a `tsr1q…` address. The wallet
  validates the checksum before submitting; a malformed address
  fails immediately, no chain round-trip.
- **Amount (TSR)** — the value to transfer. 1 TSR = 100 000 000
  satoshis (the chain's smallest unit).
- **Fee (TSR)** — leave blank for the suggested fee (a smoothed
  median of recent blocks). Override only if a priority send is
  needed.

Click **Send**. The wallet signs the transaction in the browser,
submits it to the node, and shows the new tx_id. The balance updates
within ~10 seconds once a miner includes the tx in a block.

---

## 5. Backup and restore

The wallet exists in two places:

1. **In this browser's local storage** — encrypted with the
   passphrase, persists across reloads.
2. **As an exported file** — same encrypted envelope, downloadable.

To export, click **Export wallet (encrypted)** from the footer. The
download is a `.tsr-wallet.json` file. Store it safely: a USB drive,
encrypted backup, password manager attachment.

To restore in another browser, open the wallet URL, click **Import
wallet file** from the empty state, and supply the same passphrase.

**Important:**
- The exported file is useless without the passphrase. Losing the
  passphrase loses the wallet — there is no recovery service.
- The unencrypted private key never leaves the device. The exported
  file is the encrypted envelope, not the raw key.

---

## 6. Switching devices

The recommended pattern:

1. On the source device: **Export** the wallet.
2. Move the file to the destination device via a private channel
   (USB drive, Signal, encrypted email, password manager).
3. On the destination: **Import wallet file**, enter passphrase,
   the wallet is now active there.

Both devices then have the same wallet. To use only the new one,
choose **Forget this wallet** on the source.

---

## 7. Locking and forgetting

- **Lock** — wipes the unlocked key from memory; passphrase needed
  to use the wallet again. The encrypted envelope stays in browser
  storage. Choose this when stepping away from a shared device.
- **Forget this wallet** — removes the encrypted envelope from
  browser storage. After forgetting, the wallet can only be restored
  from a backup file (or by knowing the seed phrase, which the v0.1
  wallet does not expose). Choose this only when finished with the
  wallet on this device.

---

## 8. The node URL

A small text box at the top of the wallet shows the node URL the
wallet talks to. The operator's deployment usually pre-fills this
correctly. If the cluster has multiple nodes available, any of them
will do — they all see the same chain.

Switching to a different node URL only changes which node sees the
transaction submissions; the wallet identity (address, balance) is
identical from any node's perspective.

---

## 9. Lost passphrase, lost wallet

The wallet has no recovery email, no password reset, no support
ticket. The passphrase is the only thing that decrypts the on-device
or exported envelope.

If the passphrase is lost and there is no other copy of the wallet:

- Funds previously received to that address are **permanently lost
  on the chain** (no one can move them without the key).
- The address itself can no longer be controlled and is effectively
  burned.

Treat the passphrase like a physical key — write it down somewhere
safe before walking away from a freshly-created wallet.

---

## 10. Quantum resistance — the practical version

Dilithium2 wallets remain safe even if a large quantum computer is
ever built. Ed25519 wallets do not. For a small-group cluster
storing modest amounts of value, ed25519 is normally fine; for
anything intended to outlast the next decade, Dilithium2 is the
prudent default — which is why it's the default on new wallets.

A wallet's scheme is fixed at creation. To migrate from ed25519 to
Dilithium2, create a new Dilithium2 wallet and send the funds across.
