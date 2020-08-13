# IoTeX 节点候选人手册

## 索引

- [版本状态](#status)
- [加入主网预演](#mainnet)
- [不使用Docker加入主网](#mainnet_native)
- [加入测试网络](#testnet)
- [与区块链交互](#ioctl)
- [操作您的节点](#ops)
- [升级您的节点（One Line Upgrader）](#upgrade)
- [常见问题](#qa)


## <a name="status"/>发行版本状态

这是我们正在使用的版本:

- 主网: v1.1.0
- 测试网: v1.1.0

## <a name="mainnet"/>加入主网
以下是启动IoTeX主网节点的推荐方法

1. 提取(pull) docker镜像:

```
docker pull iotex/iotex-core:v1.1.0
```

2. 使用以下命令设置运行环境:

```
mkdir -p ~/iotex-var
cd ~/iotex-var

export IOTEX_HOME=$PWD

mkdir -p $IOTEX_HOME/data
mkdir -p $IOTEX_HOME/log
mkdir -p $IOTEX_HOME/etc

curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/v1.1.0/config_mainnet.yaml > $IOTEX_HOME/etc/config.yaml
curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/v1.1.0/genesis_mainnet.yaml > $IOTEX_HOME/etc/genesis.yaml
```

3. 编辑 `$IOTEX_HOME/etc/config.yaml`, 查找 `externalHost` 和 `producerPrivKey`, 使用您的ip地址和私钥代替[...]，并且取消该行备注. 如果保持 `producerPrivKey` 为空, 则你的节点将获得一个随机分配的key

4. 如果您想从snapshot启动, 请运行以下命令:

```
curl -L https://t.iotex.me/mainnet-data-latest > $IOTEX_HOME/data.tar.gz
tar -xzf data.tar.gz
```
我们将会每天更新一次snapshot，如果计划将节点作为[网关](#gateway)运行, 请使用带有索引数据的快照:
https://t.iotex.me/mainnet-data-with-idx-latest

(可选) 如果您想要从0高度同步链数据，而不是从以太坊继承现存的节点选举数据，则可以使用以下命令设置现存的节点选举数据:

```bash
curl -L https://storage.googleapis.com/blockchain-golden/poll.mainnet.tar.gz > $IOTEX_HOME/poll.tar.gz; tar -xzf $IOTEX_HOME/poll.tar.gz --directory $IOTEX_HOME/data
```

(可选) 如果您想从0高度同步链数据，并且从以太坊获取现存的节点选举数据，请更改 config.yaml 中的`gravityChainAPIs`，从而在支持以太坊存档模式的情况下使用您的infura密钥，或者您可以将API端点更改为您可以访问的以太坊存档节点


5. 运行以下命令以启动节点:

```
docker run -d --restart on-failure --name iotex \
        -p 4689:4689 \
        -p 8080:8080 \
        -v=$IOTEX_HOME/data:/var/data:rw \
        -v=$IOTEX_HOME/log:/var/log:rw \
        -v=$IOTEX_HOME/etc/config.yaml:/etc/iotex/config_override.yaml:ro \
        -v=$IOTEX_HOME/etc/genesis.yaml:/etc/iotex/genesis.yaml:ro \
        iotex/iotex-core:v1.1.0 \
        iotex-server \
        -config-path=/etc/iotex/config_override.yaml \
        -genesis-path=/etc/iotex/genesis.yaml
```

现在您的节点应该已经被成功启动了

如果您还希望使节点成为[网关](#gateway)，可以处理用户的API请求，请改用以下命令:

```
docker run -d --restart on-failure --name iotex \
        -p 4689:4689 \
        -p 14014:14014 \
        -p 8080:8080 \
        -v=$IOTEX_HOME/data:/var/data:rw \
        -v=$IOTEX_HOME/log:/var/log:rw \
        -v=$IOTEX_HOME/etc/config.yaml:/etc/iotex/config_override.yaml:ro \
        -v=$IOTEX_HOME/etc/genesis.yaml:/etc/iotex/genesis.yaml:ro \
        iotex/iotex-core:v1.1.0 \
        iotex-server \
        -config-path=/etc/iotex/config_override.yaml \
        -genesis-path=/etc/iotex/genesis.yaml \
        -plugin=gateway
```

6. 请确保您的防火墙和负载均衡器（如果有）上的TCP端口4689, 8080（14014如果使用）已打开

## <a name="mainnet_native"/>不使用Docker加入主网
这不是启动IoTeX主网节点的推荐方式

1. 使用以下命令设置运行环境：

和 [加入主网](#mainnet) 中的第二步一样

2. 构建服务器二进制文本:

```
git clone https://github.com/iotexproject/iotex-core.git
cd iotex-core
git checkout checkout v1.1.0

// optional
export GOPROXY=https://goproxy.io
go mod download
make clean build-all
cp ./bin/server $IOTEX_HOME/iotex-server
```

3. 编辑配置

和 [加入主网](#mainnet) 中的第三步一样
如果不将其放在  `/var/data/`下，还请确保将 config.yaml 中的所有数据库路径更新为正确的位置

4. 从快照开始

和 [加入主网](#mainnet) 中的第四步一样

5. 运行以下命令以启动节点:

```
nohup $IOTEX_HOME/iotex-server \
        -config-path=$IOTEX_HOME/etc/iotex/config.yaml \
        -genesis-path=$IOTEX_HOME/etc/iotex/genesis.yaml &
```

现在，您的节点应该已成功启动

如果您还想使节点成为可以处理来自用户的API请的[网关](#gateway), 请改用以下命令：

```
nohup $IOTEX_HOME/iotex-server \
        -config-path=$IOTEX_HOME/etc/iotex/config.yaml \
        -genesis-path=$IOTEX_HOME/etc/iotex/genesis.yaml \
        -plugin=gateway &
```

6. 请确保您的防火墙和负载均衡器（如果有）上的TCP端口4689, 8080（14014如果使用）已打开

## <a name="testnet"/>加入测试网络

加入测试网络与加入主网几乎没有区别，但是在第2步中，您需要使用测试网络的config和genesis文件:

```
curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/v1.1.0/config_testnet.yaml > $IOTEX_HOME/etc/config.yaml
curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/v1.1.0/genesis_testnet.yaml > $IOTEX_HOME/etc/genesis.yaml
```

在第四步中，您需要使用针对于测试网络的快照: https://t.iotex.me/testnet-data-latest and https://t.iotex.me/testnet-data-with-idx-latest. 如果您需要用于测试网络的现存节点选举数据（poll.db），可以在此处下载: https://storage.googleapis.com/blockchain-golden/poll.testnet.tar.gz

在第五步，您需要将docker镜像的标签替换成 `v1.1.0`.

## <a name="ioctl"/>与区块链交互


你可以安装 `ioctl` (用于与IoTeX区块链交互的命令行界面)

```
curl https://raw.githubusercontent.com/iotexproject/iotex-core/master/install-cli.sh | sh
```

您可以将 `ioctl` 指向您的节点 如果您已经启用了 [网关](#gateway) ): 

```
ioctl config set endpoint localhost:14014 --insecure
```

或者您可以将它指向我们的节点:

主网安全端口: `api.iotex.one:443`
主网安非全端口: `api.iotex.one:80`
测试网安全端口: `api.testnet.iotex.one:443`
测试网安全端口: `api.testnet.iotex.one:80`

如果你准备使用非安全端口，你需要添加`--insecure` 参数

生成密钥:
```
ioctl account create
```

获得当前epoch共识节点的数量:
```
ioctl node delegate
```

参考 [CLI document](https://github.com/iotexproject/iotex-core/blob/master/ioctl/README.md) 获得更多细节

### 其他常用命令

获取奖励:
```
ioctl action claim ${amountInIOTX} -l 10000 -p 1 -s ${ioAddress|alias}
```

通过Tube服务将IoTeX令牌交换到以太坊上的ERC20令牌:
```
ioctl action invoke io1p99pprm79rftj4r6kenfjcp8jkp6zc6mytuah5 ${amountInIOTX} -s ${ioAddress|alias} -l 400000 -p 1 -b d0e30db0
```
请点击 [IoTeX Tube docs](https://github.com/iotexproject/iotex-bootstrap/blob/master/tube/tube.md) 以获取更多关于Tube服务的文档

## <a name="ops"/>操作你的节点

### 检查节点记录

可以使用以下命令访问容器(container)日志

```
docker logs iotex
```

可以使用以下命令筛选内容:

```
docker logs -f --tail 100 iotex |grep --color -E "epoch|height|error|rolldposctx"
```

### 停止和删除容器(container)

When starting the container with ```--name=iotex```, you must remove the old container before a new build.
当以`--name = iotex`启动容器(container)时，您必须在产生一个新的container之前先移除之前的container

```
docker stop iotex
docker rm iotex
```

### 暂停和重启容器(container)

可以使用以下命令“停止”和“重新启动”容器(container):

```
docker stop iotex
docker start iotex
```

## <a name="upgrade"/>升级您的节点(One Line Upgrader)
确保您已经设置了`$IOTEX_HOME` ，并且所有文件（配置文件，数据库文件等）都放置在正确的位置（请参阅[加入主网](#mainnet) 部分）

请使用以下命令升级主网节点。在默认情况下，它将升级到最新的主网版本
```bash
sudo bash # If your docker requires root privilege
bash <(curl -s https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/scripts/setup_fullnode.sh)
```

To enable [gateway](#gateway) on mainnet
在主网上启用[网关](#gateway)插件
```bash
sudo bash # If your docker requires root privilege
bash <(curl -s https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/scripts/setup_fullnode.sh) plugin=gateway
```

要升级测试网络节点，只需在命令末尾添加`testnet`。
```bash
sudo bash # If your docker requires root privilege
bash <(curl -s https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/scripts/setup_fullnode.sh) testnet
```

Currently, auto upgrade is turned on by default. To disable this feature, enter `N` when asked following question:在默认情况下，自动升级将处于打开状态。 要禁用此功能，请在询问以下问题时输入 `N`：
```bash
Do you want to auto update the node [Y/N] (Default: Y)? N
```

要停止自动升级cron job和iotex服务器程序，可以使用以下命令：
```bash
sudo bash # If your docker requires root privilege
bash <(curl -s https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/scripts/stop_fullnode.sh)
```
## <a name="gateway"/> 网关插件
启用了网关插件的节点将执行额外的索引，以完成获取更详细的链信息的API请求，例如查询块中的动作数或通过哈希查询动作

### 启用系统级操作日志的索引
系统级操作日志是一项通过合同跟踪交易的功能。 要打开此索引，请在`$IOTEX_HOME/etc/config.yaml`中将`enableSystemLog`的值更改为true

## <a name="qa"/>常见问题
请参阅 [这里](https://github.com/iotexproject/iotex-bootstrap/wiki/Q&A)





------OLD------------OLD-------------OLD-----------OLD---------

*最新版本请参考 https://github.com/iotexproject/iotex-bootstrap/blob/master/README.md*

## 最新动态
- 我们在v0.9.0中发现了一个错误，该错误可能导致节点在委托列表上不一致。我们已经推出了v0.9.1版来解决此问题。
- v0.9.0已发布，因此委托节点应将其软件升级到此新版本。该分叉将发生在块高度1641601处。在使用v0.9.0 docker image重新启动之前，请首先重新获取最新的mainnet创世区块配置文件。这是此升级的必需步骤。此外，请注意，此升级将导致重新启动时迁移数据库，这可能需要30分钟到1个小时才能完成。因此，请在委托节点不在有效共识时期时进行升级。
- 我们重置了测试网，并部署了v0.8.3，最后将其升级到v0.9.0。配置文件也已更新。
- 我们已经将主网升级到v0.8.3。它包含一些重大更改，这些更改将在块高度1512001上激活。代表必须在此之前将您的节点升级到新版本。
- 我们已经将testnet升级到v0.8.4。
- 我们已将testnet升级到v0.8.3，其中包含新的错误代码和共识改进。

## 索引

- [版本状态](#status)
- [加入主网预演](#mainnet)
- [加入测试网络](#testnet)
- [与区块链交互](#ioctl)
- [操作您的节点](#ops)


## 发行版本状态

主网：v0.9.1
测试网：v0.9.1


## 加入主网内测

1. 提取(pull) docker镜像

```
docker pull iotex/iotex-core:v0.9.1
```
请检查你的docker镜像的摘要是`ad495eee20a758402d2a7b01eee9e2fdb842be9a1786ba2fb67bf6d440c21625`。

2. 使用以下命令设置运行环境

```
mkdir -p ~/iotex-var
cd ~/iotex-var

export IOTEX_HOME=$PWD

mkdir -p $IOTEX_HOME/data
mkdir -p $IOTEX_HOME/log
mkdir -p $IOTEX_HOME/etc

curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/config_mainnet.yaml > $IOTEX_HOME/etc/config.yaml
curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/genesis_mainnet.yaml > $IOTEX_HOME/etc/genesis.yaml
```

3. 编辑 `$IOTEX_HOME/etc/config.yaml`, 查找 `externalHost` and `producerPrivKey`, 使用您的ip地址和私钥代替`[...]`，并且取消该行备注 

4. (可选) 如果您想从snapshot启动, 请运行以下命令:

```
curl -L https://t.iotex.me/data-latest > $IOTEX_HOME/data.tar.gz
tar -xzf data.tar.gz
```

我们将会每天更新一次snapshot，如果计划将节点作为网关运行，请使用带有索引数据的快照：https://t.iotex.me/mainnet-data-with-idx-latest.


5. 运行以下命令以启动节点:

```
docker run -d --restart on-failure --name iotex \
        -p 4689:4689 \
        -p 8080:8080 \
        -v=$IOTEX_HOME/data:/var/data:rw \
        -v=$IOTEX_HOME/log:/var/log:rw \
        -v=$IOTEX_HOME/etc/config.yaml:/etc/iotex/config_override.yaml:ro \
        -v=$IOTEX_HOME/etc/genesis.yaml:/etc/iotex/genesis.yaml:ro \
        iotex/iotex-core:v0.9.1 \
        iotex-server \
        -config-path=/etc/iotex/config_override.yaml \
        -genesis-path=/etc/iotex/genesis.yaml \
```

现在您的节点应该已经被成功启动了

如果您还希望使节点成为网关，可以处理用户的API请求，请改用以下命令：
```
docker run -d --restart on-failure --name iotex \
        -p 4689:4689 \
        -p 14014:14014 \
        -p 8080:8080 \
        -v=$IOTEX_HOME/data:/var/data:rw \
        -v=$IOTEX_HOME/log:/var/log:rw \
        -v=$IOTEX_HOME/etc/config.yaml:/etc/iotex/config_override.yaml:ro \
        -v=$IOTEX_HOME/etc/genesis.yaml:/etc/iotex/genesis.yaml:ro \
        iotex/iotex-core:v0.9.1 \
        iotex-server \
        -config-path=/etc/iotex/config_override.yaml \
        -genesis-path=/etc/iotex/genesis.yaml \
        -plugin=gateway
```

6. 确保您的防火墙和负载均衡器（如果有）上的TCP端口4689, 8080（14014如果使用）已打开。

## 加入测试网络

加入测试网络基本没有什么不同，只是在第二步，您需要使用以下的源文件：
```
curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/config_testnet.yaml > $IOTEX_HOME/etc/config.yaml
curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/genesis_testnet.yaml > $IOTEX_HOME/etc/genesis.yaml
```

在第四步，您需要使用针对于测试网络的snapshot:  https://t.iotex.me/testnet-data-latest 和 https://t.iotex.me/testnet-data-with-idx-latest.

在第五步，您需要将docker镜像的标签替换成``v0.9.1``。


## 与区块链交互


你可以安装 ioctl (用于与IoTeX区块链交互的命令行界面)

```
curl https://raw.githubusercontent.com/iotexproject/iotex-core/master/install-cli.sh | sh
```

您可以将`ioctl`指向您的节点
```
ioctl config set endpoint localhost:14014 --insecure
```

或者您可以将它指向我们的节点:

- 主网安全端口: api.iotex.one:443
- 主网安非全端口: api.iotex.one:80
- 测试网安全端口: api.testnet.iotex.one:443
- 测试网安全端口: api.testnet.iotex.one:80

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

## 其他常用命令
获取奖励:
```
ioctl action claim ${amountInIOTX} -l 10000 -p 1 -s ${ioAddress|alias}
```

通过Tube服务将IoTeX令牌交换到以太坊上的ERC20令牌:

```
ioctl action invoke io1p99pprm79rftj4r6kenfjcp8jkp6zc6mytuah5 ${amountInIOTX} -s ${ioAddress|alias} -l 400000 -p 1 -b d0e30db0
```

## 操作你的节点

## 检查节点记录

可以使用以下命令访问容器(container)日志。

```
docker logs iotex
```

内容可以用以下命令筛选:

```
docker logs -f --tail 100 iotex |grep --color -E "epoch|height|error|rolldposctx"
```

## 停止和删除容器(container)

你必须在产生一个新的container之前先移除之前的container

```
docker stop iotex
docker rm iotex
```

## 暂停和重启container

可以使用以下命令“停止”和“重新启动”container:

```
docker stop iotex
docker start iotex
```
