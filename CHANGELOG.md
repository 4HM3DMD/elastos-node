# Changelog

All notable changes to this project are documented in this file. Releases are tagged `vMAJOR.MINOR.PATCH`.

## v1.0.0-rc.3 - Warn instead of refuse on a missing cold reward address

### Changed
- A mining side chain without a cold reward address now starts (matching upstream behavior) instead of refusing to start. Every such start prints a prominent red warning stating that block rewards will credit the node's local hot account, together with the `reward set` command. The refusal policy introduced in v0.2.0 proved too strict for operators who accept mining to the local account.
- The refusal-based guards tied to the old policy are removed: `update` no longer blocks on a missing reward address, `restart` no longer declines to stop such a chain, `migrate --apply` no longer skips it, and the bulk `start` no longer collects refused chains (nothing refuses anymore, which also removes the arbiter skip case added in v1.0.0-rc.2).
- Internals: `require_cold_miner` and `guard_cold_for_update` are replaced by `has_cold_miner` (silent predicate) and `warn_hot_miner` (red warning, never blocks).

## v1.0.0-rc.2 - Final compatibility review fixes

A five-reviewer compatibility audit of v1.0.0-rc.1 against the upstream runner confirmed command-surface, start-flag, init/update, and status-output parity, and found the following defects, all fixed and covered by isolated function tests.

### Fixed
- **Critical: `start`, `up`, `restart --force`, and the `@reboot` autostart refused to start the ELA main chain on every initialized node.** The cold-reward-address gate treated the ELA keystore password file as evidence of a mining configuration and demanded a `miner_address.txt` that never exists for ELA. The gate is now scoped to the EVM side chains only; ELA, the oracles, and the arbiter are never gated. Introduced in v0.9.7; the direct `ela start` command was unaffected.
- `start` no longer hangs when side chains are refused for a missing cold reward address. The arbiter start loop (inherited from upstream) respawns until the arbiter stays up, which never happens while its dependencies are down; `start` now skips the arbiter in that case, reports it, and exits non-zero.
- `--profile` given without a value caused an infinite busy loop in the argument parser. It now exits with an error message.
- `remove_log` failed with `command not found` on a full-profile node: `pg-oracle_remove_log` did not exist. The function slot held a dead duplicate of `pg-oracle_update` operating on the pgp-oracle directories (inherited from upstream, where the same defect exists). The dead duplicate was replaced with a correct `pg-oracle_remove_log`.
- `health` reported `healthy` for a chain whose daemon was running but whose RPC was unreachable. That state now reports `running (rpc unreachable)` and exits non-zero.
- `status --pretty` was advertised but not wired to a handler; it now renders the health-first view.
- The single-chain `status` in the piped (non-TTY) classic path no longer discards stderr, matching upstream diagnostics for automation that captures both streams.

### Changed
- `eco` and `eco-oracle` are directly addressable again (`node.sh eco stop`, `status`, `logs`), so an operator upgrading a host with a running ECO daemon retains managed control of it. ECO remains excluded from every profile and from all bulk commands.
- `setup` now installs the 10-minute `compress_log` cron entry in addition to the `@reboot` autostart, and its deduplication recognizes both the absolute-path and the upstream tilde-path entry forms, preventing duplicate `@reboot` entries.

## v1.0.0-rc.1 - Documentation release

Release candidate for v1.0.0. No functional script changes other than the version string.

### Documentation
- Rewrote `README.md` as a complete reference: installation, quick start, deployment profiles, full command tables, security model, migration, updating, file locations.
- Rewrote `SECURITY.md`: default posture, RPC bind override, remote-access guidance, full port table, vulnerability reporting.
- Added `docs/COMPARISON.md`: a feature and security comparison against the upstream `elastos/Elastos.Node` runner.
- Added `docs/MIGRATION.md`: the step-by-step procedure for moving an existing installation onto this fork, including verification and rollback.
- Normalized this changelog to a consistent, neutral format.

### Status
- Feature-complete against the project's security and operations plans. Field validation on a clean Ubuntu host is the remaining step before v1.0.0.

## v0.9.8 - Review fixes for v0.9.7

Seven issues found in a code review of v0.9.7, each verified with isolated function tests.

