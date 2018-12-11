# Tendermint 代码笔记 命令和服务

[TOC]

应该说，命令和服务构成了Tendermint的经纬线，当然，不要纠缠哪个是经哪个是纬，或者也可以说，命令和服务构成了Tendermint整体架构的骨架。

## 命令 Command

在tendermint中，使用了一个叫cobra的包，我们所说的Command就在这个包中定义，具体的定义可以参见源文件，路径大概会是在*github.com/spf13/cobra/command.go*。

至于命令是什么，比方说，我们要编译一个go程序，会使用命令行:`go build xx.go`，这里的*build*就是命令，同样的，对于tendermint，启动一个节点时的命令行:`tendermint node`中的*node*就是命令。

而使用这个*cobra*包，我们可以以一种形式化的方式实现一条命令。下面开始举例说明。

首先就看*RootCmd*，这是tendermint中所有其他命令的根，但这并不是说RootCmd会类似于一个oop中的基类，而是更像一个容器，保存了一系列如上面说的命令。源文件是*github.com/tendermint/tendermint/cmd/tendermint/commands/root.go*。文件里边还有一些执行初始化的函数就不说了，只看这个定义：

```go
// RootCmd is the root command for Tendermint core.
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

这个定义也很平凡，就是创建了一个cobra.Command对象，并将其指针赋给RootCmd。由于在go语言当中`.`操作符统一了C/C++中的`.`和`->`，所以大多数情况下我们也不需要刻意去区分一个变量是指针还是实体，但是在实现方法的时候，如果没有特别的理由，最好还是用指针的形式。

在这个Command命令中，我们看到申明了一个函数`PersistentPreRunE: func(cmd *cobra.Command, args []string) (err error)`，当然不是要讨论这个函数本身，而是说，Command的类型定义里，申明了一系列函数变量，我们自定义命令的时候可以根据需要自行实现一些函数，这些函数是有个执行顺序的：

```go
// The *Run functions are executed in the following order:                                                                                                                                   
    //   * PersistentPreRun() 
    //   * PreRun()
    //   * Run() 
    //   * PostRun()
    //   * PersistentPostRun()
    // All functions get the same args, the arguments after the command name. 
```

而`PersistentPreRunE`这个函数由于是在*RootCmd*中申明的，所以其所有的子命令都会执行这个函数。通常来说，子命令只需要实现*Run()*或者*RunE()*。

接下来我们看一条子命令，比方说*init*。其源文件是*github.com/tendermint/tendermint/cmd/tendermint/commands/init.go*

```go
var InitFilesCmd = &cobra.Command{
    Use:   "init",
    Short: "Initialize Tendermint",
    RunE:  initFiles,
}

func initFiles(cmd *cobra.Command, args []string) error {
    return initFilesWithConfig(config)
}

```

这里定义了一个函数*initFiles*，并将其赋给了*RunE*这个函数变量，而*`initFilesWithConfig(config)`*参看源文件，就不讨论了。

实现了这么一条命令后，就可以通过*rootCmd.AddCommand( cmd.InitFilesCmd)*将其加入到rootCmd的命令树中，*rootCmd*是一个指针，被初始化为前面所说的*RootCmd*，当然，这也是个指针。

我们已经知道，在执行一条命令时，并不会从命令树中取命令，而最多只是通过遍历树来判断一条命令是否存在。

一条命令的执行流程我们已经讨论过，就不再详述。总之，在*github.com/spf13/cobra/command.go*中定义的这个函数`func (c *Command) execute(a []string) (err error)`，规定了这个流程。

而正如我们已经讨论过的，***node*这条命令中的服务就是在其*RunE*中启动的。**

在讲主流程的时候我们已经看到了

### PrepareBaseCmd

```go
func PrepareBaseCmd(cmd *cobra.Command, envPrefix, defaultHome string) Executor { 
    cobra.OnInitialize(func() { initEnv(envPrefix) })
    cmd.PersistentFlags().StringP(HomeFlag, "", defaultHome, "directory for config and data")
    cmd.PersistentFlags().Bool(TraceFlag, false, "print out full stack trace on errors")
    cmd.PersistentPreRunE = concatCobraCmdFuncs(bindFlagsLoadViper, cmd.PersistentPreRunE)
    return Executor{cmd, os.Exit}  
}  
```

这个函数实际上整个程序的初始化函数，包括但不限于读取配置文件。

```go
cobra.OnInitialize(func() { initEnv(envPrefix) })

