# Changelog

## v0.1.0 — Hardened fork

First release of the hardened `node.sh`, forked from upstream `elastos/Elastos.Node`.

### Security
- **EVM RPC/WS exposure closed.** All EVM side-chain start paths (`esc`, `eid`, `pg`, and the dormant `eco`/`pgp`) now:
  - bind `--rpcaddr` and `--wsaddr` to `127.0.0.1` (were `0.0.0.0`);
  - drop `--unlock` and `--allow-insecure-unlock` — block sealing is unaffected because consensus signs with the dedicated PBFT/ELA keystore, not the EVM account;
  - reduce `--rpcapi` to `eth,net,web3,txpool,pbft` (mining) / `eth,net,web3,txpool` (follower), removing `personal`, `db`, `miner`, and `admin`.
- Net effect: the no-password `eth_sendTransaction` path reachable over public RPC is removed.

### Accuracy
- **Status no longer crashes a syncing node.** `ela status` previously called `dposv2rewardinfo` (all-address form), which can panic the ELA daemon during sync (`concurrent map iteration and map write`). It is now gated on `ela_synced` and reports `N/A` until the node is fully synced.

### Features
- **Deployment profiles.** New `mainchain` and `full` profiles, persisted to `~/.config/elastos/profile`, with a `--profile <p>` per-command override and a `profile [set <p>]` command. The bulk commands (`init`/`start`/`stop`/`status`/`update`) act on the active profile's chains; individual chains always work directly.
- **`help` / `-h` / `--help`** top-level command.

### Removed
- **ECO** side chain (and `eco-oracle`) removed from dispatch and from all profiles (decommissioned). The dormant `eco_*` functions remain in the source but are unreachable via the dispatcher.

### Roadmap (v0.2)
- Mandatory cold `miner_address` enforcement — fail closed (abort start) if a mining node has no valid cold reward address, instead of silently falling back to the hot account.
- Apple-grade status UX: a `--pretty` health-first single-chain view, a `--summary` one-row-per-chain fleet table, semantic color **plus** colorblind-safe symbols, `NO_COLOR`, and a `--json` machine-readable mode (all additive; the default human output is unchanged for existing scripts).
- `set -o pipefail` and a targeted error-checking / exit-code sweep.
- Supply-chain integrity: SHA-256 + signature verification for self-update and binary downloads (the upstream self-update is unverified).
- Defense-in-depth: `--wsapi` allow-list and non-wildcard `--rpcvhosts`.
