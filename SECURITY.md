# Security

## Default posture

`elastos-node` runs the Elastos chains with the following defaults:

- **Loopback-only RPC.** Every chain's JSON-RPC and WebSocket endpoints bind to `127.0.0.1`. They are reachable by the node's own oracle, arbiter, and CLI over loopback, but not from the network.
- **No remotely-spendable account.** EVM mining nodes do not unlock a signing account on startup. Block production uses the dedicated PBFT keystore; the EVM account is never unlocked for RPC, so there is no node-side `eth_sendTransaction` signing path.
- **Reduced RPC surface.** The `personal`, `admin`, `db`, and `miner` namespaces are not exposed.
- **Mandatory cold reward address.** A mining side chain refuses to start unless a valid cold reward address is configured, so block rewards never accrue to an account stored on the node.
- **Verified self-update.** `update` checks the download against a published SHA-256 checksum and runs a syntax check before replacing the script.
- **Sync-safe status.** Status queries that can panic a syncing daemon are gated behind a sync check.

## Changing the RPC bind address

The bind address for EVM RPC and WebSocket endpoints can be changed deliberately, for example to serve RPC inside a private network:

- environment variable: `EVM_RPC_BIND=<address>`
- persistent file: `~/.config/elastos/evm_rpc_bind`

An invalid value falls back to `127.0.0.1`. A non-loopback bind is reported at every chain start. The removal of `--unlock` and of the `personal` namespace applies regardless of the bind address.

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
