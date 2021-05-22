# 概述
在Tendermint/mempool/doc.go文件里，有对mempool详细的解释。
- 当客户端提交一笔交易时，这笔交易首先会被mempool进行打包。而mempool的主要作用之一就是保存从其他peer或者自己收到的交易。以便其它模块的使用。
- 交易进入到mempool之前还会调用一次客户端的CheckTx() 函数。并把检查通过的交易放入到交易池中。
- 交易被放到交易池保证了交易的顺序性。
- 共识引擎通过调用Update() 和 ReapMaxBytesMaxGas() 方法来更新内存池中的交易。
- mempool的底层使用的是链表
# mempool 接口
我们先看一下在mempool/mempool.go 文件里定义了的mempool功能的接口。
```go
// 对内存池的更新需要与提交区块同步，这样应用程序就可以在提交时重置它们的瞬间状态。
type Mempool interface {
	// CheckTx对应用程序执行一个新事务，来确定它的有效性，以及是否应该将它添加到内存池。
	CheckTx(tx types.Tx, callback func(*abci.Response), txInfo TxInfo) error

	// ReapMaxBytesMaxGas从内存池中获取最多为maxBytes字节总数的事务，条件是总gasWanted必须小于maxGas。
	// 如果两个最大值都是负数，则所有返回交易的规模没有上限，返回所有可用的交易
	ReapMaxBytesMaxGas(maxBytes, maxGas int64) types.Txs

	// ReapMaxTxs从内存池中获取最多的事务。
	// 如果两个最大值都是负数，则所有返回交易的规模没有上限，返回所有可用的交易
	ReapMaxTxs(max int) types.Txs

	// Lock 锁定mempool。共识必须能持锁才能安全更新。
	Lock()

	// 解锁mempool
	Unlock()

	// Update通知内存池，给定的txs已经提交，可以被丢弃。
	// Update 应该在区块被共识提交后调用
	Update(
		blockHeight int64,
		blockTxs types.Txs,
		deliverTxResponses []*abci.ResponseDeliverTx,
		newPreFn PreCheckFunc,
		newPostFn PostCheckFunc,
	) error

	// FlushAppConn flushes the mempool connection to ensure async reqResCb calls are
	// done. E.g. from CheckTx.
	// NOTE: Lock/Unlock must be managed by caller
	FlushAppConn() error

	// Flush removes all transactions from the mempool and cache
	Flush()

	// TxsAvailable returns a channel which fires once for every height,
	// and only when transactions are available in the mempool.
	// NOTE: the returned channel may be nil if EnableTxsAvailable was not called.
	TxsAvailable() <-chan struct{}

	// EnableTxsAvailable initializes the TxsAvailable channel, ensuring it will
	// trigger once every height when transactions are available.
	EnableTxsAvailable()

	// Size returns the number of transactions in the mempool.
	Size() int

	// TxsBytes returns the total size of all txs in the mempool.
	TxsBytes() int64

	// InitWAL creates a directory for the WAL file and opens a file itself. If
	// there is an error, it will be of type *PathError.
	InitWAL() error

	// CloseWAL closes and discards the underlying WAL file.
	// Any further writes will not be relayed to disk.
	CloseWAL()
}
```
# ClistMempool
ClistMempool.go 实现了mempool中定义的接口，并添加了一些其他的功能。其主要作用就是检查交易的合法性并判断是否加入交易池。
## NewCListMempool
NewClistMempool 使用给定的配置和连接的application等参数，创建一个新的mempool。
```go
func NewCListMempool(
	config *cfg.MempoolConfig,
	proxyAppConn proxy.AppConnMempool,
	height int64,
	options ...CListMempoolOption,
) *CListMempool {
	// 初始化相关的成员变量
	mempool := &CListMempool{
		// 给定的配置
		config:        config,
		// 应用层连接
		proxyAppConn:  proxyAppConn,
		// 创建一个双向链表，用来保存交易
		txs:           clist.New(),
		height:        height,
		recheckCursor: nil,
		recheckEnd:    nil,
		logger:        log.NewNopLogger(),
		metrics:       NopMetrics(),
	}
	if config.CacheSize > 0 {
		// 创建内存池缓存
		mempool.cache = newMapTxCache(config.CacheSize)
	} else {
		mempool.cache = nopTxCache{}
	}
	// 设置了代理连接的回调函数为globalCb(req *abci.Request, res *abci.Response)
	// 因为交易池在收到交易后会把交易提交给APP，根据APP的返回来决定后续如何这个交易
	// 如何处理，所以在APP处理完后的交易后回调mempool.globalCb 进而让mempool来继续决定当前交易如何处理
	proxyAppConn.SetResponseCallback(mempool.globalCb)
	for _, option := range options {
		option(mempool)
	}
	return mempool
}
```
## globalCb
Global callback 将会在每次ABCI 响应后进行调用。
```go
func (mem *CListMempool) globalCb(req *abci.Request, res *abci.Response) {
	if mem.recheckCursor == nil {
		// 符合要求，什么也不做
		return
	}

	mem.metrics.RecheckTimes.Add(1)
	// 也是检测交易是否符合要求，符合就什么也不做
	mem.resCbRecheck(req, res)

	// update metrics
	mem.metrics.Size.Set(float64(mem.Size()))
}
```
## CheckTx
```go
func (mem *CListMempool) CheckTx(tx types.Tx, cb func(*abci.Response), txInfo TxInfo) error {
	// 并发控制，也就是说当共识引擎在进行交易池读取和更新的时候，此函数应该是阻塞的
	mem.updateMtx.RLock()
	// use defer to unlock mutex because application (*local client*) might panic
	defer mem.updateMtx.RUnlock()

	txSize := len(tx)


	// 判断交易池是否满了
	if err := mem.isFull(txSize); err != nil {
		return err
	}

	// 如果已经超过了设置的内存池则放弃加入
	if txSize > mem.config.MaxTxBytes {
		return ErrTxTooLarge{mem.config.MaxTxBytes, txSize}
	}

	if mem.preCheck != nil {
		if err := mem.preCheck(tx); err != nil {
			return ErrPreCheck{err}
		}
	}

	// NOTE: writing to the WAL and calling proxy must be done before adding tx
	// to the cache. otherwise, if either of them fails, next time CheckTx is
	// called with tx, ErrTxInCache will be returned without tx being checked at
	// all even once.
	if mem.wal != nil {
		// TODO: Notify administrators when WAL fails
		_, err := mem.wal.Write(append([]byte(tx), newline...))
		if err != nil {
			return fmt.Errorf("wal.Write: %w", err)
		}
	}

	// NOTE: proxyAppConn may error if tx buffer is full
	if err := mem.proxyAppConn.Error(); err != nil {
		return err
	}

	// 先加入交易池cache 如果cache中存在此交易则返回false
	if !mem.cache.Push(tx) {
		// Record a new sender for a tx we've already seen.
		// Note it's possible a tx is still in the cache but no longer in the mempool
		// (eg. after committing a block, txs are removed from mempool but not cache),
		// so we only record the sender for txs still in the mempool.
		if e, ok := mem.txsMap.Load(TxKey(tx)); ok {
			memTx := e.(*clist.CElement).Value.(*mempoolTx)
			_, loaded := memTx.senders.LoadOrStore(txInfo.SenderID, true)
			// TODO: consider punishing peer for dups,
			// its non-trivial since invalid txs can become valid,
			// but they can spam the same tx with little cost to them atm.
			if loaded {
				return ErrTxInCache
			}
		}

		mem.logger.Debug("tx exists already in cache", "tx_hash", tx.Hash())
		return nil
	}

	ctx := context.Background()
	if txInfo.Context != nil {
		ctx = txInfo.Context
	}

	// 此时把交易传给proxyAppConn，并得到APP检查的结果
	reqRes, err := mem.proxyAppConn.CheckTxAsync(ctx, abci.RequestCheckTx{Tx: tx})
	if err != nil {
		// 如果检查不通过，就从交易池缓存中移除这笔交易
		mem.cache.Remove(tx)
		return err
	}
	reqRes.SetCallback(mem.reqResCb(tx, txInfo.SenderID, txInfo.SenderP2PID, cb))

	return nil
}

```
# Reactor
mempool reactor主要功能是在peer之间广播包含交易的mempool。
跳过Reactor结构体和NewReactor()，我们直接来看OnStart()
## OnStart

```go
func (r *Reactor) OnStart() error {
	if !r.config.Broadcast {
		r.Logger.Info("tx broadcasting is disabled")
	}

	// 处理Mempool通道的消息，最后会调用mempool中的CheckTx方法来判断是否要加入到内存池
	go r.processMempoolCh()
	// 处理每个节点的更新
	go r.processPeerUpdates()

	return nil
}
```
下图为从客户端提交的交易添加到交易池的过程：




# 最后
至此，Tendermint mempool源码就分析完了。如果有错误之处，希望路过的大佬能够指点指点。最后推荐一位大佬的公众号，欢迎关注哦：**区块链技术栈**

另外这个地址上还有很多区块链学习资料：[https://github.com/mindcarver/blockchain_guide](https://github.com/mindcarver/blockchain_guide)

参考文章：https://gitee.com/wupeaking/tendermint_code_analysis/blob/master/Mempool%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md