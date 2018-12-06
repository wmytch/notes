# Tendermint 代码笔记 主流程

[TOC]
## main.main()

~~~go
package main

import (
	"os"
	"path/filepath"

	"github.com/tendermint/tendermint/libs/cli"

	cmd "github.com/tendermint/tendermint/cmd/tendermint/commands"
	cfg "github.com/tendermint/tendermint/config"
	nm "github.com/tendermint/tendermint/node"
)
~~~
上面这一段就不说了，一看就明白，不过要注意的是在这些导入的包中，如果有*init()*函数，那么会在*main()*函数之前执行。
在研究main函数之前，我们需要看看文档里边运行tendermint的命令：

```bash
tendermint init
tendermint testnet --help
tendermint node
tendermint node --proxy_app=kvstore
```
从c/c++的角度来说，本能的会这样来实现：

```c
if (strcmp("init",argv[1]) == 0){  //C++的话通常是argv[1]=="init"
    return init(argc,argv);
} else if(strcmp("testnet",argv[1])==0){
    return testnet(argc,argv);
}else if(strcmp("node",argv[1])) == 0 ){
    return node(argc,argv);
}    
```

当然并不是说这样好或者不好，至少到目前为止，没有任何理由可以说这样处理不好。我们下面看在tendermint中怎么处理的：
```go
	rootCmd := cmd.RootCmd
	rootCmd.AddCommand(
		cmd.GenValidatorCmd,
		cmd.InitFilesCmd,
		cmd.ProbeUpnpCmd,
		cmd.LiteCmd,
		cmd.ReplayCmd,
		cmd.ReplayConsoleCmd,
		cmd.ResetAllCmd,
		cmd.ResetPrivValidatorCmd,
		cmd.ShowValidatorCmd,
		cmd.TestnetFilesCmd,
		cmd.ShowNodeIDCmd,
		cmd.GenNodeKeyCmd,
		cmd.VersionCmd)
```
*cmd.RootCmd*的定义是这样的：

```go
var RootCmd = &cobra.Command{ 
    Use:   "tendermint",      
    Short: "Tendermint Core (BFT Consensus) in Go",
    PersistentPreRunE: func(cmd *cobra.Command, args []string) (err error) {                                                                                                                     
        if cmd.Name() == VersionCmd.Name() {
            return nil
        }
        config, err = ParseConfig()    
        if err != nil {
            return err
        }
        logger, err = tmflags.ParseLogLevel(config.LogLevel, logger, cfg.DefaultLogLevel())                                                                                                      
        if err != nil {
            return err
        }
        if viper.GetBool(cli.TraceFlag) {  
            logger = log.NewTracingLogger(logger)                                                                                                                                                
        }
        logger = logger.With("module", "main")
        return nil
    },
}
```

显然这是个指针，*rootCmd*当然就是指向同一个对象的另一个指针的名字。实际上，我们可以从`rootCmd := cmd.RootCmd`和`var RootCmd = &cobra.Command`的区别体会下go语言中变量初始化的两种不同方式。

接下来用*AddCommand*注册了许多命令，其中就包括我们前面看到的*init*和*testnet*,当然，命令名和命令行参数并不一样，也没有这样的要求。必须要注意的是这里只是注册，并没有执行这些命令。

下面这段是又增加了一条命令，对应的是前面的*node*参数。

```go
	// NOTE:
	// Users wishing to:
	//	* Use an external signer for their validators
	//	* Supply an in-proc abci app
	//	* Supply a genesis doc file from another source
	//	* Provide their own DB implementation
	// can copy this file and use something other than the
	// DefaultNewNode function
	nodeFunc := nm.DefaultNewNode

	// Create & start node
	rootCmd.AddCommand(cmd.NewRunNodeCmd(nodeFunc))
```
那段Note的意思如果你想写自己的什么什么，就把这个文件拷贝过去，然后自己添加命令就好啦。  

我们继续看代码，*nm.DefaultNewNode*的定义是：