func OnInitialize(y ...func()) {
    initializers = append(initializers, y...)
}  
```

显然，这里是设置了一些初始化函数，这些初始化函数会在*preRun*函数里依次执行，注意是*preRun*，不是*PreRun*。如前所述，可以参见command.go里的`func (c *Command) execute(a []string) (err error)`，就不细说了。

接下来我们再看

```go
cmd.PersistentPreRunE = concatCobraCmdFuncs(bindFlagsLoadViper, cmd.PersistentPreRunE)
```

这里发生了什么呢？我们知道这时候的cmd就是RootCmd，而我们在初始化RootCmd的时候已经申明了一个PersistentPreRunE，而这里又重新给PersistentPreRunE赋了一个值，当然要记得这时候还没有开始执行RootCmd。显然，concatCobraCmdFuncs这个名字已经提示了我们这里只是把两个函数连接起来，怎么连接的呢，且看

```go
func concatCobraCmdFuncs(fs ...cobraCmdFunc) cobraCmdFunc {
    return func(cmd *cobra.Command, args []string) error {
        for _, f := range fs {
            if f != nil {
                if err := f(cmd, args); err != nil {
                    return err
                }
            }
        }
        return nil
    }
}
```

这里需要注意的是

- 可变参数列表(fs ...cobraCmdFunc)在调用这个函数的时候就已经确定并且被闭包锁捕获，也就是两个函数变量*bindFlagsLoadViper*和*cmd.PersistentPreRunE*，不存在将来执行PersistentPreRunE时才确定fs是什么的问题。

- RootCmd初始化时定义的PersistentPreRunE是存在于内存中并且可被寻址的，不然就无从作为argument传入。

于是，这个函数就成为rootCmd的新的PersistentPreRunE将来被调用。

接下来看

```go
func bindFlagsLoadViper(cmd *cobra.Command, args []string) error {
    // cmd.Flags() includes flags from this command and all persistent flags from the parent
    if err := viper.BindPFlags(cmd.Flags()); err != nil {
        return err
    }

    homeDir := viper.GetString(HomeFlag)
    viper.Set(HomeFlag, homeDir)   
    viper.SetConfigName("config")                         // name of config file (without extension)
    viper.AddConfigPath(homeDir)                          // search root directory       
    viper.AddConfigPath(filepath.Join(homeDir, "config")) // search root directory /config
       
    // If a config file is found, read it in.
    if err := viper.ReadInConfig(); err == nil {
        // stderr, so if we redirect output to json file, this doesn't appear
        // fmt.Fprintln(os.Stderr, "Using config file:", viper.ConfigFileUsed())
    } else if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
        // ignore not found error, return other errors
        return err
    }                         
    return nil
}


```

如前面所说，读取了配置文件，但这并不意味着配置就生效了，而是要到在*PersistentPreRunE()*中调用到*ParseConfig()*时才会真正对配置进行解析。

## 服务 Service

说完命令，我们讨论服务。以BlockPool为例

​```go
type BlockPool struct {
​    cmn.BaseService
​    startTime time.Time

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

​```go
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

也就是继承就可以了。实际上，在*C++*里，纯虚类也只能被继承，如果一定要组合，那么*Service*就不能是一个接口，讲道理，这里BaseService和Service应该交换名字才对。

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

总结一下，如果我们需要定义一个*Service*实例，在这个实例的定义中必须包含一个*BaseService*的引用，在这个实例的构造函数，也就是通常的*MakeNewXXXX*一类的函数中，我们需要调用*NewBaseService*函数，并把这个*Service*实例作为参数传入，这样在*BaseService*里就包含了对这个实例的引用，接下来我们至少要实现*OnStart()*和*OnStop()*两个函数，这两个函数的接收者是*Service*实例本身，因为，比方说，在*BaseService*的*Start()*函数中会通过*impl*来调用*Service*实例的*OnStart()*，*OnStop()*同理，为了方便理解和记忆，*impl*就是实现的意思，实现什么呢？当然是*Service*这个接口。当然这是对*tendermint*的框架而言，并不是go本身需要如此。