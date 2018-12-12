# Tendermint 代码笔记 (四) node

[TOC]
node命令实现了Tendermint core的内容，正如之前讨论过的，包括两个部分，命令和服务，命令指的是node这个命令本身，服务指在node中提供的服务。我们分别讨论。

## 命令

就node命令本身而言，其定义在github.com/tendermint/tendermint/cmd/tendermint/commands/run_node.go中，里面的内容在《代码笔记 (一)》中已经讨论过，就不再重复了。

## 服务

### Node定义

```go
// Node is the highest level interface to a full Tendermint node.
// It includes all configuration information and running services.                                                                                                                               
type Node struct {            
    cmn.BaseService           
                              
    // config 
    config        *cfg.Config 
    genesisDoc    *types.GenesisDoc   // initial validator set
    privValidator types.PrivValidator // local node's validator key                                                                                                                              
                              
    // network 
    transport   *p2p.MultiplexTransport        
    sw          *p2p.Switch  // p2p connections
    addrBook    pex.AddrBook // known peers    
    nodeInfo    p2p.NodeInfo
    nodeKey     *p2p.NodeKey // our node privkey
    isListening bool          

    // services
    eventBus         *types.EventBus // pub/sub for services                                                                                                                                     
    stateDB          dbm.DB
    blockStore       *bc.BlockStore         // store the blockchain to disk
    bcReactor        *bc.BlockchainReactor  // for fast-syncing
    mempoolReactor   *mempl.MempoolReactor  // for gossipping transactions 
    consensusState   *cs.ConsensusState     // latest consensus state
    consensusReactor *cs.ConsensusReactor   // for participating in the consensus
    evidencePool     *evidence.EvidencePool // tracking evidence
    proxyApp         proxy.AppConns         // connection to the application
    rpcListeners     []net.Listener         // rpc servers
    txIndexer        txindex.TxIndexer
    indexerService   *txindex.IndexerService
    prometheusSrv    *http.Server                             }  
```

显然这是个服务的定义，其中的许多字段本身也是服务，我们会逐个研究。

### NewNode()函数

当然第一步是创建一个node。在*DefaultNewNode(config \*cfg.Config, logger log.Logger)*函数中，是这样调用*NewNode()*的：

```go
NewNode(config,    
     privval.LoadOrGenFilePV(config.PrivValidatorFile()),                                                                                                                                     
     nodeKey, 
     proxy.DefaultClientCreator(config.ProxyApp, config.ABCI, config.DBDir()),                                                                                                                
     DefaultGenesisDocProviderFunc(config),
     DefaultDBProvider,
     DefaultMetricsProvider(config.Instrumentation),                                                                                                                             
     logger,
 )           
```

注意logger参数后面那个逗号，go中是允许的，这并不是个错误。

其对应的函数签名是

```go
// NewNode returns a new, ready to go, Tendermint Node.
func NewNode(config *cfg.Config,
	privValidator types.PrivValidator,
	nodeKey *p2p.NodeKey,
	clientCreator proxy.ClientCreator,
	genesisDocProvider GenesisDocProvider,
	dbProvider DBProvider,
	metricsProvider MetricsProvider,
	logger log.Logger) (*Node, error) 
```

在接下来的讨论中我们会说明这些参数是什么。

#### 数据库

除去两条打印语句，接下来马上就是

```go
	// Get BlockStore
	blockStoreDB, err := dbProvider(&DBContext{"blockstore", config})
	if err != nil {
		return nil, err
	}
```
显然这里创建了一个数据库，这里*dbProvider*类型是*DBProvider*，一个函数变量，被赋值为*DefaultDBProvider*。其定义都在*node.go*文件中。
```go
// DBProvider takes a DBContext and returns an instantiated DB.
type DBProvider func(*DBContext) (dbm.DB, error)
// DefaultDBProvider returns a database using the DBBackend and DBDir
// specified in the ctx.Config.
func DefaultDBProvider(ctx *DBContext) (dbm.DB, error) {
    dbType := dbm.DBBackendType(ctx.Config.DBBackend)
    return dbm.NewDB(ctx.ID, dbType, ctx.Config.DBDir()), nil
}  
// DBContext specifies config information for loading a new DB.
type DBContext struct {
    ID     string
    Config *cfg.Config
} 
```
我们知道tendermint缺省使用的数据库是levelDB，这是缺省配置里边写好的。