```go
func DefaultNewNode(config *cfg.Config, logger log.Logger) (*Node, error) {
    // Generate node PrivKey
    nodeKey, err := p2p.LoadOrGenNodeKey(config.NodeKeyFile())
    if err != nil {
        return nil, err
    }
    return NewNode(config,
        privval.LoadOrGenFilePV(config.PrivValidatorFile()),
        nodeKey,              
        proxy.DefaultClientCreator(config.ProxyApp, config.ABCI, config.DBDir()),
        DefaultGenesisDocProviderFunc(config),
        DefaultDBProvider,    
        DefaultMetricsProvider(config.Instrumentation),
        logger,
    )
}  
```

这是个函数的定义，`nodeFunc := nm.DefaultNewNode`也就意味着*nodeFunc*是个函数变量，从c/c++的角度来说首先要声明*nodeFunc*为一个函数指针然后再赋值，而在go里面就这样。

接下来我们看*NewRunNodeCmd*:

```go
func NewRunNodeCmd(nodeProvider nm.NodeProvider) *cobra.Command {
    cmd := &cobra.Command{    
        Use:   "node",        
        Short: "Run the tendermint node",
        RunE: func(cmd *cobra.Command, args []string) error {
            n, err := nodeProvider(config, logger)
            if err != nil {   
                return fmt.Errorf("Failed to create node: %v", err)
            }                 
    
            // Stop upon receiving SIGTERM or CTRL-C
            c := make(chan os.Signal, 1)   
            signal.Notify(c, os.Interrupt, syscall.SIGTERM) 
            go func() {
                for sig := range c {           
                    logger.Error(fmt.Sprintf("captured %v, exiting...", sig))
                    if n.IsRunning() {             
                        n.Stop()                       
                    }
                    os.Exit(1)
                }
            }()

            if err := n.Start(); err != nil { 
                return fmt.Errorf("Failed to start node: %v", err)
            }
            logger.Info("Started node", "nodeInfo", n.Switch().NodeInfo())

            // Run forever
            select {}

            return nil
        },
    }

    AddNodeFlags(cmd)
    return cmd
}
```

很直白，主要工作是构造并返回一个*cobra.Command*对象的指针，另外就是`AddNodeFlags(cmd)`，给node命令增加些标志或者参数，这个函数细节暂时不深究。

这里必须要跟C/C++做个比较，在C/C++中调用*malloc*或者*new*函数来分配对象的存储空间，并且在将来要记得*free*或者*delete*，而在go中创建对象就像上面代码这么做就可以了，将来也不需要手工回收空间，go是有垃圾收集机制的，而且虽然这看上去像是个局部变量，在go中也不必担心这个问题。

最后就是开始执行命令：

```go
	cmd := cli.PrepareBaseCmd(rootCmd, "TM", os.ExpandEnv(filepath.Join("$HOME", cfg.DefaultTendermintDir)))
	if err := cmd.Execute(); err != nil {
		panic(err)
	}
}
```
注意这里的*cmd*跟前面的*cmd*已经不是一回事，因为：
```go
func PrepareBaseCmd(cmd *cobra.Command, envPrefix, defaultHome string) Executor { 
    cobra.OnInitialize(func() { initEnv(envPrefix) })
    cmd.PersistentFlags().StringP(HomeFlag, "", defaultHome, "directory for config and data")
    cmd.PersistentFlags().Bool(TraceFlag, false, "print out full stack trace on errors")
    cmd.PersistentPreRunE = concatCobraCmdFuncs(bindFlagsLoadViper, cmd.PersistentPreRunE)
    return Executor{cmd, os.Exit}  
}  
```

而：

```go
type Executor struct {
    *cobra.Command
    Exit func(int) // this is os.Exit by default, override in tests
}
```

**不是太明白为什么这里要继续使用*cmd*这个名字，大概是作者的恶趣味吧。**

*cmd.Execute()*实际上就是

## cli.Execute()

```go
func (e Executor) Execute() error {
    e.SilenceUsage = true     
    e.SilenceErrors = true    
    err := e.Command.Execute()
    ...
    return err
}                
```

并没有多少东西，除了这一句`err := e.Command.Execute()`，我们上面已经看到了，这里饶了一个圈，又回头执行Command去了，到这里为止，我们知道这个Command就是rootCmd。于是

