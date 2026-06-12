# Security

## Default posture

`elastos-node` runs the Elastos chains with the following defaults:

- **Loopback-only RPC.** Every chain's JSON-RPC and WebSocket endpoints bind to `127.0.0.1`. They are reachable by the node's own oracle, arbiter, and CLI over loopback, but not from the network.
- **No remotely-spendable account.** EVM mining nodes do not unlock a signing account on startup. Block production uses the dedicated PBFT keystore; the EVM account is never unlocked for RPC, so there is no node-side `eth_sendTransaction` signing path.
- **Reduced RPC surface.** The `personal`, `admin`, `db`, and `miner` namespaces are not exposed.
- **Cold reward address warning.** A mining side chain without a configured cold reward address starts, but prints a prominent red warning at every start: block rewards then credit the node's local hot account. Configure a cold address with `node.sh reward set`.
- **Verified self-update.** `update` checks the download against a published SHA-256 checksum and runs a syntax check before replacing the script.
- **Sync-safe status.** Status queries that can panic a syncing daemon are gated behind a sync check.

## Changing the RPC bind address

The bind address for EVM RPC and WebSocket endpoints can be changed deliberately, for example to serve RPC inside a private network:

- environment variable: `EVM_RPC_BIND=<address>`
- persistent file: `~/.config/elastos/evm_rpc_bind`

An invalid value falls back to `127.0.0.1`. A non-loopback bind is reported at every chain start. The removal of `--unlock` and of the `personal` namespace applies regardless of the bind address.

Note: binding to a single non-loopback address means the daemon no longer listens on `127.0.0.1`, so local tooling (`node.sh <chain> jsonrpc` and the RPC-derived status fields) will report `N/A`. If local tooling must keep working alongside a network bind, use `0.0.0.0` behind a firewall instead of a single external address.

## Hardening applies in two layers

Closing the exposure on an EVM side chain happens in two steps, because the bind address is fixed at process start:

1. **Firewall (immediate).** The host firewall is closed on the RPC and WebSocket ports. This blocks the internet at once, restarts nothing, and is reversible. It is performed automatically by `migrate`, `update_script`, and the `harden` command. Local tooling (oracle, arbiter, CLI) is unaffected because it reaches the node over loopback, which a host firewall does not filter.
2. **Daemon rebind (on restart).** Binding RPC/WS to `127.0.0.1` and dropping `--unlock` and the `personal` namespace take effect only when the chain is restarted. On a producing node this is staged by the operator, one chain at a time, after each chain is synced.

Run `node.sh harden` at any time to close the firewall ports and list which running chains still need a restart for the second layer. Until that restart, the firewall is what blocks external access, so it must remain in place.

## Remote access

Do not expose the RPC or WebSocket ports to the internet. For remote monitoring or tooling, use an SSH tunnel or a VPN:

```bash
# example: reach the esc RPC locally through an SSH tunnel
ssh -N -L 20636:127.0.0.1:20636 operator@your-node
```

## Port table

Keep the peer-to-peer and consensus ports open. The RPC and WebSocket ports bind to loopback and require no firewall rule.

| Chain | Open (P2P / consensus) | Loopback only (RPC / WS) |
|---|---|---|
| ela | `20338`, `20339` | `20336` (restricted by the node's IP whitelist) |
| esc | `20638`, `20639` | `20636`, `20635` |
| eid | `20648`, `20649` | `20646`, `20645` |
| pg | `20678`, `20679` | `20676`, `20675` |

The `firewall` command opens the correct set for the active profile. Equivalent UFW rules:

```bash
ufw allow 20338,20339,20638,20639,20648,20649,20678,20679/tcp   # P2P / consensus
# RPC/WS need no rule; they bind to 127.0.0.1
```

## Reporting a vulnerability

Report security issues privately, for example through a GitHub private security advisory, rather than opening a public issue, so that operators can update before details are published.