dbm是*github.com/tendermint/tendermint/libs/db*的别名。

注意*DBBackendType(ctx.Config.DBBackend)*，因为

```go
type DBBackendType string
```

这里自定义了一个类型，*DBBackendType(ctx.Config.DBBackend)*是类型转换，*ctx.Config.DBBackend*按定义是个string。注意跟C语言类型转换句法的区别。

于是我们在*ctx.Config.DBDir()*目录下建立了一个叫*ctx.ID*,也就是*blockstore*的levelDB数据库。

```go
func NewDB(name string, backend DBBackendType, dir string) DB {
    dbCreator, ok := backends[backend]
    if !ok {                  
        keys := make([]string, len(backends))                                                                                                                                                    
        i := 0
        for k := range backends {          
            keys[i] = string(k)            
            i++
        }
        panic(fmt.Sprintf("Unknown db_backend %s, expected either %s", backend, strings.Join(keys, " or ")))                                                                                     
    }

    db, err := dbCreator(name, dir)
    if err != nil {
        panic(fmt.Sprintf("Error initializing DB: %v", err))                                                                                                                                     
    }
    return db
}
```
*backends*是一个*map*：

```go
var backends = map[DBBackendType]dbCreator{}
```

*dbCreator*是个函数变量：
```go
type dbCreator func(name string, dir string) (DB, error)
```
而*DBBackendType*则是：
```go
const (
    LevelDBBackend   DBBackendType = "leveldb" // legacy, defaults to goleveldb unless +gcc
    CLevelDBBackend  DBBackendType = "cleveldb"
    GoLevelDBBackend DBBackendType = "goleveldb"
    MemDBBackend     DBBackendType = "memdb"
    FSDBBackend      DBBackendType = "fsdb" // using the filesystem naively 
) 
```
那么，*backends*是怎么初始化的呢？是通过在相应的数据库包的*init()*函数中调用*registerDBCreator*，我们知道这样的*init()*总是在*main()*之前执行。

```go
func registerDBCreator(backend DBBackendType, creator dbCreator, force bool) {                                                                                                                   
    _, ok := backends[backend]
    if !force && ok { 
        return
    }
    backends[backend] = creator
}     
```
比方说在*github.com/tendermint/tendermint/libs/db/go_level_db.go*中的*init()*函数

```go
func init() {                 
    dbCreator := func(name string, dir string) (DB, error) {
        return NewGoLevelDB(name, dir) 
    }
    registerDBCreator(LevelDBBackend, dbCreator, false)
    registerDBCreator(GoLevelDBBackend, dbCreator, false)
}  
```

#### BlockStore

接下来

```go
blockStore := bc.NewBlockStore(blockStoreDB)
```

创建了一个*BlockStore*

```go
type BlockStore struct {
    db dbm.DB                 
    mtx    sync.RWMutex       
    height int64
}                             

func NewBlockStore(db dbm.DB) *BlockStore {
    bsjson := LoadBlockStoreStateJSON(db) 
    return &BlockStore{       
        height: bsjson.Height,
        db:     db,
    }
}
```

总之，这是个存储区块的地方。为了不至于迷失在代码网中，就先到这。*NewNode()*的流程看完后，再去研究各个包的实现。

#### State

```go
	// Get State
	stateDB, err := dbProvider(&DBContext{"state", config})
	if err != nil {
		return nil, err
	}
```
因为tendermint是个状态机，必然有个初始状态，这个初始状态在哪里呢，在一个初始文件中，所以我们要打开它或者创建它，注意跟*stateDB*跟*genDoc*的联系。
```go
	// Get genesis doc
	// TODO: move to state package?
	genDoc, err := loadGenesisDoc(stateDB)
	if err != nil {
		genDoc, err = genesisDocProvider()
		if err != nil {
			return nil, err
		}
		// save genesis doc to prevent a certain class of user errors (e.g. when it
		// was changed, accidentally or not). Also good for audit trail.
		saveGenesisDoc(stateDB, genDoc)
	}
```
接下来是获取*state*，这个*state*不是上面说的那个数据的名字，那么是什么呢？我们同样知道tendermint维护一个状态机，这个*state*就是状态机的状态。
```go
	state, err := sm.LoadStateFromDBOrGenesisDoc(stateDB, genDoc)
	if err != nil {
		return nil, err
	}
```
#### proxyApp