### Fixed
- `FORCE_ELA` is reset at startup. Previously an exported `FORCE_ELA=1` in the environment could silently re-enable main-chain restarts; now only the `--force` flag enables them.
- `restart` aggregates per-chain results and exits non-zero if any chain failed, matching the behavior of `start`.
- Interactive prompts (profile, network, orphan-config) read piped input when present and fall back to a safe default only at end of input. This restores correct behavior for piped invocations of `init`.
- `EVM_RPC_BIND` is validated, and can be persisted in `~/.config/elastos/evm_rpc_bind`. An invalid value falls back to `127.0.0.1`.
- The RPC bind notice is printed only after the daemon is confirmed running.
- The `sponsors` download aborts a stalled transfer after approximately 30 seconds (`--speed-limit` / `--speed-time`) instead of waiting for the full timeout.

### Changed
- Added the `EVM_CHAINS` constant and the `is_evm_chain`, `evm_rpc_bind`, and `guard_cold_for_update` helpers, replacing duplicated chain lists and guards.

## v0.9.7 - Compatibility audit fixes

A compatibility audit against the official `elastos/Elastos.Node` confirmed that a file swap is a zero-downtime operation, and identified the following issues, fixed in this release.

### Safety
- `restart` and `ela restart` no longer restart the ELA main chain by default, since that interrupts council consensus. The `--force` (or `--include-ela`) flag overrides this.
- `restart` and `update` check the cold reward address before stopping a mining side chain. A chain that could not be restarted is left running, with a message.
- `migrate --apply` covers the eco and pgp chains in addition to esc, eid, and pg.
- Migration rollback snapshots use a unique suffix to avoid same-second collisions.

### Visibility
- Every EVM chain start prints a one-line notice of the RPC/WS bind address.
- `start` collects side chains that refused to start (missing cold reward address) and exits non-zero with a summary.

### Compatibility
- New `EVM_RPC_BIND` environment variable (default `127.0.0.1`) for deliberately serving RPC on another interface. The removal of `--unlock` and the `personal` namespace applies regardless of the bind address.
- The `sponsors` download and the version probe have `curl` timeouts, so `ela start` cannot hang on a network failure.
- `status` falls back to the full classic output when not attached to a TTY, so existing scripts that parse the old per-chain output keep working.

### Automation
- Interactive prompts (profile, network, orphan-config, `uninstall`, `migrate --apply`) are TTY-guarded: in a non-interactive shell they take the safe default or refuse cleanly. `uninstall` refuses to delete unattended.

## v0.9.6 - Labeled status view

### Changed
- `node.sh status` shows each chain's status as a labeled block, one field per line, aligned. Retained fields: version, state, `Address`, `Public Key`, `Height`, `#Peers`, `Uptime`, `RAM`, `Disk`, and the governance block (`BPoS Name / State / Staked / Votes / Rewards`, `CRC Name / State`). Removed fields: `Balance`, `PID`, `#Files`, `#TCP`, and the TCP/UDP port lists.
- `node.sh <chain> status --verbose` shows the complete dump, including the removed fields. `node.sh summary` / `ps` remains the one-row-per-chain view.

## v0.9.5 - Governance information in status

### Changed
- The ELA status includes the governance line (BPoS or CRC name, state, votes, rewards), the address, and the full public key. An unregistered node is reported as such, with a pointer to `register-bpos` / `register-crc`.
- Side chains show the configured reward address. The arbiter shows its bridge heights (spv, esc, eid, pg).

## v0.9.4 - Status cards

### Changed
- `node.sh status` was redesigned as a compact per-chain card showing health verdict, version, peers, height, uptime, RAM, and disk. Superseded by the labeled view in v0.9.6.
- The three status views are layered: `summary` / `ps` (one row per chain), `status` (per-chain detail), `status --verbose` (complete dump).

## v0.9.3 - Global command

### Added
- `setup` installs a wrapper at `/usr/local/bin/node.sh`, so commands can be run from any directory without a path prefix. `SCRIPT_PATH` resolves through symlinks and wrappers to locate the installation directory.

### Changed
- `node.sh status` defaults to the summary view (superseded by v0.9.6). The detailed output remains available via `--verbose`.

## v0.9.2 - Initialization and start guards

