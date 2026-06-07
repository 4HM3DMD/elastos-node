# elastos-node

A hardened, operator-friendly runner for [Elastos](https://elastos.org) full nodes — the **ELA main chain** and its EVM **side chains**, their cross-chain **oracles**, and the **arbiter** — all managed through a single `node.sh`.

This is a security-hardened fork of the upstream [`elastos/Elastos.Node`](https://github.com/elastos/Elastos.Node) runner, with two headline changes:

- **Deployment profiles** — choose **main chain only** or the **full cross-chain stack**. Not every operator runs everything.
- **Secure by default** — loopback-only RPC, a reduced RPC surface, no remotely-unlockable signing account, and a status command that can't crash a syncing node.

> Built for Ubuntu. Aimed at council / supernode operators who need a node that is correct, secure, and pleasant to run.

## Quick start

```bash
mkdir -p ~/node && cd ~/node
curl -fsSLO https://raw.githubusercontent.com/4HM3DMD/elastos-node/main/node.sh
chmod +x node.sh

# choose what this node runs
./node.sh profile set mainchain     # ELA main chain only
#   or
./node.sh profile set full          # ELA + side chains + oracles + arbiter (default)

./node.sh init                      # set up the active profile's chains
./node.sh start
./node.sh status
```

## Deployment profiles

| Profile | Runs | For |
|---|---|---|
| `mainchain` | `ela` | BPoS supernode, CR Council vote node, or a plain ELA full node |
| `full` *(default)* | `ela` + `esc`, `eid`, `pg` + their oracles + `arbiter` | Operators running the full cross-chain stack |

The profile is chosen once and persisted to `~/.config/elastos/profile`. It governs the **bulk** commands:

```bash
./node.sh start      # start every chain in the active profile
./node.sh status     # status for the active profile
./node.sh stop
./node.sh update
```

Override for a single command without changing the saved profile:

```bash
./node.sh --profile mainchain status
```

An individual chain always works directly, regardless of profile:

```bash
./node.sh esc start
./node.sh ela status
```

Show or change the profile anytime:

```bash
./node.sh profile                   # show current profile + active chains
./node.sh profile set mainchain
```

> ECO and PGP side chains are decommissioned and are not part of any profile.

## Security hardening

This fork closes the most dangerous default in the upstream runner and tightens the surface:

- **RPC and WebSocket bind to `127.0.0.1`** (were `0.0.0.0`). The JSON-RPC is reachable by the node's own oracle / arbiter / tools over loopback, but not from the public internet.
- **No remotely-unlockable signing account.** EVM mining nodes no longer start with `--unlock` / `--allow-insecure-unlock`. Block sealing is unaffected — consensus signs with the dedicated PBFT/ELA keystore, not the EVM account.
- **Reduced RPC API surface** — `personal`, `db`, `miner`, and `admin` are no longer exposed.
- **Status can't crash the node.** The main-chain status no longer issues the reward query that panics a syncing daemon; it reports `N/A` until the node is fully synced.

For remote monitoring, front the node with an SSH tunnel or VPN rather than exposing the RPC port. See [`SECURITY.md`](SECURITY.md).

## Commands

```
./node.sh                       # usage
./node.sh help | -h | --help    # usage
./node.sh profile [set <p>]     # show / set deployment profile (mainchain | full)
./node.sh init|start|stop|status|update    # act on the active profile
./node.sh <chain> <command>     # act on a single chain: ela, esc, eid, pg, *-oracle, arbiter
./node.sh --profile <p> <cmd>   # override the profile for one command
```

## Credits

Derived from [`elastos/Elastos.Node`](https://github.com/elastos/Elastos.Node) — the upstream runner is the work of the Elastos contributors. This repository adds security hardening and the deployment-profile system. See [`CHANGELOG.md`](CHANGELOG.md) for the full list of changes.
