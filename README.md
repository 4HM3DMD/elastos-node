# elastos-node

`elastos-node` is a security-hardened fork of the [`elastos/Elastos.Node`](https://github.com/elastos/Elastos.Node) runner. It manages the Elastos main chain (ELA), the EVM side chains (ESC, EID, PG), their cross-chain oracles, and the arbiter through a single `node.sh` script.

The fork differs from the upstream runner in two areas:

- **Security defaults.** JSON-RPC and WebSocket endpoints bind to `127.0.0.1`, no signing account is unlocked at startup, the exposed RPC API surface is reduced, and self-update verifies a published SHA-256 checksum. See [SECURITY.md](SECURITY.md).
- **Deployment profiles.** A node can run the main chain only, or the full cross-chain stack. See [Deployment profiles](#deployment-profiles).

All upstream commands continue to work unchanged. The fork uses the same directory layout, binaries, keystore files, and log scheme as the upstream runner, so it functions as a drop-in replacement on an existing installation. A full feature comparison is available in [docs/COMPARISON.md](docs/COMPARISON.md).

## Supported components

| Component | Description |
|---|---|
| `ela` | Elastos main chain (BPoS consensus, CR Council governance) |
| `esc` | Elastos Smart Chain (EVM side chain) |
| `eid` | Elastos Identity Chain (EVM side chain) |
| `pg` | PGA chain (EVM side chain) |
| `esc-oracle`, `eid-oracle`, `pg-oracle` | Cross-chain oracle services |
| `arbiter` | Cross-chain arbiter |

The decommissioned ECO and PGP side chains are excluded from all profiles and are not started by this script.

## Requirements

- Ubuntu (tested on 22.04 and 24.04)
- `curl`, `jq`, and the packages installed automatically by `setup`
- Open inbound peer-to-peer and consensus ports (handled by `setup` or the `firewall` command; see [SECURITY.md](SECURITY.md) for the port table)

## Installation

```bash
mkdir -p ~/node && cd ~/node
curl -fsSLO https://raw.githubusercontent.com/4HM3DMD/elastos-node/main/node.sh
chmod +x node.sh
```

## Quick start

```bash
./node.sh setup      # dependencies + swap + firewall + autostart, then init
./node.sh start
./node.sh summary
```

`setup` prompts for the deployment profile (main chain only, or full stack) and uses `sudo` for system changes. It also installs a global `node.sh` wrapper in `/usr/local/bin`, so subsequent commands can be run from any directory.

To perform the steps individually instead:

```bash
./node.sh profile set mainchain   # or: full
./node.sh init                    # download binaries + create the keystore
./node.sh firewall                # open peer/consensus ports (RPC stays on loopback)
./node.sh start
```

Side chains that mine require a cold reward address before they will start:

```bash
node.sh reward set 0xYOURCOLDADDRESS
```

## Deployment profiles

| Profile | Runs | Intended use |
|---|---|---|
| `mainchain` | `ela` | BPoS supernode, CR Council node, or a standalone ELA full node |
| `full` (default) | `ela`, `esc`, `eid`, `pg`, the three oracles, `arbiter` | Full cross-chain stack |

The profile is persisted to `~/.config/elastos/profile` and governs the bulk commands (`start`, `stop`, `status`, `update`, `summary`, `health`).

```bash
node.sh profile                    # show the active profile
node.sh profile set mainchain      # change it
node.sh --profile full status      # override for a single command
```

Individual chains can always be addressed directly, regardless of profile:

```bash
node.sh esc start
node.sh ela status
```

## Command reference

### Global commands

| Command | Description |
|---|---|
| `setup` | Prepare a fresh host: dependencies, 16 GB swap, firewall, autostart, then `init` |
| `init` | Download binaries and create the keystore |
| `start` / `up` | Start every chain in the active profile |
| `stop` / `down` | Stop every chain in the active profile |
| `restart` | Restart the profile's chains one at a time (excludes `ela` unless `--force` is given) |
| `status` | Per-chain status for the active profile (`--verbose` for the complete dump) |
| `summary` / `ps` | One row per chain: state, height, peers, sync (`--json` available) |
| `health` | One-line verdict per chain; exits non-zero if any chain is unhealthy |
| `logs [<chain>] [-f]` | Show the most recent log for a chain (`-f` to follow) |
| `update` | Update chain binaries (cold-reward check before stopping a miner) |
| `profile [set <p>]` | Show or set the deployment profile |
| `firewall` | Open the peer/consensus ports for the active profile |
| `reward [set <0x..>]` | Show or set the cold mining reward address for all side chains |
| `migrate [--dry-run]` | Move an existing upstream or older-fork installation onto this fork |
| `migrate --apply [--yes]` | Staged restart that applies the hardened RPC binding (side chains only) |
| `uninstall` | Stop all processes and remove the installation (keystore backed up first) |
| `version` / `-v` | Script and chain versions |
| `help` / `-h` | Full command reference |

### Per-chain commands

```
node.sh <chain> <command>
```

where `<chain>` is one of `ela`, `esc`, `esc-oracle`, `eid`, `eid-oracle`, `pg`, `pg-oracle`, `arbiter`.

| Command | Description |
|---|---|
| `start` / `up`, `stop` / `down`, `restart` | Process control |
| `status [--pretty] [--json] [--verbose]` | Status in labeled, health-first, machine-readable, or complete form |
| `health` | Single-chain health check with exit code |
| `logs [-f]` | Most recent log file |
| `client` | Invoke the chain's CLI client |
| `jsonrpc` / `rpc` | Send a JSON-RPC request to the chain |
| `init`, `update`, `version` | Per-chain lifecycle |
| `compress_log`, `remove_log` | Log maintenance |

ELA additionally supports the governance commands from the upstream runner: `register_bpos`, `activate_bpos`, `unregister_bpos`, `vote_bpos`, `stake_bpos`, `unstake_bpos`, `claim_bpos`, `register_crc`, `activate_crc`, `unregister_crc`, `send`, `transfer`. Commands may be written in kebab-case (`register-bpos`) or snake_case (`register_bpos`).

### Output control

| Flag / variable | Effect |
|---|---|
| `--json` | Machine-readable output for `summary` and `status` |
| `--no-color` or `NO_COLOR=1` | Disable ANSI color |
| `--profile <p>` | Override the active profile for one command |
| Non-TTY `status` | Falls back to the classic full dump so existing parsers keep working |

## Security model

Summary of the defaults; details and the full port table are in [SECURITY.md](SECURITY.md).

- RPC and WebSocket bind to `127.0.0.1`. The bind address can be changed deliberately via the `EVM_RPC_BIND` environment variable or `~/.config/elastos/evm_rpc_bind`; an invalid value falls back to `127.0.0.1`.
- No `--unlock` and no `--allow-insecure-unlock`. Block production signs with the dedicated PBFT/ELA keystore, so sealing is unaffected.
- The `personal`, `admin`, `db`, and `miner` RPC namespaces are not exposed.
- A mining side chain refuses to start without a valid cold reward address.
- `update` verifies a published SHA-256 checksum and a syntax check before replacing the script.
- The ELA `sponsors` file required past block ~1,801,550 is downloaded automatically when missing.
- The main-chain status query that can panic a syncing daemon is gated behind a sync check and reports `N/A` until the node is synced.

For remote monitoring, use an SSH tunnel or VPN rather than exposing RPC ports.

## Migrating an existing node

The `migrate` command moves an installation running the upstream `node.sh`, or an older version of this fork, onto the current version. It preserves the keystore, chain data, and configuration, writes a rollback snapshot, and never restarts or deletes anything by itself.

```bash
node.sh migrate --dry-run    # preview, changes nothing
node.sh migrate              # write the profile + rollback snapshot
node.sh migrate --apply      # staged side-chain restarts to apply the hardened binding
```

`migrate --apply` restarts only stale side chains, one at a time, verifying each returns on `127.0.0.1` before continuing. The ELA main chain is never restarted, so a council producer keeps signing throughout. The full procedure, including verification and rollback, is documented in [docs/MIGRATION.md](docs/MIGRATION.md).

## Updating

```bash
node.sh update
```

Self-update downloads `node.sh` from this repository, verifies it against the published `node.sh.sha256`, runs a syntax check, and only then replaces the installed script. Chain binaries are updated from the official Elastos distribution servers, with a cold-reward-address check before any mining chain is stopped.

## File locations

| Path | Purpose |
|---|---|
| `~/node/` | Installation root (one subdirectory per chain) |
| `~/node/<chain>/` | Binary, configuration, and data for a chain |
| `~/node/<chain>/logs/` | Rotated log files |
| `~/.config/elastos/` | Keystore passwords, profile, node configuration |
| `~/.config/elastos/profile` | Active deployment profile |
| `/usr/local/bin/node.sh` | Global wrapper installed by `setup` |

## Versioning

Releases are tagged `vMAJOR.MINOR.PATCH` and documented in [CHANGELOG.md](CHANGELOG.md). The running version is shown by `node.sh version`.

## License

See [LICENSE](LICENSE).

## Acknowledgements

Derived from [`elastos/Elastos.Node`](https://github.com/elastos/Elastos.Node). The upstream runner is the work of the Elastos contributors. This repository adds the security hardening, deployment profiles, migration tooling, and operator interface documented above.