## cobra.Execute()

```go
func (c *Command) Execute() error {
    _, err := c.ExecuteC()    
    return err
}  
```

继续绕圈

## cobra.ExecuteC()

```go
func (c *Command) ExecuteC() (cmd *Command, err error) {
    // Regardless of what command execute is called on, run on Root only
    if c.HasParent() {
        return c.Root().ExecuteC() 
    }
```
当然，这里我们知道rootCmd是没有父命令的，所以往下看
```go
// windows hook
    if preExecHookFn != nil { 
        preExecHookFn(c)      
    }
   
    // initialize help as the last point possible to allow for user
    // overriding
    c.InitDefaultHelpCmd()
```
到这里对理解流程是无关紧要的，我们先不管它。
```go
    var args []string

    // Workaround FAIL with "go test -v" or "cobra.test -test.v", see #155
    if c.args == nil && filepath.Base(os.Args[0]) != "cobra.test" {
        args = os.Args[1:]
    } else {
        args = c.args
    }
```
保证args是除了程序名字也就是tendermint之外剩下的命令行参数。
```go
    var flags []string
    if c.TraverseChildren {
        cmd, flags, err = c.Traverse(args)
    } else {
        cmd, flags, err = c.Find(args)
    }
    if err != nil {
        // If found parse to a subcommand and then failed, talk about the subcommand
        if cmd != nil {
            c = cmd
        }
        if !c.SilenceErrors {
            c.Println("Error:", err.Error())
            c.Printf("Run '%v --help' for usage.\n", c.CommandPath())
        }
        return c, err
    }
```
以上找到子命令，比方说*init*,*node*以及对应这些子命令的参数列表，然后就是执行这个子命令
```go
    err = cmd.execute(flags)
    ...
    return cmd, err
}

```

于是我们到达了问题的核心

## cobra.execute(a []string) (err error)

这个函数事实上是一条命令的执行的流程，就不细说了，出于我们的目的，只看跟*node*这条命令有关的部分。前面我们看到，在定义*node*的时候定义了一个函数变量*RunE*，这个函数变量对应的函数会在*execute*执行过程中调用，所以，我们只要看*RunE*所代表的那个函数

## commands.NewRunNodeCmd.RunE

```go
RunE: func(cmd *cobra.Command, args []string) error {
            n, err := nodeProvider(config, logger)
            ...
            if err := n.Start(); err != nil { 
                return fmt.Errorf("Failed to start node: %v", err)
            }
            logger.Info("Started node", "nodeInfo", n.Switch().NodeInfo())

            // Run forever
            select {}

            return nil
        },
    }
```

这里我们有两个函数需要关注的，一个是*nodeProvider*，一个是*Start*，而*nodeProvider*正是前面我们已经见过的*DefaultNewNode*,

### DefaultNewNode

```go
func DefaultNewNode(config *cfg.Config, logger log.Logger) (*Node, error) {
    // Generate node PrivKey
    nodeKey, err := p2p.LoadOrGenNodeKey(config.NodeKeyFile())
    if err != nil {
        return nil, err
    }
    return NewNode(config,
        privval.LoadOrGenFilePV(config.PrivValidatorFile()),
        nodeKey,              
        proxy.DefaultClientCreator(config.ProxyApp, config.ABCI, config.DBDir()),
        DefaultGenesisDocProviderFunc(config),
        DefaultDBProvider,    
        DefaultMetricsProvider(config.Instrumentation),
        logger,
    )
}  
```

这个函数返回了一个新建的节点*n*。

于是

### n.Start()

```go
func (bs *BaseService) Start() error {
    if atomic.CompareAndSwapUint32(&bs.started, 0, 1) {
        if atomic.LoadUint32(&bs.stopped) == 1 {
            bs.Logger.Error(fmt.Sprintf("Not starting %v -- already stopped", bs.name), "impl", bs.impl)                                                                                         
            // revert flag    
            atomic.StoreUint32(&bs.started, 0)
            return ErrAlreadyStopped   
        }
        bs.Logger.Info(fmt.Sprintf("Starting %v", bs.name), "impl", bs.impl)                                                                                                                     
        err := bs.impl.OnStart()       
        if err != nil {       
            // revert flag    
            atomic.StoreUint32(&bs.started, 0)
            return err        
        }
        return nil
    }
    bs.Logger.Debug(fmt.Sprintf("Not starting %v -- already started", bs.name), "impl", bs.impl)                                                                                                 
    return ErrAlreadyStarted  
}  
```

