# 前言
官方文档：[https://docs.tendermint.com/master/spec/p2p/messages/](https://docs.tendermint.com/master/spec/p2p/messages/)
## message
Tendermint的P2P 中的消息分为两部分：channel和message。
## P2P配置
其配置在$TMHOME/config/config.toml文件中，[配置说明](https://docs.tendermint.com/master/nodes/configuration.html)
配置截取如下：
```toml
#######################################################
###           P2P Configuration Options             ###
#######################################################
[p2p]

# Enable the new p2p layer.
disable-legacy = false

# Select the p2p internal queue
queue-type = "priority"

# Address to listen for incoming connections
laddr = "tcp://0.0.0.0:26656"

# Address to advertise to peers for them to dial
# If empty, will use the same port as the laddr,
# and will introspect on the listener or use UPnP
# to figure out the address.
external-address = ""

# Comma separated list of seed nodes to connect to
seeds = ""

# Comma separated list of nodes to keep persistent connections to
persistent-peers = ""

# UPNP port forwarding
upnp = false

# Path to address book
addr-book-file = "config/addrbook.json"

# Set true for strict address routability rules
# Set false for private or local networks
addr-book-strict = true

# Maximum number of inbound peers
#
# TODO: Remove once p2p refactor is complete in favor of MaxConnections.
# ref: https://github.com/tendermint/tendermint/issues/5670
max-num-inbound-peers = 40

# Maximum number of outbound peers to connect to, excluding persistent peers
#
# TODO: Remove once p2p refactor is complete in favor of MaxConnections.
# ref: https://github.com/tendermint/tendermint/issues/5670
max-num-outbound-peers = 10

# Maximum number of connections (inbound and outbound).
max-connections = 64

# Rate limits the number of incoming connection attempts per IP address.
max-incoming-connection-attempts = 100

# List of node IDs, to which a connection will be (re)established ignoring any existing limits
unconditional-peer-ids = ""

# Maximum pause when redialing a persistent peer (if zero, exponential backoff is used)
persistent-peers-max-dial-period = "0s"

# Time to wait before flushing messages out on the connection
flush-throttle-timeout = "100ms"

# Maximum size of a message packet payload, in bytes
max-packet-msg-payload-size = 1400

# Rate at which packets can be sent, in bytes/second
send-rate = 5120000

# Rate at which packets can be received, in bytes/second
recv-rate = 5120000

# Set true to enable the peer-exchange reactor
pex = true

# Comma separated list of peer IDs to keep private (will not be gossiped to other peers)
private-peer-ids = ""

# Toggle to disable guard against peers connecting from the same ip.
allow-duplicate-ip = false

# Peer connection configuration.
handshake-timeout = "20s"
dial-timeout = "3s"
```
# 启用P2P
在node/node.go的NewNode里，会发现这样一段代码。
```go
// setup Transport and Switch
	// 创建switch
	sw := createSwitch(
		config, transport, p2pMetrics, mpReactorShim, bcReactorForSwitch,
		stateSyncReactorShim, csReactorShim, evReactorShim, proxyApp, nodeInfo, nodeKey, p2pLogger,
	)

	err = sw.AddPersistentPeers(splitAndTrimEmpty(config.P2P.PersistentPeers, ",", " "))
	if err != nil {
		return nil, fmt.Errorf("could not add peers from persistent-peers field: %w", err)
	}

	err = sw.AddUnconditionalPeerIDs(splitAndTrimEmpty(config.P2P.UnconditionalPeerIDs, ",", " "))
	if err != nil {
		return nil, fmt.Errorf("could not add peer ids from unconditional_peer_ids field: %w", err)
	}
	// 设置地址簿
	addrBook, err := createAddrBookAndSetOnSwitch(config, sw, p2pLogger, nodeKey)
	if err != nil {
		return nil, fmt.Errorf("could not create addrbook: %w", err)
	}

	// Optionally, start the pex reactor
	//
	// TODO:
	//
	// We need to set Seeds and PersistentPeers on the switch,
	// since it needs to be able to use these (and their DNS names)
	// even if the PEX is off. We can include the DNS name in the NetAddress,
	// but it would still be nice to have a clear list of the current "PersistentPeers"
	// somewhere that we can return with net_info.
	//
	// If PEX is on, it should handle dialing the seeds. Otherwise the switch does it.
	// Note we currently use the addrBook regardless at least for AddOurAddress
	var (
		pexReactor   *pex.Reactor
		pexReactorV2 *pex.ReactorV2
	)
	// 创建PEX
	if config.P2P.PexReactor {
		pexReactor = createPEXReactorAndAddToSwitch(addrBook, config, sw, logger)
		pexReactorV2, err = createPEXReactorV2(config, logger, peerManager, router)
		if err != nil {
			return nil, err
		}

		router.AddChannelDescriptors(pexReactor.GetChannels())
	}
```
所以说，P2P的启动入口应该是先启动Switch实例
# switch
Switch处理对等连接并公开一个API来接收传入的消息。每个“Reactor”负责处理一个/或多个“Channels”的传入消息。因此，当发送传出消息通常在对等机上执行时，传入消息在 reactor 上接收。其实大概意思就是连接各个reactor，进行信息交换。

至于reactor，在p2p/base_reactor.go里对Reactor的含义及作用进行了解释。简单来说，它就是和整个P2P网络进行交互的组件。在Tendermint中，一共有六个Reactor：mempool、blockchain、consensus、evidence、pex。
## Switch结构
```go
type Switch struct {
	// 继承BaseService，方便统一启动和停止
	service.BaseService

	// 找到P2P配置文件，所以说P2P的启动入口应该是先启动Switch实例
	config       *config.P2PConfig
	// 所有的创建的reactor集合
	reactors     map[string]Reactor
	// reactor和 channel 之间的对应关系，也是通过这个传递给peer再往下传递到MConnection
	chDescs      []*conn.ChannelDescriptor
	reactorsByCh map[byte]Reactor
	// peer 集合
	peers        *PeerSet
	dialing      *cmap.CMap
	reconnecting *cmap.CMap
	nodeInfo     NodeInfo // our node info
	nodeKey      NodeKey  // our node privkey
	addrBook     AddrBook
	// peers addresses with whom we'll maintain constant connection
	persistentPeersAddrs []*NetAddress
	unconditionalPeerIDs map[NodeID]struct{}

	transport Transport

	filterTimeout time.Duration
	peerFilters   []PeerFilterFunc
	connFilters   []ConnFilterFunc
	conns         ConnSet

	rng *rand.Rand // seed for randomizing dial times and orders

	metrics *Metrics
}
```
## 创建Switch
```go
func NewSwitch(
	cfg *config.P2PConfig,
	transport Transport,
	options ...SwitchOption,
) *Switch {
	sw := &Switch{
		config:               cfg,
		reactors:             make(map[string]Reactor),
		chDescs:              make([]*conn.ChannelDescriptor, 0),
		reactorsByCh:         make(map[byte]Reactor),
		peers:                NewPeerSet(),
		dialing:              cmap.NewCMap(),
		reconnecting:         cmap.NewCMap(),
		metrics:              NopMetrics(),
		transport:            transport,
		persistentPeersAddrs: make([]*NetAddress, 0),
		unconditionalPeerIDs: make(map[NodeID]struct{}),
		filterTimeout:        defaultFilterTimeout,
		conns:                NewConnSet(),
	}

	// Ensure we have a completely undeterministic PRNG.
	sw.rng = rand.NewRand()

	sw.BaseService = *service.NewBaseService(nil, "P2P Switch", sw)

	for _, option := range options {
		option(sw)
	}

	return sw
}
```
## 启动Switch
启动switch，它会启动所有的reactors 和peers
```go
func (sw *Switch) OnStart() error {

	// FIXME: Temporary hack to pass channel descriptors to MConn transport,
	// since they are not available when it is constructed. This will be
	// fixed when we implement the new router abstraction.
	if t, ok := sw.transport.(*MConnTransport); ok {
		t.channelDescs = sw.chDescs
	}

	// 首先调用Reactor 启动所有的Reactor.
	for _, reactor := range sw.reactors {
		err := reactor.Start()
		if err != nil {
			return fmt.Errorf("failed to start %v: %w", reactor, err)
		}
	}

	// 开始接受 peers
	go sw.acceptRoutine()

	return nil
}
```
## acceptRoutine
```go
func (sw *Switch) acceptRoutine() {
	for {
		var peerNodeInfo NodeInfo
		// 接收一个新连接好的peer
		c, err := sw.transport.Accept()
		if err == nil {
			// 在使用peer之前，需要在连接上执行一次握手
			// 以前的MConn transport 使用Accept() 进行handshaing。
			// 它是这是异步的，避免了head-of-line-blocking。
			// 但是随着handshakes从transport中迁移出去。
			// 我们在这里同步进行handshakes。
			// 主要作用是获取节点的信息
			peerNodeInfo, _, err = sw.handshakePeer(c, "")
		}
		if err == nil {
			err = sw.filterConn(c.(*mConnConnection).conn)
		}
		if err != nil {
			if c != nil {
				_ = c.Close()
			}
			if err == io.EOF {
				err = ErrTransportClosed{}
			}
			switch err := err.(type) {
			case ErrRejected:
				// 避免连接自己
				if err.IsSelf() {
					// Remove the given address from the address book and add to our addresses
					// to avoid dialing in the future.
					addr := err.Addr()
					sw.addrBook.RemoveAddress(&addr)
					sw.addrBook.AddOurAddress(&addr)
				}

				sw.Logger.Info(
					"Inbound Peer rejected",
					"err", err,
					"numPeers", sw.peers.Size(),
				)

				continue
			// 过滤超时peer
			case ErrFilterTimeout:
				sw.Logger.Error(
					"Peer filter timed out",
					"err", err,
				)

				continue
			// 判断是否为已经关闭的Transport
			case ErrTransportClosed:
				sw.Logger.Error(
					"Stopped accept routine, as transport is closed",
					"numPeers", sw.peers.Size(),
				)
			default:
				sw.Logger.Error(
					"Accept on transport errored",
					"err", err,
					"numPeers", sw.peers.Size(),
				)
				// We could instead have a retry loop around the acceptRoutine,
				// but that would need to stop and let the node shutdown eventually.
				// So might as well panic and let process managers restart the node.
				// There's no point in letting the node run without the acceptRoutine,
				// since it won't be able to accept new connections.
				panic(fmt.Errorf("accept routine exited: %v", err))
			}

			break
		}

		isPersistent := false
		addr, err := peerNodeInfo.NetAddress()
		if err == nil {
			isPersistent = sw.IsPeerPersistent(addr)
		}

		// 创建新的peer实例
		p := newPeer(
			peerNodeInfo,
			newPeerConn(false, isPersistent, c),
			sw.reactorsByCh,
			sw.StopPeerForError,
			PeerMetrics(sw.metrics),
		)

		if !sw.IsPeerUnconditional(p.NodeInfo().ID()) {
			// 如果我们已经有足够的peer数量，就忽略
			_, in, _ := sw.NumPeers()
			if in >= sw.config.MaxNumInboundPeers {
				sw.Logger.Info(
					"Ignoring inbound connection: already have enough inbound peers",
					"address", p.SocketAddr(),
					"have", in,
					"max", sw.config.MaxNumInboundPeers,
				)
				_ = p.CloseConn()
				continue
			}

		}

		// 把peer添加到switch中
		if err := sw.addPeer(p); err != nil {
			_ = p.CloseConn()
			if p.IsRunning() {
				_ = p.Stop()
			}
			sw.conns.RemoveAddr(p.RemoteAddr())
			sw.Logger.Info(
				"Ignoring inbound connection: error while adding peer",
				"err", err,
				"id", p.ID(),
			)
		}
	}
}
```
# peer
peer在p2p中表示一个对等体。 在tendermint中也是它和应用程序之间进行直接的消息交互。 peer实现了Peer这个接口的定义。
## Peer的结构
```go
type peer struct {
	service.BaseService

	// 原始的 peerConn 和 multiplex 连接
	// peerConn 的创建使用 newOutboundPeerConn 和 newInboundPeerConn。这两个函数是在switch组件中被调用的
	peerConn

	// peer's node info and the channel it knows about
	// channels = nodeInfo.Channels
	// cached to avoid copying nodeInfo in hasChannel
	nodeInfo    NodeInfo
	channels    []byte
	reactors    map[byte]Reactor
	onPeerError func(Peer, interface{})

	// 用户数据
	Data *cmap.CMap

	metrics       *Metrics
	metricsTicker *time.Ticker
}
```
## 创建Peer
创建peer实例，负责初始化一个peer实例。
```go
func newPeer(
	nodeInfo NodeInfo,
	pc peerConn,
	reactorsByCh map[byte]Reactor,
	onPeerError func(Peer, interface{}),
	options ...PeerOption,
) *peer {
	p := &peer{
		// 初始化peerConn实例
		peerConn:      pc,
		nodeInfo:      nodeInfo,
		channels:      nodeInfo.Channels, // TODO
		reactors:      reactorsByCh,
		onPeerError:   onPeerError,
		Data:          cmap.NewCMap(),
		metricsTicker: time.NewTicker(metricsTickerDuration),
		metrics:       NopMetrics(),
	}

	p.BaseService = *service.NewBaseService(nil, "Peer", p)
	for _, option := range options {
		option(p)
	}

	return p
}
```
## 启动Peer
```go
func (p *peer) OnStart() error {
	// 不需要调用 BaseService.OnStart()，所以直接返回了 nil
	if err := p.BaseService.OnStart(); err != nil {
		return err
	}

	// 处理从connection接收到的消息。
	go p.processMessages()
	// 心跳检测
	// 每隔10s，获取链接状态，并且将发送数据的通道大小与peerID关联起来
	go p.metricsReporter()

	return nil
}
```
# MConnection
transport的功能实现最终会到p2p/conn/connection.go文件里的MConnection上，所以MConnection是P2P最底层的部分。消息的写入和读取都是通过此组件完成的。它维护了网络连接、进行底层的网络数据传输。

MConnection是一个多路连接，在单个TCP连接上支持多个独立的流，具有不同的服务质量保证。每个流被称为一个channel，每个channel都有一个全局的唯一字节ID，每个channel都有优先级。每个channel的id和优先级在初始连接时配置。

MConnection支持三种类型的packet：

- Ping
- Pong
- Msg

当我们在pingTimeout规定的时间内没有收到任何消息时，我们就会发出一个ping消息，当一个对等端收到ping消息并且没有别的附加消息时，对等端就会回应一个pong消息。如果我们在规定的时间内没有收到pong消息，我们就会断开连接。
## 创建MConnection
```go
func NewMConnectionWithConfig(
	conn net.Conn,
	chDescs []*ChannelDescriptor,
	onReceive receiveCbFunc,
	onError errorCbFunc,
	config MConnConfig,
) *MConnection {
	if config.PongTimeout >= config.PingInterval {
		panic("pongTimeout must be less than pingInterval (otherwise, next ping will reset pong timer)")
	}

	mconn := &MConnection{
		// TCP连接成功返回的对象
		conn:          conn,
		// net.Con封装成bufio的读写，可以方便用类似文件IO的形式来对TCP流进行读写操作。
		bufConnReader: bufio.NewReaderSize(conn, minReadBufferSize),
		bufConnWriter: bufio.NewWriterSize(conn, minWriteBufferSize),
		sendMonitor:   flow.New(0, 0),
		recvMonitor:   flow.New(0, 0),
		send:          make(chan struct{}, 1),
		pong:          make(chan struct{}, 1),
		onReceive:     onReceive,
		onError:       onError,
		config:        config,
		created:       time.Now(),
	}

	// 创建通道
	var channelsIdx = map[byte]*Channel{}
	var channels = []*Channel{}

	for _, desc := range chDescs {
		channel := newChannel(mconn, *desc)
		channelsIdx[channel.desc.ID] = channel
		channels = append(channels, channel)
	}
	mconn.channels = channels
	mconn.channelsIdx = channelsIdx

	mconn.BaseService = *service.NewBaseService(nil, "MConnection", mconn)

	// maxPacketMsgSize() is a bit heavy, so call just once
	mconn._maxPacketMsgSize = mconn.maxPacketMsgSize()

	return mconn
}
```
## channel
```go
type Channel struct {
	conn          *MConnection
	desc          ChannelDescriptor
	sendQueue     chan []byte	// 发送队列
	sendQueueSize int32 // atomic.
	recving       []byte	// 接受缓存队列
	sending       []byte	// 发送缓冲区
	recentlySent  int64 // exponential moving average

	maxPacketMsgPayloadSize int

	Logger log.Logger
}
```
Peer调用Send发送消息其实是调用MConnecttion的Send方法，那么MConnecttion的Send其实也只是把内容发送到Channel的sendQueue中, 然后会有专门的routine读取Channel进行实际的消息发送。

## 启动MConnection
```go
func (c *MConnection) OnStart() error {
	if err := c.BaseService.OnStart(); err != nil {
		return err
	}

	// 同步周期
	c.flushTimer = timer.NewThrottleTimer("flush", c.config.FlushThrottle)
	// ping周期
	c.pingTimer = time.NewTicker(c.config.PingInterval)
	c.pongTimeoutCh = make(chan bool, 1)
	c.chStatsTimer = time.NewTicker(updateStats)
	c.quitSendRoutine = make(chan struct{})
	c.doneSendRoutine = make(chan struct{})
	c.quitRecvRoutine = make(chan struct{})
	// 发送任务循环
	go c.sendRoutine()
	// 接收任务循环
	go c.recvRoutine()
	return nil
}
```

## sendRoutine
```go
func (c *MConnection) sendRoutine() {
	defer c._recover()

	protoWriter := protoio.NewDelimitedWriter(c.bufConnWriter)

FOR_LOOP:
	for {
		var _n int
		var err error
	SELECTION:
		select {
		// 进行周期性的flush
		case <-c.flushTimer.Ch:
			// NOTE: flushTimer.Set() must be called every time
			// something is written to .bufConnWriter.
			c.flush()
		case <-c.chStatsTimer.C:
			for _, channel := range c.channels {
				channel.updateStats()
			}
		// 进行周期性的向TCP连接写入ping消息
		case <-c.pingTimer.C:
			c.Logger.Debug("Send Ping")
			_n, err = protoWriter.WriteMsg(mustWrapPacket(&tmp2p.PacketPing{}))
			if err != nil {
				c.Logger.Error("Failed to send PacketPing", "err", err)
				break SELECTION
			}
			c.sendMonitor.Update(_n)
			c.Logger.Debug("Starting pong timer", "dur", c.config.PongTimeout)
			c.pongTimer = time.AfterFunc(c.config.PongTimeout, func() {
				select {
				case c.pongTimeoutCh <- true:
				default:
				}
			})
			c.flush()
		case timeout := <-c.pongTimeoutCh:
			if timeout {
				c.Logger.Debug("Pong timeout")
				err = errors.New("pong timeout")
			} else {
				c.stopPongTimer()
			}
		// c.pong 表示需要进行pong回复，这个不是周期性的写入，因为收到了对方发来的ping消息，这个通道的写入是在recvRoutine函数中进行的
		case <-c.pong:
			c.Logger.Debug("Send Pong")
			_n, err = protoWriter.WriteMsg(mustWrapPacket(&tmp2p.PacketPong{}))
			if err != nil {
				c.Logger.Error("Failed to send PacketPong", "err", err)
				break SELECTION
			}
			c.sendMonitor.Update(_n)
			c.flush()
		case <-c.quitSendRoutine:
			break FOR_LOOP
		// 当c.send有写入，我们就应该信息包发送了
		case <-c.send:
			// Send some PacketMsgs
			// 发送一些包
			eof := c.sendSomePacketMsgs()
			if !eof {
				// Keep sendRoutine awake.
				select {
				case c.send <- struct{}{}:
				default:
				}
			}
		}

		if !c.IsRunning() {
			break FOR_LOOP
		}
		if err != nil {
			c.Logger.Error("Connection failed @ sendRoutine", "conn", c, "err", err)
			c.stopForError(err)
			break FOR_LOOP
		}
	}

	// Cleanup
	c.stopPongTimer()
	close(c.doneSendRoutine)
}
```
## sendPacketMsg
如果通道中的message被发送完, 返回true
```go
func (c *MConnection) sendPacketMsg() bool {

	// 选择的通道将是最近发送/优先级最小的通道。

	var leastRatio float32 = math.MaxFloat32
	var leastChannel *Channel
	for _, channel := range c.channels {
	
		// 检查channel.sendQueue 是否为0，channel.sending缓存区是否为空，如果空着说明没有需要发送的内容
		// 如果缓冲区为空了 就要把channel.sendQueue内部排队的内容移出一份到缓冲区
		// 如果有任何PacketMsgs等待发送，则返回true。
		if !channel.isSendPending() {
			continue
		}
		// Get ratio, and keep track of lowest ratio.
		ratio := float32(channel.recentlySent) / float32(channel.desc.Priority)
		if ratio < leastRatio {
			leastRatio = ratio
			leastChannel = channel
		}
	}

	// Nothing to send?
	if leastChannel == nil {
		return true
	}
	// c.Logger.Info("Found a msgPacket to send")

	// Make & send a PacketMsg from this channel
	// 执行到这里说明某个channel内部有消息没法送出去，将消息发送出去
	_n, err := leastChannel.writePacketMsgTo(c.bufConnWriter)
	if err != nil {
		c.Logger.Error("Failed to write PacketMsg", "err", err)
		c.stopForError(err)
		return true
	}
	c.sendMonitor.Update(_n)
	c.flushTimer.Set()
	return false
}
```
## recvRoutine
```go
func (c *MConnection) recvRoutine() {
	defer c._recover()

	protoReader := protoio.NewDelimitedReader(c.bufConnReader, c._maxPacketMsgSize)

FOR_LOOP:
	for {
		// Block until .recvMonitor says we can read.
		c.recvMonitor.Limit(c._maxPacketMsgSize, atomic.LoadInt64(&c.config.RecvRate), true)

		// Read packet type
		var packet tmp2p.Packet

		_n, err := protoReader.ReadMsg(&packet)
		c.recvMonitor.Update(_n)
		if err != nil {
			// stopServices was invoked and we are shutting down
			// receiving is excpected to fail since we will close the connection
			select {
			case <-c.quitRecvRoutine:
				break FOR_LOOP
			default:
			}

			if c.IsRunning() {
				if err == io.EOF {
					c.Logger.Info("Connection is closed @ recvRoutine (likely by the other side)", "conn", c)
				} else {
					c.Logger.Debug("Connection failed @ recvRoutine (reading byte)", "conn", c, "err", err)
				}
				c.stopForError(err)
			}
			break FOR_LOOP
		}

		// 根据解析出来的报文类型做相应的操作
		switch pkt := packet.Sum.(type) {
		case *tmp2p.Packet_PacketPing:
			// TODO: prevent abuse, as they cause flush()'s.
			// https://github.com/tendermint/tendermint/issues/1190
			c.Logger.Debug("Receive Ping")
			// 对法要求我们发送一个pong，所以职置位pong标识，在sendRoutine中做响应操作
			select {
			case c.pong <- struct{}{}:
			default:
				// never block
			}
		case *tmp2p.Packet_PacketPong:
			c.Logger.Debug("Receive Pong")
			// 更新pong超时状态
			select {
			case c.pongTimeoutCh <- false:
			default:
				// never block
			}
		case *tmp2p.Packet_PacketMsg:
			channel, ok := c.channelsIdx[byte(pkt.PacketMsg.ChannelID)]
			if !ok || channel == nil {
				err := fmt.Errorf("unknown channel %X", pkt.PacketMsg.ChannelID)
				c.Logger.Debug("Connection failed @ recvRoutine", "conn", c, "err", err)
				c.stopForError(err)
				break FOR_LOOP
			}

			msgBytes, err := channel.recvPacketMsg(*pkt.PacketMsg)
			// 根据接收到的报文，选择对应的channel，放入到对应的接收缓存区中。缓存区的作用是什么呢
			//  在上文的发送报文中我们发现一个PackageMsg包中可能并没有包含完整的内容，只有EOF为1才标识发送完成。
			//  所以下面这个函数其实就是先将接收到的内容放入到缓存区，只有所有内容收到以后才能组装成一个完整的内容
			if err != nil {
				if c.IsRunning() {
					c.Logger.Debug("Connection failed @ recvRoutine", "conn", c, "err", err)
					c.stopForError(err)
				}
				break FOR_LOOP
			}
			if msgBytes != nil {
				c.Logger.Debug("Received bytes", "chID", pkt.PacketMsg.ChannelID, "msgBytes", msgBytes)
			
				// 注意这个函数的调用非常重要，记得之前我说为啥只有Send没有Receive呢， 答案就在此处。
				// 也就是说MConnecttion会把接收到的完整消息通过回调的形式返回给上面。 这个onReceive回调和Reactor的OnReceive是啥关系呢
				// 以及这个ChannelID和Reactor又是啥关系呢 不着急， 后面我们慢慢分析。 反正可以确定的是MConnecttion通过这个回调函数把接收到的消息
				// 返回给你应用层。
				c.onReceive(byte(pkt.PacketMsg.ChannelID), msgBytes)
			}
		default:
			err := fmt.Errorf("unknown message type %v", reflect.TypeOf(packet))
			c.Logger.Error("Connection failed @ recvRoutine", "conn", c, "err", err)
			c.stopForError(err)
			break FOR_LOOP
		}
	}

	// Cleanup
	close(c.pong)
	for range c.pong {
		// Drain
	}
}
```

# PEX
上面的分析是先实现了如何向对等体发送消息，而PEX 实现了对节点的发现的功能。
```go
p2p/pex/pex_reactor.go
func NewReactor(b AddrBook, config *ReactorConfig) *Reactor {
	r := &Reactor{
		book:                 b,
		config:               config,
		ensurePeersPeriod:    defaultEnsurePeersPeriod,
		requestsSent:         cmap.NewCMap(),
		lastReceivedRequests: cmap.NewCMap(),
		crawlPeerInfos:       make(map[p2p.NodeID]crawlPeerInfo),
	}
	r.BaseReactor = *p2p.NewBaseReactor("PEX", r)
	return r
}
```
## 启动PEX
```go
func (r *Reactor) OnStart() error {
	// 启动reactor
	err := r.book.Start()
	if err != nil && err != service.ErrAlreadyStarted {
		return err
	}

	// 检查配置的种子节点格式是否正确，其格式为：<ID>@<IP>:<PORT>
	numOnline, seedAddrs, err := r.checkSeeds()
	if err != nil {
		return err
	} else if numOnline == 0 && r.book.Empty() {
		return errors.New("address book is empty and couldn't resolve any seed nodes")
	}
    
	r.seedAddrs = seedAddrs

	// 检查节点以什么方式启动，seed或者crawler
	if r.config.SeedMode {
		go r.crawlPeersRoutine()
	} else {
		go r.ensurePeersRoutine()
	}
	return nil
}
```
可以看出，节点发现有两种模式，一种是seed模式，一种是crawler模式。

种子模式
```go
func (r *Reactor) crawlPeersRoutine() {
	// 如果我们有任何种子节点，拨号一次
	if len(r.seedAddrs) > 0 {
		r.dialSeeds()
	} else {
		// 做一个最初的查找节点
		r.crawlPeers(r.book.GetSelection())
	}

	ticker := time.NewTicker(crawlPeerPeriod)

	for {
		select {
		case <-ticker.C:
			// 查看每一个peer和自己连接时长，如果超过规定的时间就断开
			r.attemptDisconnects()
			// 在指定范围内随机选取地址，然后进行连接
			r.crawlPeers(r.book.GetSelection())
			r.cleanupCrawlPeerInfos()
		case <-r.Quit():
			return
		}
	}
}


func (r *Reactor) crawlPeers(addrs []*p2p.NetAddress) {
	now := time.Now()
	
 	// addrs 是由GetSelection选出的特定地址，必须满足peer-exchange协议 
	for _, addr := range addrs {
		// peer的信息
		peerInfo, ok := r.crawlPeerInfos[addr.ID]

		// 如果上次尝试连接的时间和此处相差不到2分钟，则不进行连接
		if ok && now.Sub(peerInfo.LastCrawled) < minTimeBetweenCrawls {
			continue
		}

		// 更新最后一次尝试连接的时间和次数
		r.crawlPeerInfos[addr.ID] = crawlPeerInfo{
			Addr:        addr,
			LastCrawled: now,
		}

		// 尝试和这个地址进行一次连接
		err := r.dialPeer(addr)
		if err != nil {
			switch err.(type) {
			case errMaxAttemptsToDial, errTooEarlyToDial, p2p.ErrCurrentlyDialingOrExistingAddress:
				r.Logger.Debug(err.Error(), "addr", addr)
			default:
				r.Logger.Error(err.Error(), "addr", addr)
			}
			continue
		}

		peer := r.Switch.Peers().Get(addr.ID)
		if peer != nil {
			// 如果连接成功了，就向这个地址发送一个报文，这个报文的目的就是请求此peer知道的所有peer地址
			r.RequestAddrs(peer)
		}
	}
}
```
crawler模式
```go
// 确保有足够的peer正在连接
func (r *Reactor) ensurePeersRoutine() {
	var (
		seed   = tmrand.NewRand()
		jitter = seed.Int63n(r.ensurePeersPeriod.Nanoseconds())
	)

    // Randomize first round of communication to avoid thundering herd.
	// If no peers are present directly start connecting so we guarantee swift
	// setup with the help of configured seeds.
	if r.nodeHasSomePeersOrDialingAny() {
		time.Sleep(time.Duration(jitter))
	}

	// 如果地址簿是空的，那么立马对seed进行拨号，确保有足够的节点连接
	r.ensurePeers()

	// fire periodically
	ticker := time.NewTicker(r.ensurePeersPeriod)
	for {
		select {
		case <-ticker.C:
			r.ensurePeers()
		case <-r.Quit():
			ticker.Stop()
			return
		}
	}
}


func (r *Reactor) ensurePeers() {
	// 获取当前正在连接的peer信息
	var (
		out, in, dial = r.Switch.NumPeers()
		numToDial     = r.Switch.MaxNumOutboundPeers() - (out + dial)
	)
	r.Logger.Info(
		"Ensure peers",
		"numOutPeers", out,
		"numInPeers", in,
		"numDialing", dial,
		"numToDial", numToDial,
	)

	if numToDial <= 0 {
		return
	}

	// bias to prefer more vetted peers when we have fewer connections.
	// not perfect, but somewhate ensures that we prioritize connecting to more-vetted
	// NOTE: range here is [10, 90]. Too high ?
	newBias := tmmath.MinInt(out, 8)*10 + 10

	toDial := make(map[p2p.NodeID]*p2p.NetAddress)
	// Try maxAttempts times to pick numToDial addresses to dial
	maxAttempts := numToDial * 3

	for i := 0; i < maxAttempts && len(toDial) < numToDial; i++ {
		// 根据 newBias 随机的从bucket中挑选地址
		try := r.book.PickAddress(newBias)
		if try == nil {
			continue
		}
		if _, selected := toDial[try.ID]; selected {
			continue
		}
		if r.Switch.IsDialingOrExistingAddress(try) {
			continue
		}
		// TODO: consider moving some checks from toDial into here
		// so we don't even consider dialing peers that we want to wait
		// before dialing again, or have dialed too many times already
		r.Logger.Info("Will dial address", "addr", try)
		toDial[try.ID] = try
	}

	// 对挑选的地址进行拨号
	for _, addr := range toDial {
		go func(addr *p2p.NetAddress) {
			err := r.dialPeer(addr)
			if err != nil {
				switch err.(type) {
				case errMaxAttemptsToDial, errTooEarlyToDial:
					r.Logger.Debug(err.Error(), "addr", addr)
				default:
					r.Logger.Error(err.Error(), "addr", addr)
				}
			}
		}(addr)
	}

	if r.book.NeedMoreAddrs() {
		// 检查被禁止的节点是否可以恢复
		r.book.ReinstateBadPeers()
	}

	if r.book.NeedMoreAddrs() {
		// 1) Pick a random peer and ask for more.
		peers := r.Switch.Peers().List()
		peersCount := len(peers)
		if peersCount > 0 {
			peer := peers[tmrand.Int()%peersCount]
			r.Logger.Info("We need more addresses. Sending pexRequest to random peer", "peer", peer)
			r.RequestAddrs(peer)
		}

		// 2) Dial seeds if we are not dialing anyone.
		// This is done in addition to asking a peer for addresses to work-around
		// peers not participating in PEX.
		if len(toDial) == 0 {
			r.Logger.Info("No addresses to dial. Falling back to seeds")
			// 如果没有对任何地址拨号就对种子节点进行拨号
             r.dialSeeds()
		}
	}
}

func (r *Reactor) dialPeer(addr *p2p.NetAddress) error {
	attempts, lastDialed := r.dialAttemptsInfo(addr)
	// 如果是禁止的节点或者尝试次数过多，就标记为坏掉的节点
    if !r.Switch.IsPeerPersistent(addr) && attempts > maxAttemptsToDial {
		r.book.MarkBad(addr, defaultBanTime)
		return errMaxAttemptsToDial{}
	}

	// exponential backoff if it's not our first attempt to dial given address
	if attempts > 0 {
		jitter := time.Duration(tmrand.Float64() * float64(time.Second)) // 1s == (1e9 ns)
		backoffDuration := jitter + ((1 << uint(attempts)) * time.Second)
		backoffDuration = r.maxBackoffDurationForPeer(addr, backoffDuration)
		sinceLastDialed := time.Since(lastDialed)
		if sinceLastDialed < backoffDuration {
			return errTooEarlyToDial{backoffDuration, lastDialed}
		}
	}

	// 尝试和这个地址进行一次连接
	err := r.Switch.DialPeerWithAddress(addr)
	if err != nil {
		if _, ok := err.(p2p.ErrCurrentlyDialingOrExistingAddress); ok {
			return err
		}

		markAddrInBookBasedOnErr(addr, r.book, err)
		switch err.(type) {
		case p2p.ErrSwitchAuthenticationFailure:
			// NOTE: addr is removed from addrbook in markAddrInBookBasedOnErr
			r.attemptsToDial.Delete(addr.DialString())
		default:
			r.attemptsToDial.Store(addr.DialString(), _attemptsToDial{attempts + 1, time.Now()})
		}
		return fmt.Errorf("dialing failed (attempts: %d): %w", attempts+1, err)
	}

	// cleanup any history
	r.attemptsToDial.Delete(addr.DialString())
	return nil
}

func (sw *Switch) DialPeerWithAddress(addr *NetAddress) error {
	// 判断节点是否合法
	if sw.IsDialingOrExistingAddress(addr) {
		return ErrCurrentlyDialingOrExistingAddress{addr.String()}
	}

	sw.dialing.Set(string(addr.ID), addr)
	defer sw.dialing.Delete(string(addr.ID))

    // 添加参数中的节点到outbound
	return sw.addOutboundPeerWithConfig(addr, sw.config)
}
```
# 最后
至此，Tendermint P2P源码就分析完了。如果有错误之处，多多指教哦。

最后推荐一位大佬的公众号，欢迎关注哦：**区块链技术栈**

参考文章：https://gitee.com/wupeaking/tendermint_code_analysis/blob/master/p2p%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md

