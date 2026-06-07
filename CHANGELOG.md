# Changelog

## v0.2.0 — Safer + apple-grade UX

### Security / safety
- **Self-update targets the fork, not upstream.** `update_script` now pulls `4HM3DMD/elastos-node@main`, **verifies a published SHA-256 checksum** (`node.sh.sha256`), and runs `bash -n` on the download before installing. The hardening can no longer be silently reverted by an update.
- **Mandatory cold reward address.** A mining chain refuses to start unless `<chain>/data/miner_address.txt` contains a valid `0x…` address, so block rewards can never fall back to this node's local account. Set one with:
  ```bash
  echo 0xYOURCOLDADDRESS > <chain>/data/miner_address.txt && chmod 600 <chain>/data/miner_address.txt
  ```

### UX — all additive (default output is unchanged for existing scripts)
- **`node.sh summary`** — one row per chain in the active profile (the fleet glance), with health glyphs and a running/stopped count.
- **`node.sh <chain> status --pretty`** — a health-first verdict banner above the normal status.
- **Color is `NO_COLOR`- and TTY-aware**, with a `--no-color` flag.
- **Did-you-mean** suggestions on an unknown chain or command, alongside the valid list.
- **Guided `init`** — asks main-chain-only vs full stack the first time and persists the choice.

### Roadmap (next)
- Deeper `--pretty` / `--summary` columns: height vs network tip, peer quorum, hot-wallet flag — plus a `--json` mode, all with per-call `--max-time`.
- Status-truth fixes in the legacy per-chain status (hex-height parse, RPC-error vs a real zero, post-spawn health check).
- `set -o pipefail` sweep (pending verification on a Linux node).
- Binary-download integrity (the remaining `# TODO: verify checksum` path).

---

## v0.1.0 — Hardened fork

First release of the hardened `node.sh`, forked from upstream `elastos/Elastos.Node`.

### Security
- **EVM RPC/WS exposure closed.** All EVM side-chain start paths (`esc`, `eid`, `pg`, and the dormant `eco`/`pgp`) now:
  - bind `--rpcaddr` and `--wsaddr` to `127.0.0.1` (were `0.0.0.0`);
  - drop `--unlock` and `--allow-insecure-unlock` — block sealing is unaffected because consensus signs with the dedicated PBFT/ELA keystore, not the EVM account;
  - reduce `--rpcapi` to `eth,net,web3,txpool,pbft` (mining) / `eth,net,web3,txpool` (follower), removing `personal`, `db`, `miner`, and `admin`.
- Net effect: the no-password `eth_sendTransaction` path reachable over public RPC is removed.

### Accuracy
- **Status no longer crashes a syncing node.** `ela status` previously called `dposv2rewardinfo` (all-address form), which can panic the ELA daemon during sync. It is now gated on `ela_synced` and reports `N/A` until fully synced.

### Features
- **Deployment profiles.** `mainchain` and `full` profiles, persisted to `~/.config/elastos/profile`, with a `--profile <p>` override and a `profile [set <p>]` command. Bulk commands act on the active profile's chains.
- **`help` / `-h` / `--help`** top-level command.

### Removed
- **ECO** side chain (and `eco-oracle`) removed from dispatch and from all profiles (decommissioned). The dormant `eco_*` functions remain in the source but are unreachable via the dispatcher.
