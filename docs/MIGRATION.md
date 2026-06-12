# Migration guide

This document describes how to move an existing Elastos node onto this fork. It covers two starting points:

- a node running the official `elastos/Elastos.Node` runner
- a node running an older version of this fork

The migration is designed so that the ELA main chain is never restarted. A council or supernode operator keeps producing and signing throughout.

## What migration does and does not do

| Preserved, never touched | Written | Never done automatically |
|---|---|---|
| ELA keystore (`ela/keystore.dat`) | Deployment profile (`~/.config/elastos/profile`) | Restarting any process |
| Chain data for every chain | Rollback snapshots of the script and configuration | Deleting any file |
| Keystore passwords and configuration | | Changing chain data |

## Prerequisites

- A working installation in `~/node` with `ela/keystore.dat` present. Migration aborts if the keystore is missing.
- A cold reward address for mining side chains is recommended. A mining chain without one still starts, but prints a red warning at every start, because block rewards then credit the node's local hot account. The migration reports which chains are affected.

## Procedure

### 1. Install the fork's script alongside the existing one

```bash
cd ~/node
curl -fsSL -o node.sh.new https://raw.githubusercontent.com/4HM3DMD/elastos-node/main/node.sh
curl -fsSL https://raw.githubusercontent.com/4HM3DMD/elastos-node/main/node.sh.sha256
shasum -a 256 node.sh.new    # compare with the published checksum
chmod +x node.sh.new
```

### 2. Preview

```bash
./node.sh.new migrate --dry-run
```

The dry run reports the detected source installation, the keystore preflight, the inferred profile, the reward-address status per mining chain, and which running side chains hold stale (unhardened) flags. It changes nothing.

### 3. Migrate

```bash
mv node.sh node.sh.upstream
mv node.sh.new node.sh
./node.sh migrate
```

This writes the profile and the rollback snapshots. Running daemons are not touched; they continue under the flags they were started with. The script swap itself is zero-downtime.

### 4. Set the cold reward address, if reported

```bash
./node.sh reward set 0xYOURCOLDADDRESS
```

This step is recommended rather than required. A mining chain without a cold reward address starts with a red warning, and its rewards credit the node's local hot account until one is set.

### 5. Apply the hardened binding

The hardened RPC binding takes effect when a side chain restarts. Apply it in stages:

```bash
./node.sh migrate --apply
```

This restarts only the side chains still running with stale flags, one at a time, and verifies each returns on `127.0.0.1` before restarting the next. The ELA main chain is excluded. The procedure stops at the first failure.

For a fleet (for example, 12 council nodes), apply on one node at a time per chain, so the number of simultaneously restarting nodes for any chain stays well below the consensus quorum.

### 6. Verify

```bash
node.sh summary        # every chain running, heights advancing
node.sh health         # exit code 0
ss -tlnp | grep -E '20636|20646|20676'   # RPC listeners on 127.0.0.1 only
```

## Rollback

Migration writes two snapshots before changing anything:

- `~/node/node.sh.bak.<timestamp>`: the previous script
- `~/.config/elastos.bak.<timestamp>`: the previous configuration directory

To roll back:

```bash
cd ~/node
cp node.sh.bak.<timestamp> node.sh
rm -rf ~/.config/elastos && cp -rp ~/.config/elastos.bak.<timestamp> ~/.config/elastos
```

Daemons started by the fork keep running during a rollback; restart them with the restored script at a convenient time.

## Compatibility notes

- All upstream commands keep working after the swap, including the BPoS and CRC governance commands.
- `status` prints the classic full output when not attached to a TTY, so existing parsing scripts continue to work.
- `restart` excludes the ELA main chain unless `--force` is given.
- The decommissioned ECO and PGP chains are not started by the fork.

## Removing a leftover ECO installation

Some upstream nodes still have the decommissioned ECO side chain installed and running. `migrate` detects this and reports it. To stop ECO and its oracle and remove their data:

```bash
node.sh eco purge
```

The command only acts when ECO is actually present on the node; otherwise it reports that there is nothing to remove. Before deleting, it stops `eco` and `eco-oracle` and backs up the ECO keystore and its password file to `~/eco-keystore-backup-<timestamp>.tar.gz`. Deletion requires typing `eco` to confirm (or the `--yes` flag for unattended use). If the ECO data directory was relocated through a symbolic link, the link target is reported so the space can be reclaimed manually.

`migrate --apply` leaves a running ECO daemon untouched and points to `eco purge` instead of restarting it.
