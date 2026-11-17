# Simple Mantle Node

## Required Software

- [docker](https://docs.docker.com/engine/install/)
- [node](https://nodejs.org/en/download/)
- [foundry](https://github.com/foundry-rs/foundry/releases)
- [zstd](https://github.com/facebook/zstd)
- [aira2](https://aria2.github.io/)
## Recommended Hardware

- 16GB+ RAM
- 8C+ CPU
- 5T+ disk(HDD works for now, SSD is better)
- 10mb/s+ download

## Installation and Setup Instructions

### Init to generate the 'jwt_secret_txt' file and the 'p2p_node_key_txt'

```sh
cd networks/


mkdir -p mainnet/secret

node -e "console.log(require('crypto').randomBytes(32).toString('hex'))" > mainnet/secret/jwt_secret_txt

cast w n |grep -i "Private Key" |awk -F ": " '{print $2}' |sed 's/0x//' > mainnet/secret/p2p_node_key_txt
```

### Operating the Node

#### Download latest snapshot from mantle

After mantle upgrade to v2, you have to start the node with latest snapshot.

You can choose different types of node running modes (fullnode or archive) based on your needs, thus selecting different snapshots for node synchronization. Additionally, we provide download links for snapshots in different regions to expedite your snapshot downloads. (Given that fullnode snapshots are relatively small, we will not provide additional download links.) Currently supported regions include:

You need to get the latest tarball date first:
```sh
MAINNET_CURRENT_TARBALL_DATE=`curl https://s3.ap-southeast-1.amazonaws.com/snapshot.mantle.xyz/current.info`
```

Then replace the MAINNET_CURRENT_TARBALL_DATE in the link below:
- **Archive**
  - **Asia:** https://s3.ap-southeast-1.amazonaws.com/snapshot.mantle.xyz/${MAINNET_CURRENT_TARBALL_DATE}-mainnet-chaindata.tar.zst
- **Fullnode**
  - **Asia:** https://s3.ap-southeast-1.amazonaws.com/snapshot.mantle.xyz/${MAINNET_CURRENT_TARBALL_DATE}-mainnet-full-chaindata.tar.zst

example:

```sh
mkdir -p ./data/mainnet-geth

# Download latest snapshot tarball
# You can choose one of two ways to download，Using aria2c to download can improve download speed, but you need to install aria2
MAINNET_CURRENT_TARBALL_DATE=`curl https://s3.ap-southeast-1.amazonaws.com/snapshot.mantle.xyz/current.info`
1.
wget -c https://s3.ap-southeast-1.amazonaws.com/snapshot.mantle.xyz/${MAINNET_CURRENT_TARBALL_DATE}-mainnet-chaindata.tar.zst
2.
aria2c -x 16 -s 16 -k 100M https://s3.ap-southeast-1.amazonaws.com/snapshot.mantle.xyz/${MAINNET_CURRENT_TARBALL_DATE}-mainnet-chaindata.tar.zst

#Then you can verify your download
MAINNET_CURRENT_TARBALL_CHECKSUM=`curl https://s3.ap-southeast-1.amazonaws.com/snapshot.mantle.xyz/${MAINNET_CURRENT_TARBALL_DATE}-mainnet-chaindata.tar.zst.sha256sum | awk '{print $1}'`
echo "${MAINNET_CURRENT_TARBALL_CHECKSUM} *${MAINNET_CURRENT_TARBALL_DATE}-mainnet-chaindata.tar.zst" | shasum -a 256 --check

# You should get the following output:
# ${MAINNET_CURRENT_TARBALL_DATE}-mainnet-chaindata.tar.zst: OK

# unzip snapshot to the ledger path:
tar --use-compress-program=unzstd -xvf ${MAINNET_CURRENT_TARBALL_DATE}-mainnet-chaindata.tar.zst -C ./data/mainnet-geth

```

Check the data was unarchived successfully:

```sh
$ ls ./data/mainnet-geth
chaindata
```

#### Start

```sh
export L1_RPC_MAINNET='https://rpc.ankr.com/eth'  #please replace
docker-compose -f docker-compose-mainnetv2-upgrade-da-indexer.yml up -d
```

Will start the node in a detatched shell (`-d`), meaning the node will continue to run in the background.
You will need to run this again if you ever turn your machine off.

The first time you start the node it synchronizes from regenesis (December 1th, 2022) to the present.
This process takes hours.

NOTE: this default without history data of v1 if you want to query mantle v1 data:

besides v2 snapshot(you still need untar v2 snapshot to ./data/mainnet-geth) , you also need download v1 snapshot and start with docker-compose-mainnetv1.yml,this will give you a full data mantle rpc but need more disk :
``` 
cd networks
wget https://s3.ap-southeast-1.amazonaws.com/snapshot.mantle.xyz/historyrpcdata-mainnet-chaindata.tar.zst

tar --use-compress-program=unzstd -xvf historyrpcdata-mainnet-chaindata.tar.zst -C ./data/gethv1
docker-compose -f docker-compose-mainnetv1.yml up -d
``` 

#### Start with L1 beacon chain（recommend）

use L1 beacon chain to pull the data for rollup node, 

you need to edit L1_BEACON_MAINNET and L1_RPC_MAINNET

L1_BEACON_MAINNET is for querying data from eth blob

then start with

```
export L1_RPC_MAINNET='https://rpc.ankr.com/eth'  #please replace
export L1_BEACON_MAINNET='https://eth-beacon-chain.drpc.org/rest/'  #please replace
docker-compose -f docker-compose-mainnetv2-upgrade-beacon.yml up -d 
```

#### Start a reth-based node (alternative execution client)

If you'd rather run [reth](https://github.com/paradigmxyz/reth) instead of op-geth as the execution client, use `docker-compose-mainnet-reth.yml`. It uses the L1 beacon chain to pull data for the rollup node (same as above), so you need `L1_RPC_MAINNET` and `L1_BEACON_MAINNET` set the same way.

Recommended hardware for the reth node: 16GB+ RAM, 8C+ CPU, 500GB+ SSD. The compressed snapshot is ~55GB.

reth only starts from the official reth snapshot (published weekly) — there's no genesis-sync path, and the reth service refuses to start if `./data/mainnet-reth` is empty. Restore the snapshot before the first `up -d`:

```sh
mkdir -p ./data/mainnet-reth

# Download the latest official reth snapshot
MAINNET_RETH_TARBALL_DATE=`curl https://s3.ap-southeast-1.amazonaws.com/snapshot.mantle.xyz/current-reth.info`
wget -c https://s3.ap-southeast-1.amazonaws.com/snapshot.mantle.xyz/${MAINNET_RETH_TARBALL_DATE}-mainnet-reth.tar.zst
# or, faster: aria2c -x 16 -s 16 -k 100M https://s3.ap-southeast-1.amazonaws.com/snapshot.mantle.xyz/${MAINNET_RETH_TARBALL_DATE}-mainnet-reth.tar.zst

# Then you can verify your download
MAINNET_RETH_TARBALL_CHECKSUM=`curl https://s3.ap-southeast-1.amazonaws.com/snapshot.mantle.xyz/${MAINNET_RETH_TARBALL_DATE}-mainnet-reth.tar.zst.sha256sum | awk '{print $1}'`
echo "${MAINNET_RETH_TARBALL_CHECKSUM} *${MAINNET_RETH_TARBALL_DATE}-mainnet-reth.tar.zst" | shasum -a 256 --check

# You should get the following output:
# ${MAINNET_RETH_TARBALL_DATE}-mainnet-reth.tar.zst: OK

# Unzip snapshot to the ledger path
tar --use-compress-program=unzstd -xvf ${MAINNET_RETH_TARBALL_DATE}-mainnet-reth.tar.zst -C ./data/mainnet-reth
```

Check the data was unarchived successfully:

```sh
$ ls ./data/mainnet-reth
db  ...
```

Then bring the stack up:

```sh
export L1_RPC_MAINNET='https://rpc.ankr.com/eth'  #please replace
export L1_BEACON_MAINNET='https://eth-beacon-chain.drpc.org/rest/'  #please replace

docker-compose -f docker-compose-mainnet-reth.yml up -d
```

`L1_RPC_MAINNET` / `L1_BEACON_MAINNET` must be `export`-ed in the **same shell session** right before `up -d`: if they're missing, docker-compose silently substitutes an empty string and op-node can't sync from L1.

**Query through the `proxyd` service, not `op-reth` directly** (`${PROXYD_HTTP_PORT:-1545}` / `${PROXYD_WS_PORT:-1546}` on the host, default `http://localhost:1545`). reth only has chain data from block `87370342` onward (where the official reth snapshot starts), and a few methods (`eth_getProof`, `debug_trace*`) are unreliable on reth regardless of height. `proxyd` forwards both cases to the public RPC (`rpc.mantle.xyz`) instead of failing locally — see `mainnet/proxyd.toml` for the exact routing rules. Querying `op-reth`'s own port directly (`${VERIFIER_HTTP_PORT:-8545}`) skips this fallback and will error on pre-cutoff blocks or those methods.

Check that it's syncing:

```sh
# local height (through proxyd) vs. the public RPC
cast bn --rpc-url http://localhost:1545
cast bn --rpc-url https://rpc.mantle.xyz

# rollup node sync status; safe_l2 / finalized_l2 may take ~30 minutes to start moving
cast rpc optimism_syncStatus --rpc-url localhost:9545 | jq .safe_l2.number

# confirm the running image tags (op-reth mantle-v1.6.3, op-node v1.6.2)
docker-compose -f docker-compose-mainnet-reth.yml images
```

To stop / restart / view logs, use the same commands as below with `-f docker-compose-mainnet-reth.yml`; the services are `op-reth`, `proxyd` and `op-node`. To restore from a newer snapshot, `down`, `rm -rf ./data/mainnet-reth`, and repeat the download / unzip steps above.

#### Stop

```sh
docker-compose -f docker-compose-mainnetv2-upgrade-da-indexer.yml down

```

Will shut down the node without wiping any volumes.
You can safely run this command and then restart the node again.

#### Wipe

```sh
docker-compose -f docker-compose-mainnetv2-upgrade-da-indexer.yml down -v
```

Will completely wipe the node by removing the volumes that were created for each container.
Note that this is a destructive action, be very careful!

#### Logs

```sh
docker-compose logs <service name>
```

Will display the logs for a given service.
You can also follow along with the logs for a service in real time by adding the flag `-f`.

The available services are:

- [`op-geth`](#mantle-node)
- [`op-node`](#mantle-node)


# Upgrade for historical user
> **Note:** When upgrading, please follow the correct update order: update **mantle-op-geth** first, then update **mantle-op-node**. Reversing this order may cause unexpected issues.
> 
> ⚠️ **Important for v1.5.4 Upgrade:** When updating to v1.5.4 version, you must ensure that mantle-op-geth starts before mantle-op-node. Failure to follow this order may cause chain fork. If a fork occurs, please rebuild the RPC node by following the full setup instructions in this document.

## 1 Stop historical node

```
docker-compose -f docker-compose-mainnetv2-upgrade-da-indexer.yml down
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
export L1_RPC_MAINNET='https://rpc.ankr.com/eth'  #please replace
docker-compose -f docker-compose-mainnetv2-upgrade-da-indexer.yml up -d
```
NOTE: this default without history data of v1 if you want to query mantle v1 data:

besides v2 snapshot(you still need untar v2 snapshot to ./data/mainnet-geth) , you also need download v1 snapshot and start with docker-compose-mainnetv1.yml,this will give you a full data mantle rpc but need more disk :
``` 
cd networks
wget https://s3.ap-southeast-1.amazonaws.com/snapshot.mantle.xyz/historyrpcdata-mainnet-chaindata.tar.zst

tar --use-compress-program=unzstd -xvf historyrpcdata-mainnet-chaindata.tar.zst -C ./data/gethv1
docker-compose -f docker-compose-mainnetv1.yml up -d
``` 

### 3.2 start with L1 beacon chain（recommend）

use L1 beacon chain to pull the data for rollup node

you need to edit L1_BEACON_MAINNET and L1_RPC_MAINNET

L1_BEACON_MAINNET is for querying data from eth blob


then start with

```
export L1_RPC_MAINNET='https://rpc.ankr.com/eth'  #please replace
export L1_BEACON_MAINNET='https://eth-beacon-chain.drpc.org/rest/'  #please replace
docker-compose -f docker-compose-mainnetv2-upgrade-beacon.yml up -d 
```

## How To Check If The Deployment Is Successful

### Check Service

If the service status is 'up,' it means that the service has started without any issues.

```sh
docker-compose -f docker-compose-mainnetv2-upgrade-da-indexer.yml ps
```

### Check Data

Use the command 'cast bn' to execute multiple times and check if the height increases.

example:

```sh
cast bn
cast bn --rpc-url  https://rpc.mantle.xyz
```

Use the command 'cast rpc optimism_syncStatus' to execute multiple times and check if the safe_l2 and inalized_l2 increases.
It may need to be increased after thirty minutes

example:

```sh
cast rpc optimism_syncStatus --rpc-url localhost:9545 |jq .finalized_l2.number

cast rpc optimism_syncStatus --rpc-url localhost:9545 |jq .safe_l2.number
```