最终问题的关键就在*OnStart*和*NewNode*上，在研究这两个函数之前，我们先看个时序图，从整体上把握一下  
## Tendermint 时序图

```sequence
note over main(): cmd/tendermint/main.go
note over commands: cmd/tendermint/commands/root.go
commands -> main(): rootCmd <- RootCmd
note over cobra: spf13/cobra/command.go
main() -> cobra: rootCmd.AddCommand(...)
note over node:node/node.go
node -> main(): nodeFunc <- DefaultNewNode 
note over commands:cmd/tendermint/commands/run_node.go
main()->commands:NewRunNodeCmd(nodeFunc)
commands->main():返回一个note命令
main()->cobra:rootCmd.AddCommand(note命令)
note over cli:libs/cli/setup.go
main()->cli:PrepareBaseCmd(rootCmd)
cli->main():cmd<-Executor{rootCmd, os.Exit}
main()->cli:cmd.Execute()
cli->cobra:Execute():rootCmd
cobra->cobra:ExecuteC():rootCmd
note over cobra:找到子命令及其参数->cmd:node
cobra->cobra:cmd.execute():node
note over commands:run_node.go
cobra->commands:RunE
commands->node:DefaultNewNode()
node->node:NewNode()
node->commands:n<-NewNode
note over common:libs/common/service.go 
commands->common:n.Start()
common->node:OnStart()
note over node:就到这，在OnStart函数里边启动了一系列命令
```
我们先看
## NewNode()
```go
// NewNode returns a new, ready to go, Tendermint Node.
func NewNode(config *cfg.Config,
	privValidator types.PrivValidator,
	nodeKey *p2p.NodeKey,
	clientCreator proxy.ClientCreator,
	genesisDocProvider GenesisDocProvider,
	dbProvider DBProvider,
	metricsProvider MetricsProvider,
	logger log.Logger) (*Node, error) {
```
上面这部分我们不管它，往下看

###BlockStore

```go
	// Get BlockStore
	blockStoreDB, err := dbProvider(&DBContext{"blockstore", config})
	if err != nil {
		return nil, err
	}
	blockStore := bc.NewBlockStore(blockStoreDB)
```
这里创建了一个数据库
*dbProvider*是什么呢：

#### dbProvider

```go
// DBContext specifies config information for loading a new DB.
type DBContext struct {
    ID     string
    Config *cfg.Config
}  
   
// DBProvider takes a DBContext and returns an instantiated DB.
type DBProvider func(*DBContext) (dbm.DB, error)
   
// DefaultDBProvider returns a database using the DBBackend and DBDir
// specified in the ctx.Config.
func DefaultDBProvider(ctx *DBContext) (dbm.DB, error) {
    dbType := dbm.DBBackendType(ctx.Config.DBBackend)
    return dbm.NewDB(ctx.ID, dbType, ctx.Config.DBDir()), nil
}  
```
没错，就是这个*DefaultDBProvider*，这是在*DefaultNewNode*函数里调用*NewNode*时传入的参数。这个函数最后返回一个由*NewDB*创建的名字是*ctx.ID*,位于*ctx.Config.DBDir()*目录的数据库，我们知道tendermint使用的是levelDB。

##### NewDB

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
*backends*是一个*map*：`var backends = map[DBBackendType]dbCreator{}`,并且初始化为空。

*dbCreator*是个函数变量：`type dbCreator func(name string, dir string) (DB, error)`。
而*DBBackendType*则是：