我们知道，tendermint启动一个节点有两种方式，如果在启动*node*时带有*--proxy_app=kvstore*参数，这时候会启动内置的*kvstore*这个*ABCI APP*并与之建立三个连接(*consensus, mempool, query*)，而如果没有这个参数，那么就必须另外启动一个指定地址和端口的*ABCI APP*，所谓指定，指的是在配置文件里边设置好，再启动tendermint时，同样会建立同上的三个连接。

```go
	// Create the proxyApp and establish connections to the ABCI app (consensus, mempool, query).
	proxyApp := proxy.NewAppConns(clientCreator)
	proxyApp.SetLogger(logger.With("module", "proxy"))
	if err := proxyApp.Start(); err != nil {
		return nil, fmt.Errorf("Error starting proxy app connections: %v", err)
	}
```
*NewAppConns*要做什么呢？不过我们还是先看*clientCreator*，从*NewNode*的参数列表中我们可以看到这是一个*proxy.ClientCreator*

##### clientCreator
我们先回到*DefaultNewNode*，可以看到传入到*NewNode*的实际参数是*proxy.DefaultClientCreator(config.ProxyApp, config.ABCI, config.DBDir())*返回的这么一个*proxy.ClientCreator*,这三个参数分别代表*proxy*的地址，连接方式(socket或者grpc)及数据库目录，连接方式缺省是*socket*。我们看看这个函数做了什么：

```go
func DefaultClientCreator(addr, transport, dbDir string) ClientCreator {
    switch addr {
    case "kvstore":           
        fallthrough           
    case "dummy": 
        return NewLocalClientCreator(kvstore.NewKVStoreApplication())
    case "persistent_kvstore":
        fallthrough 
    case "persistent_dummy":  
        return NewLocalClientCreator(kvstore.NewPersistentKVStoreApplication(dbDir))
    case "nilapp":            
        return NewLocalClientCreator(types.NewBaseApplication())
    default:                  
        mustConnect := false // loop retrying
        return NewRemoteClientCreator(addr, transport, mustConnect)
    }
}  

```

在go中，数值，字符串都是合法的*switch*的条件，*case*也不需要*break*，所以需要*fallthrough*，这是与c/c++不同的地方。

我们先看*NewRemoteClientCreator*，*addr*可以是配置文件里边读取的，也就是缺省的*proxy_app = "tcp://127.0.0.1:26658”*，或者自定义的一个值，也可以是命令行参数，如果启动Tendermint时使用的是这样的命令行：

> tendermint node --proxy_app=kvstore

那么配置里的这个值就会被这个kvstore代替。另外，我们从这里可以看到tendermint启动node时proxy_app参数的可选值。
```go
func NewRemoteClientCreator(addr, transport string, mustConnect bool) ClientCreator {
    return &remoteClientCreator{   
        addr:        addr,    
        transport:   transport,        
        mustConnect: mustConnect,      
    }
}
```

那么问题来了，*remoteClientCreator*是什么呢？是一个*struct*，

```go
type remoteClientCreator struct {
    addr        string
    transport   string
    mustConnect bool
}
```
而
```go
type ClientCreator interface {
    NewABCIClient() (abcicli.Client, error)
}  
```

是一个接口。

如果一定要按C++的逻辑，那么这个remoteClientCreator就是抽象类ClientCreator的子类。不过在go以及python中这里就是所谓的鸭子类型。因为remoteClientCreator实现了ClientCreator的NewABCIClient()方法，于是它就是个ClientCreator。

```go
func (r *remoteClientCreator) NewABCIClient() (abcicli.Client, error) {
    remoteApp, err := abcicli.NewClient(r.addr, r.transport, r.mustConnect)
    if err != nil {
        return nil, errors.Wrap(err, "Failed to connect to proxy")
    }
    return remoteApp, nil
}
```

所以，虽然go并不是个面向对象的程序设计语言，但是用鸭子类型实现了多态。

知道clientCreator是什么之后，我们再来看*NewAppConns*。

##### NewAppConns

```go
func NewAppConns(clientCreator ClientCreator) AppConns {
    return NewMultiAppConn(clientCreator) 
}  

```

