# Security

## Posture

`elastos-node` runs Elastos chains with a security-first default configuration.

- **Loopback-only RPC.** Every chain's JSON-RPC and WebSocket endpoints bind to `127.0.0.1`. They are reachable by the node's own oracle, arbiter, and CLI over loopback, but not from the network.
- **No remotely-spendable account.** EVM mining nodes do not unlock a signing account on startup. Block production uses the dedicated PBFT keystore; the EVM account is never unlocked for RPC, so there is no node-side `eth_sendTransaction` signing path.
- **Reduced RPC surface.** The `personal`, `admin`, `db`, and `miner` namespaces are not exposed.
- **Sync-safe status.** Status queries that can panic a syncing daemon are gated behind a sync check.

## Remote access

Do **not** expose the RPC / WebSocket ports to the internet. If you need remote access for monitoring or tooling, use an SSH tunnel or a VPN:

```bash
# example: reach esc's RPC locally through an SSH tunnel
ssh -N -L 20636:127.0.0.1:20636 operator@your-node
```

## Firewall

Keep the peer-to-peer and consensus ports open; the RPC / WS ports bind to loopback and should never be reachable remotely.

| Open — P2P / consensus (peers need these) | Loopback only — RPC / WS |
|---|---|
| ela `20338`, `20339` | ela RPC is already localhost-restricted |
| esc `20638`, `20639` | esc `20636`, `20635` |
| eid `20648`, `20649` | eid `20646`, `20645` |
| pg  `20678`, `20679` | pg  `20676`, `20675` |

Example (UFW):

```bash
ufw allow 20338,20339,20638,20639,20648,20649,20678,20679/tcp   # P2P / consensus
# RPC/WS need no rule — they bind to 127.0.0.1
```

## Reporting a vulnerability

Please report security issues privately (e.g. a GitHub private security advisory) rather than opening a public issue, so operators can update before details are public.