```go
type DBBackendType string
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
总之我们创建了一个名字叫做*blockstore*，以及如下的一个叫*state*的的levelDB数据库：

### State

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
### proxyApp

我们知道，tendermint启动一个节点有两种方式，一个是在启动*node*时带有*--proxy_app=kvstore*参数，这时候会启动内置的*kvstore*这个*ABCI APP*并与之建立三个连接(*consensus, mempool, query*)，一个是没有这个参数，那么就必须另外启动一个指定地址和端口的*ABCI APP*，所谓指定，指的是在配置文件里边设置好，再启动tendermint时，同样会建立同上的三个连接。

```go
	// Create the proxyApp and establish connections to the ABCI app (consensus, mempool, query).
	proxyApp := proxy.NewAppConns(clientCreator)
	proxyApp.SetLogger(logger.With("module", "proxy"))
	if err := proxyApp.Start(); err != nil {
		return nil, fmt.Errorf("Error starting proxy app connections: %v", err)
	}
```
*NewAppConns*要做什么呢？不过我们还是先看*clientCreator*，从*NewNode*的参数列表中我们可以看到这是一个*proxy.ClientCreator*

#### clientCreator

```go
type ClientCreator interface {
    NewABCIClient() (abcicli.Client, error)
}  
```

是一个接口，我们在回到*DefaultNewNode*，可以看到传入到*NewNode*的实际参数是*proxy.DefaultClientCreator(config.ProxyApp, config.ABCI, config.DBDir())*返回的这么一个*proxy.ClientCreator*,这三个参数分别代表*proxy*的地址，连接方式及数据库目录，连接方式缺省是*socket*。我们看看这个函数做了什么：

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

在go中，数值，字符串都是合法的*switch*的条件，*case*也不需要*break*，所以需要*fallthrough*，这是与c/c++不同的地方。另外，我们从这里可以看到其实tendermint启动node的时候有6种方式。我们先看*NewRemoteClientCreator*，因为对应的是缺省的*addr*，也就是*`"socket"`*，

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

而*ClientCreator*是一个*interface*，如果一定要按C++的逻辑，那么这个*remoteClientCreator*就是抽象类*ClientCreator*的子类，不过在go以及python中这里的逻辑就是所谓的鸭子类型。因为

```go
func (r *remoteClientCreator) NewABCIClient() (abcicli.Client, error) {
    remoteApp, err := abcicli.NewClient(r.addr, r.transport, r.mustConnect)
    if err != nil {
        return nil, errors.Wrap(err, "Failed to connect to proxy")
    }
    return remoteApp, nil
}
```

所以，虽然go并不个面向对象的程序设计语言，但是用鸭子类型实现了多态。

知道clientCreator是什么之后，我们再来看*NewAppConns*

#### NewAppConns

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

#### proxyApp.Start()

事实上这个函数我们可以不用关注，而只要实现*proxyApp*的*OnStart()*即可，换句话说就是*multiAppConn*实现了接口*AppConns*，也进而实现了*cmn.Service*。

##### OnStart()

```go
func (app *multiAppConn) OnStart() error {
    // query connection
    querycli, err := app.clientCreator.NewABCIClient()
    if err != nil {
        return errors.Wrap(err, "Error creating ABCI client (query connection)")
    }
    querycli.SetLogger(app.Logger.With("module", "abci-client", "connection", "query"))
    if err := querycli.Start(); err != nil { 
        return errors.Wrap(err, "Error starting ABCI client (query connection)")
    }
    app.queryConn = NewAppConnQuery(querycli)
   
    // mempool connection     
    memcli, err := app.clientCreator.NewABCIClient()
    if err != nil {           
        return errors.Wrap(err, "Error creating ABCI client (mempool connection)")
    }
    memcli.SetLogger(app.Logger.With("module", "abci-client", "connection", "mempool"))
    if err := memcli.Start(); err != nil {
        return errors.Wrap(err, "Error starting ABCI client (mempool connection)")
    }
    app.mempoolConn = NewAppConnMempool(memcli)
                              
    // consensus connection   
    concli, err := app.clientCreator.NewABCIClient()
    if err != nil {
        return errors.Wrap(err, "Error creating ABCI client (consensus connection)")
    }
    concli.SetLogger(app.Logger.With("module", "abci-client", "connection", "consensus"))
    if err := concli.Start(); err != nil {
        return errors.Wrap(err, "Error starting ABCI client (consensus connection)")
    }
    app.consensusConn = NewAppConnConsensus(concli)

    return nil
}

