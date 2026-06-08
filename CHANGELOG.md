# Changelog

## v0.9.0 — Zero-downtime hardening apply (finishes the UX/migration roadmap)

- **`node.sh migrate --apply [--yes]`** — applies the hardened RPC binding with **near-zero downtime**:
  - restarts **only stale side chains** (`esc`/`eid`/`pg`), **one at a time**, and waits for each to come back on `127.0.0.1` (verified via its live command line) **before** restarting the next;
  - **never restarts the ELA mainchain** — your council producer keeps signing throughout, so consensus is uninterrupted;
  - **skips** any chain with no cold reward address (it would refuse to start) and **stops on failure**, so you investigate before continuing;
  - a single node can't know fleet quorum, so it reminds you to coordinate across the council.
- The version string now tracks the release (`v0.9.0`).

## v0.8.5 — `migrate` (move onto the fork safely)

- **`node.sh migrate [--dry-run]`** — move an existing install (an older fork version **or** the official Elastos `node.sh`) onto this hardened fork:
  - **detects the source** (old-fork / official-upstream / fresh) from the profile file + running processes;
  - **preserves what matters** — the ELA keystore, chaindata, and config are never touched; it **aborts** if `keystore.dat` is missing or `node.json` is invalid;
  - **infers the profile** for an upstream node (from what's installed) and writes it;
  - **bridges the cold-miner gap** — warns if a mining chain has no cold reward address (this fork refuses to mine to a hot key);
  - **snapshots for rollback** (`node.sh.bak.<ts>`, `~/.config/elastos.bak.<ts>`);
  - **never auto-restarts** — it hands you a staged, one-chain-at-a-time restart plan (the hardened RPC binding only applies after a restart), so you stay above BFT quorum;
  - **`--dry-run`** previews everything and changes nothing.

## v0.8.4 — Output polish (the dashboard explains itself)

- `summary` / `ps` now print a **glyph legend** (`● healthy   ◐ syncing/attention   ○ stopped`) and an **attention line** that names exactly which chains need it — e.g. `⚠ attention: esc(no-peers) eid(syncing) pg(stopped)`. No more guessing what `◐` means or scanning rows.
- Service rows (oracles, arbiter) show `-` for height/peers instead of a misleading `?` (they have no block height).

## v0.8.3 — Help & onboarding (discoverable at last)

- **`help` now lists every command.** The old help grep-generated only *chain names* — `summary`, `health`, `profile`, `setup`, `reward`, and all the modern verbs were invisible. It's now a clean, grouped, modern-first reference.
- **Honest per-chain help.** `node.sh <chain>` shows that chain's real commands (the old `ela_usage` advertised `watch`/`mon` that don't exist; `pg` had no help at all). ELA lists its governance commands; side chains show the cold-reward hint.
- **No more silent success.** `node.sh <chain>` with no command now prints help and **exits non-zero** (was exit `0` — a scripting landmine). Unknown command/chain errors also exit non-zero, with a richer did-you-mean across both verbs and chains.

## v0.8.2 — Modern command layer (docker/gh-style verbs)

All additive — **every old command still works byte-for-byte**; the modern verbs are aliases over the same dispatch.

### Global
- `up` (= `start`), `down` (= `stop`), `ps` (= `summary`)
- `restart` — restart the profile, one chain at a time
- `logs [<chain>] [-f]` — tail a chain's most recent log (follow with `-f`)
- `version` / `--version` / `-v` — fork + chain versions
- `reward [set <0x…>]` — show, or set the cold miner address for **all side chains at once**
- `uninstall` — stop everything + remove the install/config (backs up the keystore first; requires typing `DELETE`)

### Per-chain (`node.sh <chain> <verb>`)
- `up` / `down` / `restart` / `logs [-f]` / `rpc` (= `jsonrpc`) / `version`
- kebab-case accepted everywhere (`node.sh ela register-bpos` == `register_bpos`)

## v0.8.1 — Fix: arbiter init failed on the removed ECO chain

- `arbiter_init` still hard-required `eco-oracle` (and `pgp-oracle`) to be initialized, and listed **ECO** in its cross-chain `SideNodeList` — so on a full-stack node the arbiter step failed with `ERROR: eco-oracle not initialized` even though every other chain set up fine. The ECO/PGP removal is now complete in the arbiter path (preflight checks + both the testnet and mainnet `SideNodeList` configs now contain only ESC / EID / PG).

## v0.8.0 — Turnkey setup (deps, swap, firewall, autostart)

