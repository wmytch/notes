# Tendermint 代码笔记 (三) 配置

[TOC]
## 缺省配置

在研究node命令及相关服务之前，先看看配置的处理。

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

那么问题来了，到目前为止只是用缺省值初始化了一个Config对象，我们如果修改了配置文件，程序在怎样获取我们自定义的值呢？前面已经提到在初始化RootCmd的时候就已经读取了配置文件，但还没有解析，也就是配置还没有完全生效，只有就在这里

>err := viper.Unmarshal(conf)

才会解析配置文件，并使其生效。

Viper是个应用配置系统，源文件在*github.com/spf13/viper*，跟cobra一样，都不是Tendermint自己的包。

我们不打算深究里边是怎么处理的。但是里边会涉及到反射，所以我们简单了解下反射。

## 反射

go语言中一个变量在反射中可以表示为一个二元组(Value,Type)，而*Value*又对应一个二元组*(Value,Type)*。

看个例子

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	u := User{"张三", 20}
    
	t := reflect.TypeOf(u)
	fmt.Println(t)
    
	v := reflect.ValueOf(u)
	fmt.Println(v)
    
    u1 := v.Interface().(User)
	fmt.Println(u1)
    
	t1 := v.Type()
	fmt.Println(t1)
    
    fmt.Println(t.Kind())
	fmt.Println(v.Kind())
}

type User struct {
	Name string
	Age  int
}
```

运行结果

```bash
main.User
{张三 20}
{张三 20}
main.User
struct
struct
```

`reflect.TypeOf(u)`就得到了变量*u*的*Type*`User`，`reflect.ValueOf(u)`就得到了变量*u*的*Value*`{张三 20}`，也就是说变量*u*的反射二元组是`({张三 20},main.User)`，而通过反射的*Value*或者*Type*的*Kind()*方法可以获取一个变量的原始类型，两者是等价的。另外可以通过*Value*的*Type()*方法获得一个*Value*的*Type*，但是并没有相应的*Value()*方法来获取一个Value的*Value*，毕竟那样太蠢了。

我们再看一个例子，这个例子说明了怎样通过反射修改变量的值。

```go
var x float64 = 3.4
p := reflect.ValueOf(&x)
v := p.Elem()
v.SetFloat(7.1)
```

了解这么多基本上已经够了。

最后，要说的是，如果我们要使用Viper作为自己的配置系统，并不需要做太多工作，依样画葫芦就行。