```

这里我们只需要看函数*Start*和*NewAppConnXXXX*。所有的*Start*如前所述，分别最后会走到这里

###### OnStart

```go
func (cli *socketClient) OnStart() error {
    if err := cli.BaseService.OnStart(); err != nil {                                                                                                                                            
        return err
    }
   
    var err error
    var conn net.Conn
RETRY_LOOP:
    for {
        conn, err = cmn.Connect(cli.addr) 
        if err != nil {
            if cli.mustConnect {               
                return err
            }
            cli.Logger.Error(fmt.Sprintf("abci.socketClient failed to connect to %v.  Retrying...", cli.addr))                                                                                   
            time.Sleep(time.Second * dialRetryIntervalSeconds)                                                                                                                                   
            continue RETRY_LOOP        
        }
        cli.conn = conn
   
        go cli.sendRequestsRoutine(conn)
        go cli.recvResponseRoutine(conn)                                                                                                                                                         
   
        return nil
    }
}  

```

这个函数很直白，如果*mustConnect==false*，这是缺省情况，就会一直尝试连接指定的*ABCI App*，或者说，节点和App的启动顺序是没有关系的。连接上了之后，就会启动两个线程：`cli.sendRequestsRoutine(conn)`和`cli.recvResponseRoutine(conn)`，这两个线程的细节就不说了，分别用于发送和接收数据。

###### NewAppConnXXXX

```go
type appConnQuery struct {    
    appConn abcicli.Client    
}
   
func NewAppConnQuery(appConn abcicli.Client) *appConnQuery {
    return &appConnQuery{
        appConn: appConn,     
    }
}  

type appConnMempool struct {
    appConn abcicli.Client
}

func NewAppConnMempool(appConn abcicli.Client) *appConnMempool {
    return &appConnMempool{
        appConn: appConn,
    }
}

type appConnConsensus struct {
    appConn abcicli.Client
}

func NewAppConnConsensus(appConn abcicli.Client) *appConnConsensus {
    return &appConnConsensus{
        appConn: appConn,
    }
}
```

很直白，就是分别保存了前面建立的3个连接。

### NewHandshaker

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
###  LoadState
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
### ValidatorSocketClient
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
### metrics
```go
	csMetrics, p2pMetrics, memplMetrics, smMetrics := metricsProvider()
```
### mempool
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
### evidence
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
### blockchain
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
###  Consensus
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
### Transaction indexing
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
### p2p
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
### switch
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
### return
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

## node.OnStart()

好吧，其实也没什么特别可说的，就是启动了一些服务。

## 关于service

以BlockPool为例

```go
type BlockPool struct {
    cmn.BaseService
    startTime time.Time

    mtx sync.Mutex
    // block requests         
    requesters map[int64]*bpRequester
    height     int64 // the lowest key in requesters.                                                                                                                                            
    // peers
    peers         map[p2p.ID]*bpPeer
    maxPeerHeight int64       
   
    // atomic
    numPending int32 // number of requests pending assignment or block response                                                                                                                  
   
    requestsCh chan<- BlockRequest 
    errorsCh   chan<- peerError
}  
```
这里定义了一个*Service*，*BlockPool*，出于我们的目的，只需要关注*cmn.BaseService*，至于*cmn*，是*github.com/tendermint/tendermint/libs/common*的别名，我们且去看看*BaseService*的定义

```go
type BaseService struct {
    Logger  log.Logger
    name    string
    started uint32 // atomic
    stopped uint32 // atomic
    quit    chan struct{}

    // The "subclass" of BaseService
    impl Service
}
```

忘掉*`// The "subclass" of BaseService`*的说法，事实上这里是*BaseService*组合了*Service*，如果一定要说也应该是*BaseService*是接口*Service*的子类而不是反过来。*Service*是个接口，要牢记这一点，接下来会有进一步说明。

