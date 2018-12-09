# Tendermint 代码笔记 (三) Config

[TOC]

在研究node命令及相关服务之前，先看看配置处理。

流程从rootCmd初始化开始，所以先来到*cmd/tendermint/commands/root.go*

```go
var (                         
    config = cfg.DefaultConfig()   
    logger = log.NewTMLogger(log.NewSyncWriter(os.Stdout))
)  
   
func init() {                 
    registerFlagsRootCmd(RootCmd)  
}  
   
func registerFlagsRootCmd(cmd *cobra.Command) {
    cmd.PersistentFlags().String("log_level", config.LogLevel, "Log level")
}
```

可以看到，最前面是两个全局变量*config*和*logger*的初始化，正如我们已知的，commands包被main包导入的时候就会做这个工作，同样的也会初始化一个全局的*RootCmd*变量，虽然这部分代码在后面才出现，但这并不影响执行顺序。

*init*函数肯定会在*main*函数之前执行，而这个时候一个*RootCmd*对象已经构造出来了，我们来看代码

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

要注意这个时候只是构造了一个对象，在*PersistentPreRunE*的定义中的`config, err = ParseConfig() `还没有执行，这点是必须要记住的。这个函数只有在真正执行命令时才会被调用。

```go
func ParseConfig() (*cfg.Config, error) {
    conf := cfg.DefaultConfig()    
    err := viper.Unmarshal(conf)   
    if err != nil {           
        return nil, err
    }
    conf.SetRoot(conf.RootDir)
    cfg.EnsureRoot(conf.RootDir)   
    if err = conf.ValidateBasic(); err != nil {
        return nil, fmt.Errorf("Error in config file: %v", err)
    }
    return conf, err          
} 

```

可以看到，原先已经初始化过的*config*变量又被*DefaultConfig()*初始化了一回。

```go
func DefaultConfig() *Config {
    return &Config{
        BaseConfig:      DefaultBaseConfig(),
        RPC:             DefaultRPCConfig(),            
        P2P:             DefaultP2PConfig(),            
        Mempool:         DefaultMempoolConfig(),        
        Consensus:       DefaultConsensusConfig(),      
        TxIndex:         DefaultTxIndexConfig(),        
        Instrumentation: DefaultInstrumentationConfig(),
    }
}  
```

这个函数就是设置了许多相关的缺省值，需要的话可以查阅源文件*github.com/tendermint/tendermint/config/config.go*。

关于config我们就暂时到这一步，知道配置哪里来的就可以了，需要的话将来再仔细研究。总觉得这里复杂化了。