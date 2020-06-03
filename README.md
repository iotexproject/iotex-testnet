# IoTeX Delegate Manual

## News
- We have upgraded testnet and mainnet to `v1.0.0`. The breaking changes which will be activated on mainnet at block height 5157001.
- We have upgraded testnet to `v1.0.0-rc`. This version introduces native staking on IoTeX chain.
- We have upgraded testnet and mainnet to `v0.11.1`. A blocker indexer racing issue is fixed. The breaking changes which will still be activated on mainnet at block height 4478761. 
- We have upgraded testnet and mainnet to `v0.11.0`. This version introduces delegates probation. It contains breaking changes which will be activated on mainnet at block height 4478761. 
- We have upgraded testnet to `v0.11.0-rc`. This version introduces probation.
- We added auto upgrade feature in our upgrader. Auto upgrade will check if there is an avilable upgrade every 3 days and automactilly doing the upgrade.
- We have upgraded testnet and mainnet to `v0.10.3`. This version improve stability on reading native staking buckets.
- We have upgraded mainnet to `v0.10.2`. It contains breaking changes which will be activated on block height 3238921. Delegates must upgrade your node to the new version before that.
- We have upgraded testnet to `v0.10.2`. This version correct a gas limit issue for reading native vote bucket.
- We have upgraded mainnet to `v0.10.0`. It contains breaking changes which will be activated on block height 1816201. Delegates must upgrade your node to the new version before that.
- We have upgraded testnet to `v0.10.0`. This version reduce block interval to 5s.
- We found a bug in `v0.9.0` which may cause the nodes not agree on the delegates list. We already pushed out a build `v0.9.2` to address this issue.
- `v0.9.0` is released, so that delegates should upgrade their softwares to this new version. The fork will happen at block height 1641601. Before restarting with `v0.9.0` docker image, please re-pull the up-to-date mainnet genesis config file first. It's a MUST step for this upgrade. In addtion, note that this upgrade will result in db migration upon restart which could takes 30min to 1hr to complete. Therefore, please upgrade when the delegate node is not in the active consensus epoch.
- We have reset testnet, and deployed `v0.8.3`, and finally upgraded it to `v0.9.0`. The genesis config file has been updated as well.
- We have upgraded mainnet to `v0.8.3`. It contains breaking changes which will be activated on block height 1512001. Delegates must upgrade your node to the new version before that.
- We have upgraded testnet to `v0.8.4`.
- We have upgraded testnet to `v0.8.3`, which contains the new error code, and consensus improvement.

## Index