```go
func NewBlockPool(start int64, requestsCh chan<- BlockRequest, errorsCh chan<- peerError) *BlockPool {                                                                                           
    bp := &BlockPool{
        peers: make(map[p2p.ID]*bpPeer),                                                                                                                                                         
   
        requesters: make(map[int64]*bpRequester),  
        height:     start,
        numPending: 0,

        requestsCh: requestsCh,        
        errorsCh:   errorsCh,
    }
    bp.BaseService = *cmn.NewBaseService(nil, "BlockPool", bp)
    return bp
}

```

这个函数创建了一个*BlockPool* *Service*，*bp*，我们看看*NewBaseService*做了什么

```go
func NewBaseService(logger log.Logger, name string, impl Service) *BaseService {
    if logger == nil {
        logger = log.NewNopLogger()
    }

    return &BaseService{
        Logger: logger,
        name:   name,
        quit:   make(chan struct{}),
        impl:   impl,
    }
}
```

所以这里出现了一个环：*bp.BaseService.impl=bp*,我们比较下*C++*的实现

```c++
class Service {
    virtual bool Start()=0;
    virtual bool OnStart()=0;
    
    virtual bool Stop()=0;
    virtual bool OnStop()=0;
    ...   
};
class BaseService:public Service{
    Logger logger;
    string name;
    virtual bool Start();
    virtual bool OnStart();
    
    virtual bool Stop();
    virtual bool OnStop()
    ...
};
class bp:public BaseService{
    bool OnStart();
    bool OnStop();
    ....
};
```

也就是继承就可以了。实际上，在*C++*里，纯虚类也只能被继承，如果一定要组合，那么*Service*就不能是一个接口。

毕竟go不是一门oop。

回过头来继续说*bp*，需要并且必须实现*Service*接口的*OnStart()*和*OnStop()*

```go
	type FooService struct {
        BaseService
        // private fields
    }

    func NewFooService() *FooService {
        fs := &FooService{
            // init
        }
        fs.BaseService = *NewBaseService(log, "FooService", fs)
        return fs
    }

    func (fs *FooService) OnStart() error {
        fs.BaseService.OnStart() // Always call the overridden method.
        // initialize private fields
        // start subroutines, etc.
    }

    func (fs *FooService) OnStop() error {
        fs.BaseService.OnStop() // Always call the overridden method.
        // close/destroy private fields
        // stop subroutines, etc.
    }

```

其余的函数都在*BaseService*实现：

```go
func (bs *BaseService) Start() error {
    if atomic.CompareAndSwapUint32(&bs.started, 0, 1) {
        if atomic.LoadUint32(&bs.stopped) == 1 {
            bs.Logger.Error(fmt.Sprintf("Not starting %v -- already stopped", bs.name), "impl", bs.impl)
            // revert flag
            atomic.StoreUint32(&bs.started, 0)
            return ErrAlreadyStopped
        }
        bs.Logger.Info(fmt.Sprintf("Starting %v", bs.name), "impl", bs.impl)
        err := bs.impl.OnStart()
        if err != nil {
            // revert flag
            atomic.StoreUint32(&bs.started, 0)
            return err
        }
        return nil
    }
    bs.Logger.Debug(fmt.Sprintf("Not starting %v -- already started", bs.name), "impl", bs.impl)
    return ErrAlreadyStarted
}
```

没有必要的话我们并不需要再去增加或者修改上面这段代码。

总结一下，如果我们需要定义一个*Service*实例，在这个实例的定义中必须包含一个*BaseService*的引用，在这个实例的构造函数，也就是通常的*MakeNewXXXX*一类的函数中，我们需要调用*NewBaseService*函数，并把这个*Service*实例作为参数传入，这样在*BaseService*里就包含了对这个实例的引用，接下来我们至少要实现*OnStart()*和*OnStop()*两个函数，这两个函数的接收者是*Service*实例本身，因为，比方说，在*BaseService*的*Start()*函数中会通过*impl*来调用*Service*实例的*OnStart()*，*OnStop()*同理。当然这是对*tendermint*的框架而言，并不是go本身需要如此。