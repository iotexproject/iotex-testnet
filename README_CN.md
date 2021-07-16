# IoTeX 节点候选人手册

*最新版本请参考 https://github.com/iotexproject/iotex-bootstrap/blob/master/README.md*

## 索引

- [发布状态](#status)
- [加入主网](#mainnet)
- [不使用Docker加入主网](#mainnet_native)
- [加入测试网络](#testnet)
- [与区块链交互](#ioctl)
- [节点操作](#ops)
- [节点升级](#upgrade)
- [网关插件](#gateway)
- [常见问题](#qa)


## <a name="status"/>发布状态

以下是当前我们使用的软件版本：

- 主网：v1.3.0
- 测试网：v1.3.0

## <a name="mainnet"/>加入主网

以下是启动 IoTeX 节点的推荐方式

1. 提取(pull) docker镜像

```
docker pull iotex/iotex-core:v1.3.0
```

2. 使用以下命令设置运行环境

```
mkdir -p ~/iotex-var
cd ~/iotex-var

export IOTEX_HOME=$PWD

mkdir -p $IOTEX_HOME/data
mkdir -p $IOTEX_HOME/log
mkdir -p $IOTEX_HOME/etc

curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/v1.3.0/config_mainnet.yaml > $IOTEX_HOME/etc/config.yaml
curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/v1.3.0/genesis_mainnet.yaml > $IOTEX_HOME/etc/genesis.yaml
```

3. 编辑 `$IOTEX_HOME/etc/config.yaml`, 查找 `externalHost` and `producerPrivKey`, 取消注释行并填写您的外部 IP 和私钥。如果`producerPrivKey`放空，你的节点将被分配一个随机密钥。

4. 从snapshot启动, 请运行以下命令:
```
curl -L https://t.iotex.me/mainnet-data-latest > $IOTEX_HOME/data.tar.gz
tar -xzf data.tar.gz
```

**我们将会每天更新一次snapshot**。对于高级用户，可以考虑以下三个选项：

- 选项1：如果计划将节点作为[网关](#gateway)运行，请使用带有索引数据的快照：https://t.iotex.me/mainnet-data-with-idx-latest.

- 选择2：如果计划从 0 区块高度开始同步链上数据而不中继来自以太坊旧的节点代表数据，建议使用以下命令设置旧的节点代表数据：
```
curl -L https://storage.googleapis.com/blockchain-golden/poll.mainnet.tar.gz > $IOTEX_HOME/poll.tar.gz; tar -xzf $IOTEX_HOME/poll.tar.gz --directory $IOTEX_HOME/data
```

- 选择3：如果计划从 0 区块高度开始同步链并从以太坊获取旧的节点代表数据，请更改 config.yaml 中的 `gravityChainAPIs`并在支持以太坊存档模式的情况下使用您的 infura 密钥，或将 API 端点更改为您可以访问的以太坊存档的节点。

5. 运行以下命令以启动节点:

```
docker run -d --restart on-failure --name iotex \
        -p 4689:4689 \
        -p 8080:8080 \
        -v=$IOTEX_HOME/data:/var/data:rw \
        -v=$IOTEX_HOME/log:/var/log:rw \
        -v=$IOTEX_HOME/etc/config.yaml:/etc/iotex/config_override.yaml:ro \
        -v=$IOTEX_HOME/etc/genesis.yaml:/etc/iotex/genesis.yaml:ro \
        iotex/iotex-core:v1.3.0 \
        iotex-server \
        -config-path=/etc/iotex/config_override.yaml \
        -genesis-path=/etc/iotex/genesis.yaml
```

现在您的节点应该已经被成功启动了！

如果您还希望使节点成为[网关](#gateway)，可以处理用户的API请求，请改用以下命令：
```
docker run -d --restart on-failure --name iotex \
        -p 4689:4689 \
        -p 14014:14014 \
        -p 8080:8080 \
        -v=$IOTEX_HOME/data:/var/data:rw \
        -v=$IOTEX_HOME/log:/var/log:rw \
        -v=$IOTEX_HOME/etc/config.yaml:/etc/iotex/config_override.yaml:ro \
        -v=$IOTEX_HOME/etc/genesis.yaml:/etc/iotex/genesis.yaml:ro \
        iotex/iotex-core:v1.3.0 \
        iotex-server \
        -config-path=/etc/iotex/config_override.yaml \
        -genesis-path=/etc/iotex/genesis.yaml \
        -plugin=gateway
```

6. 确保您的防火墙和负载均衡器（如果有）上的TCP端口4689, 8080（14014如果使用）已打开。

## <a name="mainnet_native"/>不使用Docker加入主网

这不是我们推荐的启动 IoTeX 节点的首选方式

1. 使用以下命令设置环境：
与[加入主网](#mainnet)的步骤 2 相同

2. 构建服务器二进制文件：
```
git clone https://github.com/iotexproject/iotex-core.git
cd iotex-core
git checkout checkout v1.3.0

// optional
export GOPROXY=https://goproxy.io
go mod download
make clean build-all
cp ./bin/server $IOTEX_HOME/iotex-server
```

3. 编辑配置
与[加入主网](#mainnet)的步骤 3 相同。如果不将它们放在 `/var/data/` 下，请确保将 config.yaml 中的所有数据库路径更新到正确的位置

4. 从snapshot启动
与[加入主网](#mainnet)的步骤 4 相同

5. 运行以下命令以启动节点:
```
nohup $IOTEX_HOME/iotex-server \
        -config-path=$IOTEX_HOME/etc/config.yaml \
        -genesis-path=$IOTEX_HOME/etc/genesis.yaml &
```
现在此节点应该已成功启动。

如果您还希望使节点成为[网关](#gateway)，可以处理用户的API请求，请改用以下命令：
```
nohup $IOTEX_HOME/iotex-server \
        -config-path=$IOTEX_HOME/etc/config.yaml \
        -genesis-path=$IOTEX_HOME/etc/genesis.yaml \
        -plugin=gateway &
```

6. 确保您的防火墙和负载均衡器（如果有）上的TCP端口4689, 8080（14014如果使用）已打开。

## <a name="testnet"/>加入测试网络

加入测试网络基本没有什么不同，只是在第二步，您需要使用以下的源文件：
```
curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/v1.3.0/config_testnet.yaml > $IOTEX_HOME/etc/config.yaml
curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/v1.3.0/genesis_testnet.yaml > $IOTEX_HOME/etc/genesis.yaml
```

在步骤四中，您需要使用针对于测试网络的snapshot:  https://t.iotex.me/testnet-data-latest 和 https://t.iotex.me/testnet-data-with-idx-latest. 如果您需要使用测试网中旧的节点代表数据（poll.db），可以在此处下载:  https://storage.googleapis.com/blockchain-golden/poll.testnet.tar.gz

在步骤五，您需要将docker镜像的标签替换成``v1.3.0``。


## <a name="ioctl"/>与区块链交互

你可以安装 `ioctl` (用于与IoTeX区块链交互的命令行界面)

```
curl https://raw.githubusercontent.com/iotexproject/iotex-core/master/install-cli.sh | sh
```

您可以将`ioctl`指向您的节点（如果您启用[网关](#gateway)插件）：
```
ioctl config set endpoint localhost:14014 --insecure
```

或者您可以将它指向我们的节点:

- 主网安全端口: `api.iotex.one:443`
- 主网安非全端口: `api.iotex.one:80`
- 测试网安全端口: `api.testnet.iotex.one:443`
- 测试网安全端口: `api.testnet.iotex.one:80`

如果你准备使用非安全端口，你需要添加`--insecure`参数。

生成密钥:
```
ioctl account create
```

获得当前epoch共识节点的数量:
```
ioctl node delegate
```

参考 [CLI document](https://github.com/iotexproject/iotex-core/blob/master/cli/ioctl/README.md) 获得更多细节

### 其他常用命令

获取奖励:
```
ioctl action claim ${amountInIOTX} -l 10000 -p 1 -s ${ioAddress|alias}
```

通过Tube服务将IoTeX令牌交换到以太坊上的ERC20令牌:

```
ioctl action invoke io1p99pprm79rftj4r6kenfjcp8jkp6zc6mytuah5 ${amountInIOTX} -s ${ioAddress|alias} -l 400000 -p 1 -b d0e30db0
```
单击 [IoTeX Tube docs](https://github.com/iotexproject/iotex-bootstrap/blob/master/tube/tube.md) 获取tube服务的详细文档。

## <a name="ops"/>节点操作

### 检查节点日志

可以使用以下命令访问容器(container)日志。

```
docker logs iotex
```

内容可以用以下命令筛选:

```
docker logs -f --tail 100 iotex |grep --color -E "epoch|height|error|rolldposctx"
```

### 停止和删除容器(container)

使用```--name=iotex```启动container时，你必须在产生一个新的container之前先移除之前的container

```
docker stop iotex
docker rm iotex
```

### 暂停和重启container

可以使用以下命令“停止”和“重新启动”container:

```
docker stop iotex
docker start iotex
```

## <a name="upgrade"/>节点升级
确保你已经设置了`$IOTEX_HOME`，并且所有文件（configs、dbs 等）都放在正确的位置（请参阅加入“主网”）。

请使用以下命令升级主网节点。 默认情况下，它将升级到最新的主网版本：
```bash
sudo bash # If your docker requires root privilege
bash <(curl -s https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/scripts/setup_fullnode.sh)
```

在主网上启用 [网关](#gateway)
```bash
sudo bash # If your docker requires root privilege
bash <(curl -s https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/scripts/setup_fullnode.sh) plugin=gateway
```

如果需要升级测试网节点，只需在命令末尾添加 `testnet`。
```bash
sudo bash # If your docker requires root privilege
bash <(curl -s https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/scripts/setup_fullnode.sh) testnet
```

目前，节点自动升级默认开启。如果需要禁用此功能，请在系统询问以下问题时输入“N”：
```bash
Do you want to auto update the node [Y/N] (Default: Y)? N
```

如果需要停止自动升级 cron job 和 iotex 服务器程序，您可以运行以下指令：
```bash
sudo bash # If your docker requires root privilege
bash <(curl -s https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/scripts/stop_fullnode.sh)
```

## <a name="gateway"/>网关插件

为服务更多详细链信息的 API 请求，启用网关插件的节点将执行额外的索引。例如区块中的操作数量或通过哈希查询操作信息。

## <a name="qa"/>常见问题

请参考 [此处](https://github.com/iotexproject/iotex-bootstrap/wiki/Q&A) 了解常见问题。