而返回的*AppConns*是

```go
type AppConns interface {     
    cmn.Service
   
    Mempool() AppConnMempool
    Consensus() AppConnConsensus   
    Query() AppConnQuery      
}  
```

那么

```go
func NewMultiAppConn(clientCreator ClientCreator) *multiAppConn {
    multiAppConn := &multiAppConn{
        clientCreator: clientCreator,
    }
    multiAppConn.BaseService = *cmn.NewBaseService(nil, "multiAppConn", multiAppConn)
    return multiAppConn
}
```

注意上面的cmn.NewBaseService(nil, "multiAppConn", multiAppConn)的第三个参数multiAppConn，如前面我们解说服务时候提到的，这是又把multiAppConn传给了其BaseService当中的Service成员。

并且

```go
type multiAppConn struct {    
    cmn.BaseService
   
    mempoolConn   *appConnMempool  
    consensusConn *appConnConsensus
    queryConn     *appConnQuery    
   
    clientCreator ClientCreator    
}
```

因此*multiAppConn*也是个鸭子类型，实现了接口*AppConns*，当然并不是说就凭上面这几行代码就能这么说，而是在于其实现了这几个函数

```go
Mempool() AppConnMempool
Consensus() AppConnConsensus   
Query() AppConnQuery 
```

以及在函数*NewMultiAppConn*中的两行赋值语句。

这里必须要提醒的是在实现接口的函数时，其接收者尽可能用指针的形式，正如在C++中的多态也总是由类指针来实现，虽然其中的原理表面上是不一样的，但是归根结底，总还是可以归结到C++中为什么要使用指针的原理上。

##### proxyApp.Start()

这里我们不打算深究这个函数调用，简单的说，就是在其中建立了与ABCI App的3个连接并启动了收发线程。

#### NewHandshaker