### Onboarding
- **`node.sh setup`** — one command to prepare a fresh Ubuntu box and initialize the node: installs dependencies, adds a 16 GB swap, **opens the firewall**, enables `@reboot` autostart, then runs `init`. Profile-aware, idempotent, asks before making changes. This replaces the official guide's *manual* deps / swap / `ufw` / cron steps.
- **`node.sh firewall`** — open just the peer + consensus ports for the active profile:
  - mainchain: `20338`, `20339`
  - full: + `20638/20648/20678` (tcp+udp) and `20639/20649/20679`
- **RPC/WS ports are deliberately NOT opened** (they bind to `127.0.0.1`) — unlike the upstream guide, which exposes `20336/20636/20646/20676` to the internet.

## v0.7.1 — Fix: no more running a command twice on first run

- The **first** invocation of any command previously only selected the network, wrote `~/.config/elastos/node.json`, and **exited** — forcing you to re-run (e.g. `init` had to be run twice). It now writes the config and **continues with the requested command in one shot**.

## v0.7.0 — Post-spawn check for oracles

### Accuracy
- The post-spawn health check now also covers the **oracle services** (esc/eid/pg + dormant eco/pgp oracles). `start` verifies every chain *and* oracle actually stays up, reporting any dead-on-launch process immediately with its log tail.

### Roadmap (next — both externally gated)
- `set -o pipefail` sweep — pending a real Linux smoke test (won't ship blind).
- Binary-download checksum — needs upstream-published checksums at `download.elastos.io`.

## v0.6.0 — Post-spawn health check

### Accuracy
- After `start`, each chain daemon (`ela`, `esc`, `eid`, `pg`, and the dormant `eco`/`pgp`) is **verified to actually stay up**. If it died on launch, `start` now prints a clear failure and the **last lines of its log** immediately — instead of leaving the operator to notice a `stopped` status later.

### Roadmap (next)
- Extend the post-spawn check to the oracle services.
- `set -o pipefail` sweep (pending Linux verification).
- Binary-download checksum (needs upstream-published checksums).

## v0.5.0 — Honest status (no more fake zeros)

### Accuracy
- The legacy per-chain `status` now uses the correct, error-aware parser everywhere: a height or peer count from a **down or unreachable RPC shows `N/A`**, not a fake `0`. A genuine `0` (block 0, zero peers) still shows `0` — so the two are finally distinguishable. Correct hex parsing (`hex_to_dec`) replaces the `$(( ))` that silently turned errors into zero. Applies to `esc` / `eid` / `pg` (and the dormant `eco` / `pgp`) heights + peers, and `ela` peers.

> Output note: the only change is in the **error case** (RPC unreachable), `0` → `N/A`. All normal values are unchanged.

### Roadmap (next)
- Post-spawn health check on `start` (surface a daemon that dies on launch).
- `set -o pipefail` sweep (pending Linux verification).
- Binary-download checksum.

## v0.4.0 — Scriptable health checks + sync progress

### Monitoring
- **`node.sh health`** and **`node.sh <chain> health`** — a one-line verdict per chain with a **meaningful exit code** (`0` healthy; non-zero if stopped / syncing / no peers). Built for cron and alerting: `node.sh health && ok || alert`. `node.sh health` exits non-zero if *any* chain in the active profile is unhealthy.

### UX
- **`<chain> status --pretty`** now shows a **sync-progress line** (`current / highest (NN%)`) while a chain is catching up.

All additive — no change to existing output or commands.

### Roadmap (next)
- Apply the correct parsing inside the legacy per-chain `status` (RPC-error → `N/A`, not `0`).
- Post-spawn health check on `start` (surface a daemon that dies on launch).
- `set -o pipefail` sweep (pending Linux verification).
- Binary-download checksum.

## v0.3.0 — Real health dashboard + `--json`

### Accuracy
- New **timeout-bounded RPC substrate** (`curl --max-time 3`, so a status query can never hang) with **correct hex parsing** (`hex_to_dec` — no `$(( ))` octal/zero trap) and **error ≠ zero**: a down or unreachable RPC shows `?`, never a fake `0`.

### UX (additive — legacy per-chain `status` output unchanged)
- **`node.sh summary`** now shows **HEIGHT** and **PEERS** per chain, with sync- and peer-aware health glyphs (●/◐/○).
- **`node.sh <chain> status --pretty`** now reports height, peers, sync state, and a **hot-wallet** flag on the reward address, above the full status.
- **`--json`** machine-readable output: `node.sh summary --json` (array) and `node.sh <chain> status --json` (object), each `{chain, installed, running, height, peers, sync, reward}`.

### Roadmap (next)
- Network-tip / sync-% column (`eth_syncing` highestBlock + the ELA tip).
- Apply the same correct parsing inside the legacy per-chain `status` (post-spawn health check too).
- `set -o pipefail` sweep (pending verification on a Linux node).
- Binary-download checksum (the remaining `# TODO: verify checksum`).

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
