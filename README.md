# tendermint_analysis

## 区块链开发过程

### 第一代：fork 比特币源码

在最早的时候，如果想要开发一条区块链的话，最简单的方式就是fork比特币的代码库。比特币代码库具有以下特点：

- Payment，可以实现交易转账
- UTXO，采用 UTXO账户模型
- Fee by tx size，根据交易大小收取一定的费用
- Bitcoin script，支持简单的比特币脚本
- Proof of work，采用工作量证明的共识引擎

### 第二代：Ethereum Smart Contracts

第二代采用的是以以太坊为代表的智能合约，在以太坊上面，大家可以用智能合约语言去编写智能合约。其具有以下特点：

- Ether fee coin，它用以太来做交易费用
- Account model，采用账户模型
- Patricia tries，
- EVM，有以太坊虚拟机
- Proof of Work

### 第三代

随着区块技术的发展，以前的区块链应用就出现了一些问题：代码复用困难、限制了应用开发语言、速度和扩展性有待提高。在第三代里出现了许多可以用作开发的优秀项目，其中一个典型的代表就是cosmos SDK。而cosmos SDK所依赖的也正是Tendermint。cosmos SDK具有以下特点：

- Secure，本身安全高效
- Modular，采用模块化开发
- Extensible，扩展性比较好
- Proof of Stake，共识算法是POS

## Tendermint 和 Cosmos 历史

- 2014年，Jae Kwon 与 Ethan Buchman、Zarko Milosevic联合创建了Tendermint。
- 2014年，Jae Kwon 发表了《Tendermint: 无挖矿共识算法》白皮书。Tendermint 的基本思想是允许大量分布式节点就共识达成一致，而无需中本聪共识依赖的 PoW 工作量证明。
- 2014年，成立的跨链项目Cosmos
- 2016年6月，为解决跨链问题而发起的Cosmos项目第一版白皮书
- 2017年4月，Tendermint团队在不到半小时的时间筹集了1600多万美元，完成筹资不到两个星期，Tendermint团队就开始建立Cosmos中国社区。
- 2018年2月，上线Cosmos软件开发工具包（SDK）。
- 2018年3月，币安公链Binance Chain 宣布将构建在 Cosmos 的 Tendermint 协议之上，采用 DPoS 和 BFT 共识，其去中心化交易所 DEX 也将基于Cosmos的跨链协议
- 2019年3月14日，Cosmos主网成功上线

## Tendermint 介绍

Tendermint Core 是一个区块链应用平台; 相当于提供了区块链应用程序的 Web 服务器、数据库以及用来开发区块链应用的所需的库。就像为 Web 服务器服务Web 应用程序一样, Tendermint 服务于区块链应用。

具体来讲，Tenermint是个能够在多机器上安全一致地复制应用程序的软件。 而区块链的核心就是一个复制的确定性状态机。在Tendermint中，状态机就相当于应用程序，而应用程序用于处理交易。最终保证了交易的一致性。 那么Tendermint如何做到安全一致的复制应用程序的呢，那就是通过BFT共识协议。

Tendermint另一个重要的技术组件是ABCI，由于Tendermint对共识引擎和P2P网络封装成了Tendermint Core，应用程序需要通过ABCI（Application Blockchain Interface）与Tendermint Core 进行交互，以便完成交易的整个流程。

所以总体来看，Tendermint主要由两部分组成：

- Tendermint Core: 主要包括区块链共识引擎、P2P网络
- ABCI（Application Blockchain Interface）：负责应用程序和Tendermint Core进行通信。为一个协议，支持任意类型的语言进行实现。

架构图为：

![](D:\BlockChain\note\images\Tendermint架构.PNG)

再简单来讲，Tendermint可以理解为一个模块化的区块链软件框架，支持开发者个性化定制自己的区块链，而又不需要考虑共识算法以及网络传输的实现。

Tendermint的工作流程如下图所示：

![](D:\BlockChain\note\images\交易流程.jpg)

## ABCI

ABCI是Tendermint(状态机复制引擎)和应用程序(实际的状态机)之间的接口。它由一组方法组成，其中每个方法都有相应的Request和Response消息类型。Tendermint通过发送Request消息和接收Response消息来调用ABCI应用程序上的ABCI方法。任意的语言只要实现了ABCI定义的接口，就可以与Tendermint进行交互。

ABCI的方法分为四个连接：

