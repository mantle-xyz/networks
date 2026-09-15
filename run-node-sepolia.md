# Simple Mantle Node

## Required Software

* [docker](https://docs.docker.com/engine/install/)
* [node](https://nodejs.org/en/download/)
* [foundry](https://github.com/foundry-rs/foundry/releases)
* [zstd](https://github.com/facebook/zstd)
* [aira2](https://aria2.github.io/)

## Recommended Hardware

* 16GB+ RAM

* 8C+ CPU

* 1000GB+ disk (HDD works for now, SSD is better)

* 10mb/s+ download

# Installation and Setup Instructions For New User

## Installation

### 1 Download repo

```
git clone https://github.com/mantlenetworkio/networks.git
```

### 2 Generate init file

generate the 'jwt\_secret\_txt' file and the 'p2p\_node\_key\_txt'

```
cd networks/

mkdir sepolia/secret

node -e "console.log(require('crypto').randomBytes(32).toString('hex'))" > sepolia/secret/jwt_secret_txt

cast w n |grep -i "Private Key" |awk -F ": " '{print $2}' |sed 's/0x//' > sepolia/secret/p2p_node_key_txt
```

### 3 Download the latest snapshot from mantle

We recommend that you start the node with latest shapshot, so that you don't need to wait a long time to sync data.

First, create a path for ledger:

```
mkdir -p ./data/sepolia-geth
```

Second, download the latest official snapshot:
```
# Download tarball
SEPOLIA_CURRENT_TARBALL_DATE=`curl https://s3.ap-southeast-1.amazonaws.com/snapshot.sepolia.mantle.xyz/current.info`
wget -c https://s3.ap-southeast-1.amazonaws.com/snapshot.sepolia.mantle.xyz/${SEPOLIA_CURRENT_TARBALL_DATE}-sepolia-chaindata.tar.zst

# Then you can verify your download
SEPOLIA_CURRENT_TARBALL_CHECKSUM=`curl https://s3.ap-southeast-1.amazonaws.com/snapshot.sepolia.mantle.xyz/${SEPOLIA_CURRENT_TARBALL_DATE}-sepolia-chaindata.tar.zst.sha256sum | awk '{print $1}'`
echo "${SEPOLIA_CURRENT_TARBALL_CHECKSUM} *${SEPOLIA_CURRENT_TARBALL_DATE}-sepolia-chaindata.tar.zst" | shasum -a 256 --check

# You should get the following output:
# ${SEPOLIA_CURRENT_TARBALL_DATE}-sepolia-chaindata.tar.zst: OK
```


Third, unzip snapshot to the ledger path
```
tar --use-compress-program=unzstd -xvf ${SEPOLIA_CURRENT_TARBALL_DATE}-sepolia-chaindata.tar.zst -C  ./data/sepolia-geth
```

Check the data was unarchived successfully:

```
$ ls ./data/sepolia-geth
chaindata 
```

### 4 Operating the Node

#### 4.1 Start with mantle da-indexer

use mantle da-indexer to pull the data for rollup node, and you need to set up L1_RPC_SEPOLIA


```
export L1_RPC_SEPOLIA='https://rpc.ankr.com/eth_sepolia'  #please replace
docker-compose -f docker-compose-sepolia-upgrade-da-indexer.yml up -d 
```

Will start the node in a detached shell (`-d`), meaning the node will continue to run in the background. You will need to run this again if you ever turn your machine off.

Congratulations, the node has been deployed！

#### 4.2 Start with L1 beacon chain（recommend）

use L1 beacon chain to pull the data for rollup node, 
you need set up L1_BEACON_SEPOLIA and L1_RPC_SEPOLIA 

L1_BEACON_SEPOLIA is for querying data from eth blob

```
export L1_RPC_SEPOLIA='https://rpc.ankr.com/eth_sepolia'  #please replace
export L1_BEACON_SEPOLIA='https://eth-beacon-chain-sepolia.drpc.org/rest/'  #please replace

docker-compose -f docker-compose-sepolia-upgrade-beacon.yml up -d 
```

#### 4.3 Start a reth-based node (alternative execution client)

If you'd rather run [reth](https://github.com/paradigmxyz/reth) instead of op-geth as the execution client, use `docker-compose-sepolia-reth.yml`. It uses the L1 beacon chain to pull data for the rollup node (same as 4.2), so you need `L1_RPC_SEPOLIA` and `L1_BEACON_SEPOLIA` set the same way.

reth only starts from a snapshot — there's no genesis-sync path, so restore one into `./data/sepolia-reth` before the first `up -d`:

```
mkdir -p ./data/sepolia-reth

# Download the latest official snapshot
SEPOLIA_RETH_TARBALL_DATE=`curl https://s3.ap-southeast-1.amazonaws.com/snapshot.sepolia.mantle.xyz/current-reth.info`
wget -c https://s3.ap-southeast-1.amazonaws.com/snapshot.sepolia.mantle.xyz/${SEPOLIA_RETH_TARBALL_DATE}-sepolia-reth.tar.zst

# Then you can verify your download
SEPOLIA_RETH_TARBALL_CHECKSUM=`curl https://s3.ap-southeast-1.amazonaws.com/snapshot.sepolia.mantle.xyz/${SEPOLIA_RETH_TARBALL_DATE}-sepolia-reth.tar.zst.sha256sum | awk '{print $1}'`
echo "${SEPOLIA_RETH_TARBALL_CHECKSUM} *${SEPOLIA_RETH_TARBALL_DATE}-sepolia-reth.tar.zst" | shasum -a 256 --check

# Unzip snapshot to the ledger path
tar --use-compress-program=unzstd -xvf ${SEPOLIA_RETH_TARBALL_DATE}-sepolia-reth.tar.zst -C ./data/sepolia-reth
```

Then bring the stack up:

```
export L1_RPC_SEPOLIA='https://rpc.ankr.com/eth_sepolia'  #please replace
export L1_BEACON_SEPOLIA='https://eth-beacon-chain-sepolia.drpc.org/rest/'  #please replace

docker-compose -f docker-compose-sepolia-reth.yml up -d 
```

**Query through the `proxyd` service, not `op-reth` directly** (`${PROXYD_HTTP_PORT:-1545}` / `${PROXYD_WS_PORT:-1546}` on the host, default `http://localhost:1545`). reth only has chain data from a fixed height onward (see the `init-state`/`import-op` step in `docker-compose-sepolia-reth.yml`), and a few methods (`eth_getProof`, `debug_trace*`) are unreliable on reth regardless of height. `proxyd` forwards both cases to the public RPC (`rpc.sepolia.mantle.xyz`) instead of failing locally — see `sepolia/proxyd.toml` for the exact routing rules. Querying `op-reth`'s own port directly (`${VERIFIER_HTTP_PORT:-8545}`) skips this fallback and will error on pre-cutoff blocks or those methods.

## Check Installation Result

Follow these steps to check if the installation is successful

### 1 Check Service

If the service status is 'up,' it means that the service has started without any issues.

```
docker-compose -f docker-compose-sepolia-upgrade-da-indexer.yml ps
```

### 2 Check Data

Use the command 'cast bn' to execute multiple times and check if the height increases.

example:

```
# query local op-geth latest block height
cast bn

# query latest block height from mantle rpc
cast bn --rpc-url  https://rpc.sepolia.mantle.xyz 
```

Use the command 'cast rpc optimism_syncStatus' to execute multiple times and check if the safe\_l2 and inalized\_l2 increases. It may need to be increased after thirty minutes

example:

```
cast rpc optimism_syncStatus --rpc-url localhost:9545 |jq .finalized_l2.number

cast rpc optimism_syncStatus --rpc-url localhost:9545 |jq .safe_l2.number
```



## Other useful commands for Operator

### 1 Stop

```
docker-compose -f docker-compose-sepolia-upgrade-da-indexer.yml down
```

Will shut down the node without wiping any volumes. You can safely run this command and then restart the node again.

### 2 Wipe

```
docker-compose -f docker-compose-sepolia-upgrade-da-indexer.yml down -v
```

Will completely wipe the node by removing the volumes that were created for each container. Note that this is a destructive action, be very careful!

### 3 Logs

```
docker-compose logs <service name>
```

Will display the logs for a given service. You can also follow along with the logs for a service in real time by adding the flag `-f`.

The available services are:

* [`op-geth`](https://github.com/mantlenetworkio/networks/blob/ba6e673b4164864cf40768c95382423d5756bb67/run-node-sepolia.md#mantle-node)

* [`op-node`](https://github.com/mantlenetworkio/networks/blob/ba6e673b4164864cf40768c95382423d5756bb67/run-node-sepolia.md#mantle-node)


# Upgrade for historical user
> **Note:** When upgrading, please follow the correct update order: update **mantle-op-geth** first, then update **mantle-op-node**. Reversing this order may cause unexpected issues.
> 
> ⚠️ **Important for v1.5.3 Upgrade:** When updating to v1.5.3 version, you must ensure that mantle-op-geth starts before mantle-op-node. Failure to follow this order may cause chain fork. If a fork occurs, please rebuild the RPC node by following the full setup instructions in this document.

## 1 Stop historical node

```
docker-compose -f docker-compose-sepolia.yml down
docker-compose -f docker-compose-sepolia-upgrade-da-indexer.yml down
```

## 2 Pull the latest code of this repo

```
# If your local code have changes, please use 'git stash' to cache first

git pull 
```

**If you start the node using your own way, please refer to the three files from this upgrade. Otherwise, it may cause irreversible damage to the node.**

## 3 Operating the Node

### 3.1 start with mantle da-indexer


```
export L1_RPC_SEPOLIA='https://rpc.ankr.com/eth_sepolia'  #please replace
docker-compose -f docker-compose-sepolia-upgrade-da-indexer.yml up -d 
```

### 3.2 start with L1 beacon chain（recommend）

use L1 beacon chain to pull the data for rollup node

you need to edit L1_BEACON_SEPOLIA and L1_RPC_SEPOLIA 

L1_BEACON_SEPOLIA is for querying data from eth blob


then start with

```
export L1_RPC_SEPOLIA='https://rpc.ankr.com/eth_sepolia'  #please replace
export L1_BEACON_SEPOLIA='https://eth-beacon-chain-sepolia.drpc.org/rest/'  #please replace
docker-compose -f docker-compose-sepolia-upgrade-beacon.yml up -d 
```

## 4 Check data

Use the command 'cast bn' to execute multiple times and check if the height increases.

example:

```
# query local op-geth latest block height
cast bn

# query latest block height from mantle rpc
cast bn --rpc-url  https://rpc.sepolia.mantle.xyz 
```

Use the command 'cast rpc optimism_syncStatus' to execute multiple times and check if the safe\_l2 and inalized\_l2 increases. It may need to be increased after thirty minutes

example:

```
cast rpc optimism_syncStatus --rpc-url localhost:9545 |jq .finalized_l2.number

cast rpc optimism_syncStatus --rpc-url localhost:9545 |jq .safe_l2.number
```

# Restore from snapshot

If your node's data is corrupted due to abnormal operations, please refer to the following steps for recovery

## 1 Clean up historical data

```
rm -fr ./data/sepolia-geth 
```

## 2 Download the latest data

```
mkdir -p ./data/sepolia-geth

# download the latest official snapshot
SEPOLIA_CURRENT_TARBALL_DATE=`curl https://s3.ap-southeast-1.amazonaws.com/snapshot.sepolia.mantle.xyz/current.info`
wget -c https://s3.ap-southeast-1.amazonaws.com/snapshot.sepolia.mantle.xyz/${SEPOLIA_CURRENT_TARBALL_DATE}-sepolia-chaindata.tar.zst

# unzip snapshot to the ledger path
tar --use-compress-program=unzstd -xvf ${SEPOLIA_CURRENT_TARBALL_DATE}-sepolia-chaindata.tar.zst -C  ./data/sepolia-geth
```

## 3 Start the service

If you use da-indexer

```
docker-compose -f docker-compose-sepolia-upgrade-da-indexer.yml up -d 
```

Otherwise

```
docker-compose -f docker-compose-sepolia-upgrade-beacon.yml up -d 
```
