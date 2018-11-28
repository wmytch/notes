# Tendermint 代码笔记

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
上面这一段就不说了，一看就明白。
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

接下来用*AddCommand*注册了许多命令，其中就包括我们前面看到的*init*和*testnet*,当然，命令名和命令行参数并不一样，也没有这样的要求。必须要注意的是这里只是注册，并没有执行这些命令。这些命令的细节暂时留到将来再研究。

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
前面的这段Note的意思如果你想写自己的什么什么，就把这个文件拷贝过去，然后自己添加命令就好啦。我们看代码，*nm.DefaultNewNode*的定义是：

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

这是个函数的定义，而`nodeFunc := nm.DefaultNewNode`也就意味着*nodeFunc*是个函数变量，从c/c++的角度来说这里要用函数指针来实现，在go里面不需要做这个区分。

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

这里必须要跟C/C++做个比较，在C/C++中要用*malloc*或者*new*函数来分配对象的存储空间，并且在将来要记得*free*或者*delete*，而在go中创建对象就像上面代码这么做就可以了，将来也不需要手工回收空间，go是有垃圾收集机制的。

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

不是太明白为什么这里要继续使用*cmd*这个名字。大概是作者的恶趣味吧。

*cmd.Execute()*实际上就是

## cli.Execute()