```go

	// Create the handshaker, which calls RequestInfo, sets the AppVersion on the state,
	// and replays any blocks as necessary to sync tendermint with the app.
	consensusLogger := logger.With("module", "consensus")
	handshaker := cs.NewHandshaker(stateDB, state, blockStore, genDoc)
	handshaker.SetLogger(consensusLogger)
	if err := handshaker.Handshake(proxyApp); err != nil {
		return nil, fmt.Errorf("Error during handshake: %v", err)
	}
```
####  LoadState
```go
	// Reload the state. It will have the Version.Consensus.App set by the
	// Handshake, and may have other modifications as well (ie. depending on
	// what happened during block replay).
	state = sm.LoadState(stateDB)

	// Ensure the state's block version matches that of the software.
	if state.Version.Consensus.Block != version.BlockProtocol {
		return nil, fmt.Errorf(
			"Block version of the software does not match that of the state.\n"+
				"Got version.BlockProtocol=%v, state.Version.Consensus.Block=%v",
			version.BlockProtocol,
			state.Version.Consensus.Block,
		)
	}
```
#### ValidatorSocketClient
```go
	if config.PrivValidatorListenAddr != "" {
		// If an address is provided, listen on the socket for a connection from an
		// external signing process.
		// FIXME: we should start services inside OnStart
		privValidator, err = createAndStartPrivValidatorSocketClient(config.PrivValidatorListenAddr, logger)
		if err != nil {
			return nil, errors.Wrap(err, "Error with private validator socket client")
		}
	}

	// Decide whether to fast-sync or not
	// We don't fast-sync when the only validator is us.
	fastSync := config.FastSync
	if state.Validators.Size() == 1 {
		addr, _ := state.Validators.GetByIndex(0)
		if bytes.Equal(privValidator.GetAddress(), addr) {
			fastSync = false
		}
	}

	// Log whether this node is a validator or an observer
	if state.Validators.HasAddress(privValidator.GetAddress()) {
		consensusLogger.Info("This node is a validator", "addr", privValidator.GetAddress(), "pubKey", privValidator.GetPubKey())
	} else {
		consensusLogger.Info("This node is not a validator", "addr", privValidator.GetAddress(), "pubKey", privValidator.GetPubKey())
	}
```
#### metrics
```go
	csMetrics, p2pMetrics, memplMetrics, smMetrics := metricsProvider()
```
#### mempool
```go
	// Make MempoolReactor
	mempool := mempl.NewMempool(
		config.Mempool,
		proxyApp.Mempool(),
		state.LastBlockHeight,
		mempl.WithMetrics(memplMetrics),
		mempl.WithPreCheck(sm.TxPreCheck(state)),
		mempl.WithPostCheck(sm.TxPostCheck(state)),
	)
	mempoolLogger := logger.With("module", "mempool")
	mempool.SetLogger(mempoolLogger)
	if config.Mempool.WalEnabled() {
		mempool.InitWAL() // no need to have the mempool wal during tests
	}
	mempoolReactor := mempl.NewMempoolReactor(config.Mempool, mempool)
	mempoolReactor.SetLogger(mempoolLogger)

	if config.Consensus.WaitForTxs() {
		mempool.EnableTxsAvailable()
	}
```
#### evidence
```go
	// Make Evidence Reactor
	evidenceDB, err := dbProvider(&DBContext{"evidence", config})
	if err != nil {
		return nil, err
	}
	evidenceLogger := logger.With("module", "evidence")
	evidenceStore := evidence.NewEvidenceStore(evidenceDB)
	evidencePool := evidence.NewEvidencePool(stateDB, evidenceStore)
	evidencePool.SetLogger(evidenceLogger)
	evidenceReactor := evidence.NewEvidenceReactor(evidencePool)
	evidenceReactor.SetLogger(evidenceLogger)
```
#### blockchain
```go
	blockExecLogger := logger.With("module", "state")
	// make block executor for consensus and blockchain reactors to execute blocks
	blockExec := sm.NewBlockExecutor(
		stateDB,
		blockExecLogger,
		proxyApp.Consensus(),
		mempool,
		evidencePool,
		sm.BlockExecutorWithMetrics(smMetrics),
	)

	// Make BlockchainReactor
	bcReactor := bc.NewBlockchainReactor(state.Copy(), blockExec, blockStore, fastSync)
	bcReactor.SetLogger(logger.With("module", "blockchain"))
```
####  Consensus
```go
	// Make ConsensusReactor
	consensusState := cs.NewConsensusState(
		config.Consensus,
		state.Copy(),
		blockExec,
		blockStore,
		mempool,
		evidencePool,
		cs.StateMetrics(csMetrics),
	)
	consensusState.SetLogger(consensusLogger)
	if privValidator != nil {
		consensusState.SetPrivValidator(privValidator)
	}
	consensusReactor := cs.NewConsensusReactor(consensusState, fastSync, cs.ReactorMetrics(csMetrics))
	consensusReactor.SetLogger(consensusLogger)

	eventBus := types.NewEventBus()
	eventBus.SetLogger(logger.With("module", "events"))

	// services which will be publishing and/or subscribing for messages (events)
	// consensusReactor will set it on consensusState and blockExecutor
	consensusReactor.SetEventBus(eventBus)
```
#### Transaction indexing
```go
	// Transaction indexing
	var txIndexer txindex.TxIndexer
	switch config.TxIndex.Indexer {
	case "kv":
		store, err := dbProvider(&DBContext{"tx_index", config})
		if err != nil {
			return nil, err
		}
		if config.TxIndex.IndexTags != "" {
			txIndexer = kv.NewTxIndex(store, kv.IndexTags(splitAndTrimEmpty(config.TxIndex.IndexTags, ",", " ")))
		} else if config.TxIndex.IndexAllTags {
			txIndexer = kv.NewTxIndex(store, kv.IndexAllTags())
		} else {
			txIndexer = kv.NewTxIndex(store)
		}
	default:
		txIndexer = &null.TxIndex{}
	}

	indexerService := txindex.NewIndexerService(txIndexer, eventBus)
	indexerService.SetLogger(logger.With("module", "txindex"))
```
#### p2p
```go
	var (
		p2pLogger = logger.With("module", "p2p")
		nodeInfo  = makeNodeInfo(
			config,
			nodeKey.ID(),
			txIndexer,
			genDoc.ChainID,
			p2p.NewProtocolVersion(
				version.P2PProtocol, // global
				state.Version.Consensus.Block,
				state.Version.Consensus.App,
			),
		)
	)

	// Setup Transport.
	var (
		mConnConfig = p2p.MConnConfig(config.P2P)
		transport   = p2p.NewMultiplexTransport(nodeInfo, *nodeKey, mConnConfig)
		connFilters = []p2p.ConnFilterFunc{}
		peerFilters = []p2p.PeerFilterFunc{}
	)

	if !config.P2P.AllowDuplicateIP {
		connFilters = append(connFilters, p2p.ConnDuplicateIPFilter())
	}

	// Filter peers by addr or pubkey with an ABCI query.
	// If the query return code is OK, add peer.
	if config.FilterPeers {
		connFilters = append(
			connFilters,
			// ABCI query for address filtering.
			func(_ p2p.ConnSet, c net.Conn, _ []net.IP) error {
				res, err := proxyApp.Query().QuerySync(abci.RequestQuery{
					Path: fmt.Sprintf("/p2p/filter/addr/%s", c.RemoteAddr().String()),
				})
				if err != nil {
					return err
				}
				if res.IsErr() {
					return fmt.Errorf("Error querying abci app: %v", res)
				}

				return nil
			},
		)

		peerFilters = append(
			peerFilters,
			// ABCI query for ID filtering.
			func(_ p2p.IPeerSet, p p2p.Peer) error {
				res, err := proxyApp.Query().QuerySync(abci.RequestQuery{
					Path: fmt.Sprintf("/p2p/filter/id/%s", p.ID()),
				})
				if err != nil {
					return err
				}
				if res.IsErr() {
					return fmt.Errorf("Error querying abci app: %v", res)
				}

				return nil
			},
		)
	}

	p2p.MultiplexTransportConnFilters(connFilters...)(transport)
```
#### switch
```go
	// Setup Switch.
	sw := p2p.NewSwitch(
		config.P2P,
		transport,
		p2p.WithMetrics(p2pMetrics),
		p2p.SwitchPeerFilters(peerFilters...),
	)
	sw.SetLogger(p2pLogger)
	sw.AddReactor("MEMPOOL", mempoolReactor)
	sw.AddReactor("BLOCKCHAIN", bcReactor)
	sw.AddReactor("CONSENSUS", consensusReactor)
	sw.AddReactor("EVIDENCE", evidenceReactor)
	sw.SetNodeInfo(nodeInfo)
	sw.SetNodeKey(nodeKey)

	p2pLogger.Info("P2P Node ID", "ID", nodeKey.ID(), "file", config.NodeKeyFile())

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
	addrBook := pex.NewAddrBook(config.P2P.AddrBookFile(), config.P2P.AddrBookStrict)

	// Add ourselves to addrbook to prevent dialing ourselves
	addrBook.AddOurAddress(nodeInfo.NetAddress())

	addrBook.SetLogger(p2pLogger.With("book", config.P2P.AddrBookFile()))
	if config.P2P.PexReactor {
		// TODO persistent peers ? so we can have their DNS addrs saved
		pexReactor := pex.NewPEXReactor(addrBook,
			&pex.PEXReactorConfig{
				Seeds:    splitAndTrimEmpty(config.P2P.Seeds, ",", " "),
				SeedMode: config.P2P.SeedMode,
			})
		pexReactor.SetLogger(p2pLogger)
		sw.AddReactor("PEX", pexReactor)
	}

	sw.SetAddrBook(addrBook)

	// run the profile server
	profileHost := config.ProfListenAddress
	if profileHost != "" {
		go func() {
			logger.Error("Profile server", "err", http.ListenAndServe(profileHost, nil))
		}()
	}
```
#### return
```go
	node := &Node{
		config:        config,
		genesisDoc:    genDoc,
		privValidator: privValidator,

		transport: transport,
		sw:        sw,
		addrBook:  addrBook,
		nodeInfo:  nodeInfo,
		nodeKey:   nodeKey,

		stateDB:          stateDB,
		blockStore:       blockStore,
		bcReactor:        bcReactor,
		mempoolReactor:   mempoolReactor,
		consensusState:   consensusState,
		consensusReactor: consensusReactor,
		evidencePool:     evidencePool,
		proxyApp:         proxyApp,
		txIndexer:        txIndexer,
		indexerService:   indexerService,
		eventBus:         eventBus,
	}
	node.BaseService = *cmn.NewBaseService(logger, "Node", node)
	return node, nil
}
```
节点有了，接下来看看启动的时候会怎么样

### OnStart()