- consensus connection（负责区块的执行）：InitChain、BeginBlock、DeliverTx、EndBlock、Commit
- Mempool connection（验证交易）：CheckTx
- Info connection（初始化和查询）：Info、Query
- Snapshot connection（快照）：ListSnapshots、LoadSnapshotChunk、OfferSnapshot、ApplySnapshotChunk

方法的更具体的说明：https://docs.tendermint.com/master/spec/abci/abci.html

根据业务需求编写完应用程序后，就可以与Tendermint Core 进行交互了。接下来看一下RPC。

## RPC

这个视频讲解了RPC原理，如果有需要小伙伴可以去看看，这里就不再讲解了：https://www.bilibili.com/video/BV11i4y1N7LQ?p=2

官方RPC API实例：https://docs.tendermint.com/master/spec/rpc/

## P2P

p2p网络概念：https://blog.csdn.net/qaz540411484/article/details/80831196

Tendermint的P2P网络借鉴了比特币的对等发现协议，更准确的说，Tendermint是采用了BTCD的P2P地址簿（Address Book）机制。当连接建立后，新节点将自身的Address信息（包含IP，Port、ID等）发送给相邻节点，相邻节点接收到信息后加入到自己的地址簿，再将此条Address信息转播给它的相邻节点

此外为了保证节点之间数据传输的安全性，Tendermint采用了基于Station-to-Station协议的认证加密方案，此协议是一种密钥协商方案，基于经典DH算法，并提供相互密钥和实体认证。大致流程如下：　

1. 每一个节点都必须生成一对ED25519密钥对作为自己的ID
2. 当两个节点建立起TCP连接时，两者都会生成一个临时的ED25519密钥对，并把临时公钥发给对方
3. 两个节点分别将自己的私钥和对方的临时公钥相乘，得到共享密钥。这个共享密钥对称加密密钥
4. 将两个临时公钥以一定规则进行排序，并将两个临时公钥拼接起来后使用Ripemd160进行哈希处理，后面填充4个0，这样可以得到一个24字节的随机数
5. 得到的随机数作为加密种子，但为了保证相同的随机数不会被相同的私钥使用两次，我们将随机数最后一个bit置为1，这样就得到两个随机数，同时约定排序更高的公钥使用反转过的随机数来加密自己的消息，而另外一个用于解密对方节点的消息
6. 使用排序的临时公钥拼接起来，并进行SHA256哈希，得到一个挑战码
7. 每个节点都使用自己的私钥对挑战码进行签名，并将自己的公钥和签名发给其他节点校验
8. 通过校验之后，双方的认证就成功了。后续的通信就使用共享密钥和随机数进行加密，保护数据的安全

Tendermint的peer都以公开密钥的形式保持长期持久的身份证明。每个peer都有一个定义为peer的ID。其中ID = peer.PubKey.Address()。

所有p2p连接都使用TCP。在与对等端建立成功的TCP连接后，进行两次握手:一次用于验证加密，一次用于Tendermint版本控制。两次握手都有可配置的超时。



[Tendermint P2P网络源码分析](./sourceCodeAnaylsis/P2P_analysis.md)

## Mempool

内存池的作用简单来说就是保存从其他peer或者自身受到的还未被打包的交易。并且对交易进行排序

[mempool源码分析]()

## State

state就是代表了最新提交区块的描述，主要用于区块的验证。具体可查看：[State源码分析]()

## Consensus

tendermint采用的共识机制属于一种权益证明（Proof of Stake）算法，一组验证人（Validator）代替了矿工（Miner）的角色，依据抵押的权益比例轮流出块。

由于避免了POW机制，tendermint可以实现很高效的吞吐量。根据官方说法，在合理（理想）的应用数据结构支持下，可以达到42000交易/秒。不过在真实环境下，部署在全球的100个节点进行共识沟通，实际可以达到1000交易/秒

tendermint同时是拜占庭容错的（Byzantine Fault Tolerance），因此对于3f+1个验证节点组成的区块链，即使有f个节点出现拜占庭错误，也可以保证全局正确共识的达成。同时在极端情况下，tendermint在交易安全与停机风险之间选择了安全，因此当超多f个验证节点发生故障时，系统将停止工作

tendermint共识机制的另一个特点就是其共识的最终确定性：一旦达成共识就是真的达成共识，而不是像比特币或者以太坊的共识是一种概率性质的确定性，还有可能在将来某个时刻失效。因此在tendermint中不会出现区块链分叉的情况

[consensus源码分析]()

## KVStore案例分析

[KVStore案例分析]()





## 

