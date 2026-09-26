# Simple Mantle Node (Hoodi Testnet)

## Recommended Hardware

- 8GB+ RAM
- 4C+ CPU
- 100GB+ disk (HDD works for now, SSD is better)
- 10mb/s+ download

You can run a Mantle Hoodi RPC node either with **Docker Compose** (simplest,
single host) or with the bundled **Helm chart** on Kubernetes.

---

# 1. Deploy with Docker Compose

## Required Software

- [docker](https://docs.docker.com/engine/install/)
- [node](https://nodejs.org/en/download/)
- [foundry](https://github.com/foundry-rs/foundry/releases)
- [zstd](https://github.com/facebook/zstd)
- [aira2](https://aria2.github.io/)

## Installation

### 1 Download repo

```
git clone https://github.com/mantle-xyz/networks.git
```

### 2 Generate init file

generate the 'jwt\_secret\_txt' file and the 'p2p\_node\_key\_txt'

```
cd networks/

mkdir hoodi/secret

node -e "console.log(require('crypto').randomBytes(32).toString('hex'))" > hoodi/secret/jwt_secret_txt

cast w n |grep -i "Private Key" |awk -F ": " '{print $2}' |sed 's/0x//' > hoodi/secret/p2p_node_key_txt
```

### 3 Download the latest snapshot from mantle

We recommend that you start the node with latest shapshot, so that you don't need to wait a long time to sync data.

First, create a path for ledger:

```
mkdir -p ./data/hoodi-reth
```

Second, download the latest official snapshot:

```
# Download tarball
HOODI_CURRENT_TARBALL_DATE=`curl https://s3.ap-southeast-1.amazonaws.com/snapshot.hoodi.mantle.xyz/current.info`
wget -c https://s3.ap-southeast-1.amazonaws.com/snapshot.hoodi.mantle.xyz/${HOODI_CURRENT_TARBALL_DATE}-hoodi.tar.zst

# Then you can verify your download
HOODI_CURRENT_TARBALL_CHECKSUM=`curl https://s3.ap-southeast-1.amazonaws.com/snapshot.hoodi.mantle.xyz/${HOODI_CURRENT_TARBALL_DATE}-hoodi-chaindata.tar.zst.sha256sum | awk '{print $1}'`
echo "${HOODI_CURRENT_TARBALL_CHECKSUM} *${HOODI_CURRENT_TARBALL_DATE}-hoodi.tar.zst" | shasum -a 256 --check

# You should get the following output:
# ${HOODI_CURRENT_TARBALL_DATE}-hoodi.tar.zst: OK
```

Third, unzip snapshot to the ledger path

```
tar --use-compress-program=unzstd -xvf ${HOODI_CURRENT_TARBALL_DATE}-hoodi.tar.zst -C  ./data/hoodi-reth
```

Check the data was unarchived successfully:

```
$ ls ./data/hoodi-reth
blobstore  db  discovery-secret  genesis.json  invalid_block_hooks  known-peers.json  lost+found  reth.toml  static_files 
```

### 4 Operating the Node

use L1 beacon chain to pull the data for rollup node,
you need set up L1\_BEACON\_HOODI and L1\_RPC\_HOODI

L1\_BEACON\_HOODI is for querying data from eth blob,or you can use mantle da-indexer instead.the da-indexer address is <https://da-indexer-api.hoodi.mantle.xyz>

```
export L1_RPC_HOODI='HOODI_L1_RPC'        #please replace
export L1_BEACON_HOODI='HOODI_L1_BEACON'  #please replace
# export L1_BEACON_HOODI='https://da-indexer-api.hoodi.mantle.xyz'  #if you want to use mantle da-indexer.

docker-compose -f docker-compose-hoodi-upgrade.yml up -d 
```

## Check Installation Result

Follow these steps to check if the installation is successful

### 1 Check Service

If the service status is 'up,' it means that the service has started without any issues.

```
docker-compose -f docker-compose-hoodi-upgrade.yml ps
```

### 2 Check Data

```
# query local op-reth latest block height
cast bn

# query latest block height from mantle hoodi rpc
cast bn --rpc-url  https://rpc.hoodi.mantle.xyz

# check the safe and finalized height. 
cast rpc optimism_syncStatus --rpc-url localhost:9545 |jq .safe_l2.number
cast rpc optimism_syncStatus --rpc-url localhost:9545 |jq .finalized_l2.number
```

## Upgrade for historical user

### 1 Stop historical node

```
docker-compose -f docker-compose-hoodi-upgrade.yml down
```

### 2 Pull the latest code of this repo

```
# If your local code have changes, please use 'git stash' to cache first

git pull 
```

**If you start the node using your own way, please refer to the compose files in this repo for the upgrade. Otherwise, it may cause irreversible damage to the node.**

### 3 Operating the Node

```
export L1_RPC_HOODI='HOODI_L1_RPC'        #please replace
export L1_BEACON_HOODI='HOODI_L1_BEACON'  #please replace
docker-compose -f docker-compose-hoodi-upgrade.yml up -d 
```

### 4 Check data

```
# query local op-reth latest block height and mantle hoodi rpc
cast bn && cast bn --rpc-url  https://rpc.hoodi.mantle.xyz

# check the safe and finalized height. 
cast rpc optimism_syncStatus --rpc-url localhost:9545 |jq .safe_l2.number
cast rpc optimism_syncStatus --rpc-url localhost:9545 |jq .finalized_l2.number
```

---

# 2. Deploy on Kubernetes with Helm

This repo also ships a Helm chart at [`chart/mantle-node`](chart/mantle-node/).
It deploys two `StatefulSet`s (the `op-reth` execution layer and `op-node`),
provisions PVCs for chain data and peerstore, and mounts `rollup.json`
straight from this repo via a `ConfigMap`. You supply the engine-API JWT
and op-node libp2p key in the per-network values file. The official
snapshot from S3 is fetched + extracted by an init container on first
boot — no separate `genesis.json` download is needed, since the snapshot
tarball already includes it.

## Required Software

- [helm](https://helm.sh/docs/intro/install/) 3.8+
- [kubectl](https://kubernetes.io/docs/tasks/tools/) pointed at a cluster
  with a default `StorageClass` (override with
  `--set el.persistence.storageClass=...` if needed)
- [node](https://nodejs.org/en/download/) (for generating the JWT secret)
- [foundry](https://github.com/foundry-rs/foundry/releases) (for `cast` —
  generates the p2p key and queries the node)

## Installation

### 1 Download repo

```
git clone https://github.com/mantle-xyz/networks.git
cd networks
```

### 2 Generate init file

generate the 'jwt\_secret\_txt' value and the 'p2p\_node\_key\_txt' value
(same commands as the Docker path, just printed to stdout so you can paste
them into the values file below):

```
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"

cast w n |grep -i "Private Key" |awk -F ": " '{print $2}' |sed 's/0x//'
```

### 3 Edit hoodi.values.yaml

Open `chart/mantle-node/hoodi.values.yaml` and fill in your L1 endpoints
and the two secrets you just generated:

```yaml
l1:
  rpc:    "HOODI_L1_RPC"        # please replace
  beacon: "HOODI_L1_BEACON"     # please replace
  # beacon: "https://da-indexer-api.hoodi.mantle.xyz"  # or use Mantle DA indexer

secrets:
  jwtSecret:   "<paste the jwt_secret_txt value here>"
  p2pNodeKey:  "<paste the p2p_node_key_txt value here>"
```

### 4 Install

```
helm install hoodi ./chart/mantle-node \
  -f ./chart/mantle-node/hoodi.values.yaml \
  --namespace mantle-hoodi --create-namespace
```

On first install, the chart automatically downloads + extracts the latest
official Mantle Hoodi snapshot from S3
(`https://s3.ap-southeast-1.amazonaws.com/snapshot.hoodi.mantle.xyz`) into
the EL PVC. The init container is a no-op when the PVC already contains
chain data, so it is safe across `helm upgrade`.

Watch snapshot progress with:

```
kubectl -n mantle-hoodi logs -f sts/hoodi-mantle-node-el -c fetch-snapshot
```

## Check Installation Result

```
kubectl -n mantle-hoodi get pods -w
kubectl -n mantle-hoodi logs -f sts/hoodi-mantle-node-el
kubectl -n mantle-hoodi logs -f sts/hoodi-mantle-node-op-node

# Query block height
kubectl -n mantle-hoodi port-forward svc/hoodi-mantle-node-el 8545:8545 &
cast bn

# Compare against the official Mantle hoodi RPC
cast bn --rpc-url https://rpc.hoodi.mantle.xyz

# Sync status from op-node
kubectl -n mantle-hoodi port-forward svc/hoodi-mantle-node-op-node 9545:8545 &
cast rpc optimism_syncStatus --rpc-url localhost:9545 | jq .safe_l2.number
cast rpc optimism_syncStatus --rpc-url localhost:9545 | jq .finalized_l2.number
```

## Upgrade

```
git pull
helm upgrade hoodi ./chart/mantle-node \
  -f ./chart/mantle-node/hoodi.values.yaml \
  --namespace mantle-hoodi
```

See [`chart/mantle-node/README.md`](chart/mantle-node/README.md) for the
full list of values and the same-chart presets for Sepolia and Mainnet
(`sepolia.values.yaml`, `mainnet.values.yaml`).
