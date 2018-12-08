# Tendermint 代码笔记 (一) 主流程

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

最终问题的关键就在*OnStart*和*NewNode*上，这两个函数我们将来研究node命令的时候再看。最后是个时序图，从整体上把握一下。
## Tendermint  node 时序图

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
note over node:就到这，在OnStart函数里边启动了一系列服务
```
