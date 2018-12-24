# Tendermint 代码笔记 (三) 配置与日志

[TOC]
## 配置

### 时序图

首先我们先看看tendermint初始化命令的时序图
```sequence
cmd/tendermint/main.go -> libs/cli/setup.go : cli.PrepareBaseCmd()
libs/cli/setup.go -> spf13/cobra/cobra.go : cobra.OnInitialize()
libs/cli/setup.go -> spf13/cobra/command.go : cmd.PersistentFlags()
spf13/cobra/command.go -> spf13/pflag/string.go : StringP()
libs/cli/setup.go -> spf13/cobra/command.go : cmd.PersistentFlags()
spf13/cobra/command.go -> spf13/pflag/bool.go : Bool()
libs/cli/setup.go -> spf13/cobra/command.go : concatCobraCmdFuncs()
spf13/cobra/command.go -> libs/cli/setup.go :  cmd.PersistentPreRunE 
libs/cli/setup.go -> cmd/tendermint/main.go : Executor{cmd, os.Exit}
```
这里我们最主要的是要知道配置文件在什么地方什么时候读取，实际上到cli.PrepareBaseCmd()返回为止也都没有去读配置文件，而只是在调用concatCobraCmdFuncs()时把一个读取配置文件的函数bindFlagsLoadViper跟cmd原来的PersistentPreRunE合并，最后直到这个命令真正执行的时候才会读取配置文件，我们已经知道了一条命令的执行流程，这里就不多说了。不过我们可以看下是怎么合并函数的。

```go
// Returns a single function that calls each argument function in sequence
// RunE, PreRunE, PersistentPreRunE, etc. all have this same signature
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

可以看到，这个函数返回一个函数实例，返回的函数签名与PersistentPreRunE等是一样，于是在这个返回的函数里边，依次调用传入的函数(bindFlagsLoadViper, cmd.PersistentPreRunE)。另一方面，go此时已经分别给这两个函数分配好了内存空间，不论这两个函数是在哪里定义的，或者说在调用concatCobraCmdFuncs形成参数列表时，就已经捕获了参数所列的函数的地址。因此，当这个匿名函数返回时就已经有了地址，其中所涉及的fs这个函数列表中的所有元素都已确定并且已经不可更改。

### 解析配置

接下来我们看配置怎么起效的。

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

要注意这个时候只是构造了一个对象，*PersistentPreRunE*还没有真正执行，这点是必须要清楚的。这个函数只有在真正执行命令时才会被调用。我们还可以在这段代码体会下go中结构的初始化方式，也就是结构体中的各个域都可以用`域名: 值,`这种形式初始化，甚至可以在里边新定义一个值。

我们看看ParseConfig做了些什么

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

可以看到，这里用DefaultConfig()初始化了一个新的Config实例，然后调用viper.Unmarshal解析这个新实例，包括但不限于用配置文件的内容覆盖缺省内容，最后将这个新实例返回，如前面我们在RootCmd的初始化代码中看到的那样，将这个新实例赋给了config这个全局变量。换句话说，到这里已经存在了两个独立的Config对象，不过借助于go的gc，我们可以不用担心这个问题。

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

这个函数就是设置了许多相关的缺省值，细节可以查阅源文件*tendermint/tendermint/config/config.go*。

正如我们已知，cli.PrepareBaseCmd()里边会合并PersistentPreRunE，因此当命令的执行流程来到PersistentPreRunE时，便会读取配置文件，进而解析这个配置文件
```go
err := viper.Unmarshal(conf)
```
Unmarshal()的工作主要是对配置进行解析处理，并用高优先级的配置代替低优先级的配置，比方说用配置文件里的值代替缺省配置里相应项的值。

### 总结

这里暂时不打算深究Viper，我们了解了tendermint对配置所作的相关操作。tendermint使用的配置文件格式叫toml，细节可以参阅相关文档。

在config目录中有两个文件，分别是config.go和toml.go。

toml.go中定义了几个函数，主要是在运行init命令时用来创建一个配置文件，并将缺省配置写入配置文件。以及，一个defaultConfigTemplate，这个文件里边的内容暂时不打算关注，只是要记得保持这个模板的内容跟config.go里边Config结构定义的同步。

config.go里边定义了Config及一些相关的数据类型及其缺省值，所以如果要更改tendermint的缺省值，就在这个文件里了。

总之，如果要使用Viper自定义一个配置系统，那么就需要修改这两个文件里边相关的内容，包括上面提到的几个函数：DefaultConfig()，ParseConfig()，以及在适当的时候读取配置文件，不过我们自己并不是必须合并命令的。当然，依照go的机制，我们需要一个全局的Config实例，在调用viper.Unmarshal时，可以直接使用这个实例，也可以使用一个新的实例，但不管用哪种方式，都需要保证解析之后的Config实例是一个全局并且可访问的，比如我们在前面看到的，一个新的实例被赋给了原来的全局config变量，原来的config变量所代表的那个实例就被丢弃并且会由go回收。

## 日志
在配置文件里，跟日志有关的配置只有两项
```bash
 Output level for logging, including package level options
log_level = "main:info,state:info,*:error"

# Output format: 'plain' (colored text) or 'json'
log_format = "plain"
```

所以我们不需要深入研究与日志有关的配置，因为就这么点内容。

在tendermint中日志对象是一个全局变量，一如配置对象。

```go
var (
    config = cfg.DefaultConfig()   
    logger = log.NewTMLogger(log.NewSyncWriter(os.Stdout))
)  
```

我们来到*libs/log/tm_logger.go*，可以看到如下的包

```go
	kitlog "github.com/go-kit/kit/log"
    kitlevel "github.com/go-kit/kit/log/level"
    "github.com/go-kit/kit/log/term"
```

不打算深入研究这个包，知道tendermint使用的日志系统是这个就可以。

在tm_logger.go文件中，定了几个日志输出函数，Info，Debug，Error，别的我们就不需要关注了，或者说，这里封装了上面所说的log系统，我们只需要使用这个封装即可。但是如果我们需要记录到某个日志文件里边，倒是可以修改一下log.NewSyncWriter(os.Stdout)的参数。

暂时这样吧。