### Fixed
- `init` detects an orphaned keystore password (`~/.config/elastos/<chain>.txt` without a matching keystore, typically left behind by removing `~/node` without clearing the configuration) and offers to clear it so initialization can proceed.
- `start` refuses to launch an uninitialized `ela` (no `config.json`) and directs the operator to `init`, instead of starting a daemon that cannot run.
- Post-`setup` instructions print full paths and the `reward set` command.

## v0.9.1 - Automatic sponsors file

### Fixed
- The ELA main chain requires a `sponsors` file (a height-to-sponsor lookup) to validate blocks past the RecordSponsor fork at approximately block 1,801,550. The upstream runner does not download this file, so fresh nodes stall at that height. `ela_start` now downloads the file automatically when missing (mainnet only), from the distribution matching the installed binary version, and prints a warning if the download fails. The download is one-time, approximately 28 MB.

## v0.9.0 - Staged hardening apply

### Added
- `node.sh migrate --apply [--yes]`: applies the hardened RPC binding with minimal downtime. Restarts only side chains running with stale flags, one at a time, and waits for each to return on `127.0.0.1` (verified from the live process command line) before restarting the next. The ELA main chain is never restarted. Chains without a cold reward address are skipped and reported. The procedure stops on the first failure.

## v0.8.5 - Migration command

### Added
- `node.sh migrate [--dry-run]`: moves an existing installation (an older version of this fork, or the official `elastos/Elastos.Node`) onto this fork.
  - Detects the source installation from the profile file and running processes.
  - Preserves the ELA keystore, chain data, and configuration. Aborts if `keystore.dat` is missing or `node.json` is invalid.
  - Infers and writes the deployment profile for an upstream installation.
  - Warns if a mining chain has no cold reward address.
  - Writes rollback snapshots (`node.sh.bak.<timestamp>`, `~/.config/elastos.bak.<timestamp>`).
  - Never restarts processes; it prints a staged one-chain-at-a-time restart plan instead.
  - `--dry-run` previews all of the above and changes nothing.

## v0.8.4 - Summary output

### Changed
- `summary` / `ps` print a glyph legend and an attention line naming the chains that need attention, for example `attention: esc(no-peers) eid(syncing) pg(stopped)`.
- Service rows (oracles, arbiter) show `-` for height and peers, since they have no block height.

## v0.8.3 - Help system

### Fixed
- `help` lists every command. The previous help only listed chain names; `summary`, `health`, `profile`, `setup`, `reward`, and the modern verbs were not discoverable.
- Per-chain help shows the commands that exist for that chain. The upstream `ela` help advertised `watch` and `mon`, which do not exist; `pg` had no help.
- `node.sh <chain>` with no command prints help and exits non-zero (previously exit 0). Unknown commands and chains also exit non-zero, with suggestions.

## v0.8.2 - Modern command layer

All additions are aliases over the existing dispatch; every upstream command continues to work unchanged.

### Added
- Global: `up` (= `start`), `down` (= `stop`), `ps` (= `summary`), `restart`, `logs [<chain>] [-f]`, `version` / `--version` / `-v`, `reward [set <0x..>]`, `uninstall` (stops processes, backs up the keystore, removes the installation after typed confirmation).
- Per-chain: `up`, `down`, `restart`, `logs [-f]`, `rpc` (= `jsonrpc`), `version`.
- Kebab-case command names are accepted everywhere (`register-bpos` is equivalent to `register_bpos`).

## v0.8.1 - Arbiter initialization fix

### Fixed
- `arbiter_init` required the decommissioned `eco-oracle` and `pgp-oracle` services and listed ECO in its cross-chain `SideNodeList`, causing initialization to fail on a full-stack node. The preflight checks and both the mainnet and testnet `SideNodeList` configurations now contain only ESC, EID, and PG.

## v0.8.0 - Turnkey setup

### Added
- `node.sh setup`: prepares a fresh Ubuntu host and initializes the node in one command: installs dependencies, adds a 16 GB swap file, configures the firewall, enables autostart on reboot, then runs `init`. Profile-aware, idempotent, and asks before making system changes.
- `node.sh firewall`: opens the peer and consensus ports for the active profile (mainchain: `20338`, `20339`; full: additionally `20638`, `20639`, `20648`, `20649`, `20678`, `20679`). RPC and WebSocket ports are not opened; they bind to `127.0.0.1`.