- [Release Status](#status)
- [Join MainNet](#mainnet)
- [Join Mainnet without using Docker](#mainnet_native)
- [Join TestNet](#testnet)
- [Interact with Blockchain](#ioctl)
- [Operate Your Node](#ops)
- [Upgrade Your Node（One Line Upgrader）](#upgrade)
- [Q&A](#qa)


## <a name="status"/>Release Status

Here are the software versions we use:

- MainNet: v1.0.0
- TestNet: v1.0.0

## <a name="mainnet"/>Join MainNet
This is the recommended way to start an IoTeX node

1. Pull the docker image:

```
docker pull iotex/iotex-core:v1.0.0
```

2. Set the environment with the following commands:

```
mkdir -p ~/iotex-var
cd ~/iotex-var

export IOTEX_HOME=$PWD

mkdir -p $IOTEX_HOME/data
mkdir -p $IOTEX_HOME/log
mkdir -p $IOTEX_HOME/etc

curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/v1.0.0/config_mainnet.yaml > $IOTEX_HOME/etc/config.yaml
curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/v1.0.0/genesis_mainnet.yaml > $IOTEX_HOME/etc/genesis.yaml
```

3. Edit `$IOTEX_HOME/etc/config.yaml`, look for `externalHost` and `producerPrivKey`, uncomment the lines and fill in your external IP and private key. You also need to replace the `gravityChainAPIs` to use your own infura project key. And make sure you enabled archive data access if you need to access Ethereum archive node data.

4. To start from a snapshot, run the following commands:

```
curl -L https://t.iotex.me/mainnet-data-latest > $IOTEX_HOME/data.tar.gz
tar -xzf data.tar.gz
```
We will update the snapshot once a day. If you plan to run your node as a gateway, please use the snapshot with index data:
https://t.iotex.me/mainnet-data-with-idx-latest.

(Optional) If you want to sync the chain from 0 height, please change the `gravityChainAPIs` in config.yaml to use your infura key with archieve mode supported or change the API endpoint to an Ethereum node with archieve data which you can access.

5. Run the following command to start a node:

```
docker run -d --restart on-failure --name iotex \
        -p 4689:4689 \
        -p 8080:8080 \
        -v=$IOTEX_HOME/data:/var/data:rw \
        -v=$IOTEX_HOME/log:/var/log:rw \
        -v=$IOTEX_HOME/etc/config.yaml:/etc/iotex/config_override.yaml:ro \
        -v=$IOTEX_HOME/etc/genesis.yaml:/etc/iotex/genesis.yaml:ro \
        iotex/iotex-core:v1.0.0 \
        iotex-server \
        -config-path=/etc/iotex/config_override.yaml \
        -genesis-path=/etc/iotex/genesis.yaml
```

Now your node should be started successfully.

If you want to also make your node be a gateway, which could process API requests from users, use the following command instead:

```
docker run -d --restart on-failure --name iotex \
        -p 4689:4689 \
        -p 14014:14014 \
        -p 8080:8080 \
        -v=$IOTEX_HOME/data:/var/data:rw \
        -v=$IOTEX_HOME/log:/var/log:rw \
        -v=$IOTEX_HOME/etc/config.yaml:/etc/iotex/config_override.yaml:ro \
        -v=$IOTEX_HOME/etc/genesis.yaml:/etc/iotex/genesis.yaml:ro \
        iotex/iotex-core:v1.0.0 \
        iotex-server \
        -config-path=/etc/iotex/config_override.yaml \
        -genesis-path=/etc/iotex/genesis.yaml \
        -plugin=gateway
```

6. Make sure TCP ports 4689, 8080 (also 14014 if used) are open on your firewall and load balancer (if any).

## <a name="mainnet_native"/>Join Mainnet without using Docker
This is not the preferred way to start an IoTeX node

1. Set the environment with the following commands:

Same as [Join MainNet](#mainnet) step 2

2. Build server binary:

```
git clone https://github.com/iotexproject/iotex-core.git
cd iotex-core
git checkout checkout v1.0.0

// optional
export GOPROXY=https://goproxy.io
go mod download
make clean build-all
cp ./bin/server $IOTEX_HOME/iotex-server
```

3. Edit configs

Same as [Join MainNet](#mainnet) step 3. Also make sure you update all db paths in config.yaml to correct location if you don't put them under `/var/data/`

4. Start from a snapshot

Same as [Join MainNet](#mainnet) step 4

5. Run the following command to start a node:

```
nohup $IOTEX_HOME/iotex-server \
        -config-path=$IOTEX_HOME/etc/iotex/config.yaml \
        -genesis-path=$IOTEX_HOME/etc/iotex/genesis.yaml &
```

Now your node should be started successfully.

If you want to also make your node be a gateway, which could process API requests from users, use the following command instead:

```
nohup $IOTEX_HOME/iotex-server \
        -config-path=$IOTEX_HOME/etc/iotex/config.yaml \
        -genesis-path=$IOTEX_HOME/etc/iotex/genesis.yaml \
        -plugin=gateway &
```

6. Make sure TCP ports 4689, 8080 (also 14014 if used) are open on your firewall and load balancer (if any).

## <a name="testnet"/>Join TestNet

There's almost no difference to join TestNet, but in step 2, you need to use the config and genesis files for TestNet:

```
curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/v1.0.0/config_testnet.yaml > $IOTEX_HOME/etc/config.yaml
curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/v1.0.0/genesis_testnet.yaml > $IOTEX_HOME/etc/genesis.yaml
```

In step 4, you need to use the snapshot for TestNet: https://t.iotex.me/testnet-data-latest and https://t.iotex.me/testnet-data-with-idx-latest. 

In step 5, you need to replace the docker image tag in the command with `v1.0.0`.

## <a name="ioctl"/>Interact with Blockchain


You can install `ioctl` (a command-line interface for interacting with IoTeX blockchain)

```
curl https://raw.githubusercontent.com/iotexproject/iotex-core/master/install-cli.sh | sh
```

You can point `ioctl` to your node (if you enable the gateway plugin):

```
ioctl config set endpoint localhost:14014 --insecure
```

Or you can point it to our nodes:

- MainNet secure: `api.iotex.one:443`
- MainNet insecure: `api.iotex.one:80`
- TestNet secure: `api.testnet.iotex.one:443`
- TestNet insecure: `api.testnet.iotex.one:80`

If you want to set an insecure endpoint, you need to add `--insecure` option.

Generate key:
```
ioctl account create
```

Get consensus delegates of current epoch:
```
ioctl node delegate
```

Refer to [CLI document](https://github.com/iotexproject/iotex-core/blob/master/ioctl/README.md) for more details.

### Other Commonly Used Commands

Claim reward:
```
ioctl action claim ${amountInIOTX} -l 10000 -p 1 -s ${ioAddress|alias}
```

Exchange IoTeX native token to ERC20 token on Ethereum via Tube service:
```
ioctl action invoke io1p99pprm79rftj4r6kenfjcp8jkp6zc6mytuah5 ${amountInIOTX} -s ${ioAddress|alias} -l 400000 -p 1 -b d0e30db0
```
Click [IoTeX Tube docs](https://github.com/iotexproject/iotex-bootstrap/blob/master/tube/tube.md) for detailed documentation of the tube service.

## <a name="ops"/>Operate Your Node

### Checking Node log

Container logs can be accessed with the following command. 

```
docker logs iotex
```

Content can be filtered with:

```
docker logs -f --tail 100 iotex |grep --color -E "epoch|height|error|rolldposctx"
```

### Stop and remove container

When starting the container with ```--name=iotex```, you must remove the old container before a new build.

```
docker stop iotex
docker rm iotex
```

### Pause and Restarting container

Container can be "stopped" and "restarted" with:

```
docker stop iotex
docker start iotex
```

## <a name="upgrade"/>Upgrade Your Node(One Line Upgrader)
Make sure you have `$IOTEX_HOME` already set up, and all the files(configs, dbs etc) are placed in the right locations (Please refer to the Join MainNet section).

To upgrade mainnet node, use following command. By default, it will upgrade to latest mainnet version.
```bash
sudo bash # If your docker requires root privilege
bash <(curl -s https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/scripts/setup_fullnode.sh)
```

To enable gateway on mainnet
```bash
sudo bash # If your docker requires root privilege
bash <(curl -s https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/scripts/setup_fullnode.sh) plugin=gateway
```

To upgarde testnet node, just add `testnet` in the end of the command.
```bash
sudo bash # If your docker requires root privilege
bash <(curl -s https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/scripts/setup_fullnode.sh) testnet
```

Currently, auto upgrade is turned on by default. To disable this feature, enter `N` when asked following question:
```bash
Do you want to auto update the node [Y/N] (Default: Y)? N
```

To stop auto upgdrade cron job and iotex server program, you can run
```bash
sudo bash # If your docker requires root privilege
bash <(curl -s https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/scripts/stop_fullnode.sh)
```

## <a name="qa"/>Q&A
Please refer [here](https://github.com/iotexproject/iotex-bootstrap/wiki/Q&A) for Q&A.
