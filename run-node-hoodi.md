# Simple Mantle Node (Hoodi Testnet)

## Recommended Hardware

- 8GB+ RAM
- 4C+ CPU
- 100GB+ disk (SSD recommended)
- 10mb/s+ download

## Component Versions

| Component | Image |
|---|---|
| Execution client (op-reth) | `mantlenetworkio/mantle-op-reth:mantle-v1.6.2` |
| Rollup node (op-node) | `mantlenetworkio/mantle-op-node:v1.6.3` |

Both are set in [`docker-compose-hoodi-upgrade-beacon.yml`](docker-compose-hoodi-upgrade-beacon.yml).

## Required Software

- [docker](https://docs.docker.com/engine/install/) with [docker compose](https://docs.docker.com/compose/install/)
- [node](https://nodejs.org/en/download/) (to generate the JWT secret)
- [foundry](https://github.com/foundry-rs/foundry/releases) (`cast`, to generate the p2p key and query the node)
- [zstd](https://github.com/facebook/zstd) (to extract the snapshot)

> Note: recent foundry releases require glibc 2.35+. On older distributions (e.g. Amazon Linux 2023, CentOS / RHEL 8/9) `foundryup` may stop with an `anvil` error at the end; `cast` itself still works and is the only tool this guide needs — run it from `~/.foundry/versions/foundry-rs/foundry/<version>/cast`, or use any other machine to generate the key.

## Installation

### 1 Download repo

```
git clone https://github.com/mantle-xyz/networks.git
cd networks/
```

### 2 Generate init file

Generate the `jwt_secret_txt` file and the `p2p_node_key_txt` file:

```
mkdir -p hoodi/secret

node -e "console.log(require('crypto').randomBytes(32).toString('hex'))" > hoodi/secret/jwt_secret_txt

# recent cast versions print the private key to stderr, so merge stderr into the pipe
cast w n 2>&1 | grep -i "Private Key" | awk -F ": " '{print $2}' | sed 's/0x//' > hoodi/secret/p2p_node_key_txt
```

Both files must be 64 hex characters (65 bytes with the trailing newline). Check before continuing — an empty key file makes op-node fail to start:

```
wc -c hoodi/secret/jwt_secret_txt hoodi/secret/p2p_node_key_txt
```

### 3 Download the latest snapshot from mantle

We recommend starting the node from the latest snapshot, so you don't need to sync from genesis.

First, create a path for the ledger:

```
mkdir -p ./data/hoodi-reth
```

Second, download the latest official snapshot:

```
# Download tarball
HOODI_CURRENT_TARBALL_DATE=`curl https://s3.ap-southeast-1.amazonaws.com/snapshot.hoodi.mantle.xyz/current.info`
wget -c https://s3.ap-southeast-1.amazonaws.com/snapshot.hoodi.mantle.xyz/${HOODI_CURRENT_TARBALL_DATE}-hoodi.tar.zst

# Then verify your download
HOODI_CURRENT_TARBALL_CHECKSUM=`curl https://s3.ap-southeast-1.amazonaws.com/snapshot.hoodi.mantle.xyz/${HOODI_CURRENT_TARBALL_DATE}-hoodi-chaindata.tar.zst.sha256sum | awk '{print $1}'`
echo "${HOODI_CURRENT_TARBALL_CHECKSUM} *${HOODI_CURRENT_TARBALL_DATE}-hoodi.tar.zst" | shasum -a 256 --check
# If `shasum` is not installed (many Linux distributions), use sha256sum instead:
# echo "${HOODI_CURRENT_TARBALL_CHECKSUM}  ${HOODI_CURRENT_TARBALL_DATE}-hoodi.tar.zst" | sha256sum -c

# You should get the following output:
# ${HOODI_CURRENT_TARBALL_DATE}-hoodi.tar.zst: OK
```

Third, unzip the snapshot to the ledger path:

```
tar --use-compress-program=unzstd -xvf ${HOODI_CURRENT_TARBALL_DATE}-hoodi.tar.zst -C ./data/hoodi-reth
```

Check the data was unarchived successfully:

```
$ ls ./data/hoodi-reth
blobstore  db  discovery-secret  genesis.json  invalid_block_hooks  known-peers.json  lost+found  reth.toml  static_files
```

### 4 Operating the Node

The rollup node derives L2 data from L1, so you need an L1 Hoodi execution RPC (`L1_RPC_HOODI`) and an L1 Hoodi beacon API (`L1_BEACON_HOODI`). The beacon API must serve blob data (`/eth/v1/beacon/blob_sidecars/{slot}`); any regular Hoodi beacon node works.

> ⚠️ `L1_RPC_HOODI` and `L1_BEACON_HOODI` must be `export`-ed in the **same shell session**, right before the `docker compose ... up -d` command. If you run `up -d` in a new shell without re-exporting them, docker compose silently substitutes an empty string and op-node cannot sync from L1, with no obvious error pointing at the cause.

```
export L1_RPC_HOODI='HOODI_L1_RPC'        #please replace
export L1_BEACON_HOODI='HOODI_L1_BEACON'  #please replace

docker compose -f docker-compose-hoodi-upgrade-beacon.yml up -d
```

The snapshot is taken periodically, so right after starting the node will catch up on the blocks produced since the snapshot date (roughly 800 blocks per minute on the recommended hardware). The local block height reaches the public RPC height once this catch-up finishes.

## Check Installation Result

Follow these steps to check if the installation is successful.

### 1 Check Service

If the service status is `Up`, the service has started without any issues.

```
docker compose -f docker-compose-hoodi-upgrade-beacon.yml ps
```

### 2 Check Data

```
# query local op-reth latest block height
cast bn

# query latest block height from mantle hoodi rpc
cast bn --rpc-url https://rpc.hoodi.mantle.xyz

# check the safe and finalized height
cast rpc optimism_syncStatus --rpc-url localhost:9545 | jq .safe_l2.number
cast rpc optimism_syncStatus --rpc-url localhost:9545 | jq .finalized_l2.number

# check op-node is connected to the mantle hoodi peers (expect peer0 / peer1)
cast rpc opp2p_peers true --rpc-url localhost:9545 | jq '.peers | length'
```

## Upgrade for historical user

### 1 Stop historical node

```
docker compose -f docker-compose-hoodi-upgrade-beacon.yml down
```

### 2 Pull the latest code of this repo

```
# If your local code has changes, please use 'git stash' to cache first

git pull
```

**If you start the node using your own way, please refer to the compose files in this repo for the upgrade. Otherwise, it may cause irreversible damage to the node.**

### 3 Operating the Node

```
export L1_RPC_HOODI='HOODI_L1_RPC'        #please replace
export L1_BEACON_HOODI='HOODI_L1_BEACON'  #please replace
docker compose -f docker-compose-hoodi-upgrade-beacon.yml up -d
```

### 4 Check data

```
# query local op-reth latest block height and mantle hoodi rpc
cast bn && cast bn --rpc-url https://rpc.hoodi.mantle.xyz

# check the safe and finalized height
cast rpc optimism_syncStatus --rpc-url localhost:9545 | jq .safe_l2.number
cast rpc optimism_syncStatus --rpc-url localhost:9545 | jq .finalized_l2.number
```