## v0.7.1 - Single-invocation first run

### Fixed
- The first invocation of any command previously wrote `~/.config/elastos/node.json` and exited, requiring the command to be run a second time. It now writes the configuration and continues with the requested command.

## v0.7.0 - Oracle start verification

### Added
- The post-start verification covers the oracle services. `start` verifies that every chain and oracle process remains running, and reports a process that exited immediately, with the last lines of its log.

## v0.6.0 - Chain start verification

### Added
- After `start`, each chain daemon is verified to still be running. A daemon that exited on launch is reported immediately with the last lines of its log, instead of being discovered later through a stopped status.

## v0.5.0 - Error-aware status values

### Fixed
- The per-chain `status` distinguishes RPC errors from real zero values. A height or peer count from an unreachable RPC shows `N/A`; a genuine `0` still shows `0`. Hex values are parsed with a dedicated converter, replacing arithmetic expansion that could misparse hex and leading-zero values. Applies to the esc, eid, and pg heights and peer counts, and to the ela peer count.

## v0.4.0 - Health checks

### Added
- `node.sh health` and `node.sh <chain> health`: a one-line verdict per chain with a meaningful exit code (0 healthy; non-zero if stopped, syncing, or without peers). `node.sh health` exits non-zero if any chain in the active profile is unhealthy, which makes it usable from cron and alerting.
- `<chain> status --pretty` shows a sync-progress line (`current / highest (NN%)`) while a chain is catching up.

## v0.3.0 - Summary dashboard and JSON output

### Added
- `node.sh summary` shows height and peers per chain, with sync-aware and peer-aware health indicators.
- `node.sh <chain> status --pretty` reports height, peers, sync state, and flags a reward address that belongs to the node's local keystore.
- `--json` output: `node.sh summary --json` (array) and `node.sh <chain> status --json` (object), each entry `{chain, installed, running, height, peers, sync, reward}`.

### Fixed
- Status RPC calls are bounded with `curl --max-time 3`, so a status query cannot hang. Hex parsing uses a dedicated converter. An unreachable RPC is shown as unknown rather than `0`.

## v0.2.0 - Verified self-update and cold rewards

### Security
- Self-update targets this repository instead of upstream, verifies the published SHA-256 checksum (`node.sh.sha256`), and runs `bash -n` on the download before installing. An update can no longer revert the hardening.
- A mining chain refuses to start unless `<chain>/data/miner_address.txt` contains a valid address, so block rewards cannot fall back to the node's local account.

### Added
- `node.sh summary`: one row per chain in the active profile, with health indicators and a running/stopped count.
- `node.sh <chain> status --pretty`: a health verdict above the standard status.
- Color output respects `NO_COLOR` and non-TTY contexts; `--no-color` flag.
- Suggestions for misspelled chain and command names.
- Guided `init`: asks for the deployment profile on first run and persists the choice.

## v0.1.0 - Hardened fork

First release, forked from `elastos/Elastos.Node`.

### Security
- All EVM side-chain start paths bind `--rpcaddr` and `--wsaddr` to `127.0.0.1` (previously `0.0.0.0`).
- Removed `--unlock` and `--allow-insecure-unlock`. Block sealing is unaffected: consensus signs with the dedicated PBFT/ELA keystore, not the EVM account.
- Reduced `--rpcapi` to `eth,net,web3,txpool,pbft` (mining) and `eth,net,web3,txpool` (follower), removing `personal`, `db`, `miner`, and `admin`.
- Net effect: an unauthenticated `eth_sendTransaction` path reachable over public RPC no longer exists.

### Fixed
- `ela status` no longer calls `dposv2rewardinfo` on a syncing node. That call can panic the ELA daemon during synchronization. It is gated on a sync check and reports `N/A` until the node is synced.

### Added
- Deployment profiles `mainchain` and `full`, persisted to `~/.config/elastos/profile`, with a `--profile <p>` override and a `profile [set <p>]` command.
- `help` / `-h` / `--help` top-level command.

### Removed
- The decommissioned ECO side chain and `eco-oracle` are removed from dispatch and from all profiles.
