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
libs/cli/setup.go -> libs/cli/setup.go  : c 
libs/cli/setup.go -> spf13/cobra/command.go : concatCobraCmdFuncs() -> PersistentPreRunE 
libs/cli/setup.go -> cmd/tendermint/main.go : Executor{cmd, os.Exit}
```
这里我们最主要的是要知道配置文件在什么地方什么时候读取，实际上到cli.PrepareBaseCmd()返回为止也都没有去读配置文件，而只是在调用concatCobraCmdFuncs()时把一个读取配置文件的函数跟cmd原来的PersistentPreRunE合并，最后直到这个命令真正执行的时候才会读取配置文件，我们已经知道了一条命令的执行流程，这里就不多说了。

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

正如我们已知，cli.PrepareBaseCmd()里边会合并PersistentPreRunE，因此当命令的执行流程来到PersistentPreRunE时，便会读取配置文件，进而解析这个配置文件
```go
err := viper.Unmarshal(conf)
```
Unmarshal()的工作主要是对配置进行解析处理，并用高优先级的配置代替低优先级的配置，比方说用配置文件里的值代替缺省配置里相应项的值。

### 封装

不过这里暂时不打算深究Viper，我们先了解tendermint对配置所作的相关封装。

在config目录中有两个文件，分别是config.go和toml.go。

tendermint使用的配置文件格式叫toml，具体的如果有兴趣将来再去研究。

toml.go中定义了几个函数，主要是在运行init命令时用来创建一个配置文件，并将缺省配置写入配置文件。以及，一个defaultConfigTemplate，这个文件里边的内容我们不需要太过关注，只需要记得保持这个模板的内容跟config.go里边相应项的同步。

config.go里边定义了Config及一些相关的数据类型及其缺省值，所以如果要更改tendermint的缺省值，就在这个文件里了。

### 反射

在config.go文件里面我们会看到这样的定义

```go
type MempoolConfig struct {   
    RootDir   string `mapstructure:"home"`   
    Recheck   bool   `mapstructure:"recheck"`
    Broadcast bool   `mapstructure:"broadcast"`
    WalPath   string `mapstructure:"wal_dir"`
    Size      int    `mapstructure:"size"`   
    CacheSize int    `mapstructure:"cache_size"`    
}
```

像

```go
`mapstructure:"home"`
```

这样的串称为struct的tag，tag总是跟反射联系在一起的，所以我们先看看反射。

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

我们再来看tag的例子。

```go
package main
 
import (
	"fmt"
	"reflect"
	"strings"
)
 
type Person struct {
	Name        string `label:"Person Name: " uppercase:"true"`
	Age         int    `label:"Age is: "`
	Sex         string `label:"Sex is: "`
	Description string
}
 
// 按照tag打印结构体
func PrintUseTag(ptr interface{}) error {
 
	// 获取入参的类型
	t := reflect.TypeOf(ptr)
 
	// 入参类型校验，t应该是个指针，并且其所指对象应该是个Struct
	if t.Kind() != reflect.Ptr || t.Elem().Kind() != reflect.Struct {
		return fmt.Errorf("参数应该为结构体指针")
	}
 
	// 取指针指向的结构体变量，这时候reflect.ValueOf(ptr)返回的是一个指针，其指向就是ptr所指
    // 于是v就是ptr所指的对象
    // 这样处理才能够修改ptr所指的对象的值
    v := reflect.ValueOf(ptr).Elem()
 
	// 解析字段,已知v是个结构体
	for i := 0; i < v.NumField(); i++ {
 
        // 取tag，v.Type()就是这个struct本身的类型Struct，Filed(i)获取到第i个域的信息，包括tag
		fieldInfo := v.Type().Field(i)
		tag := fieldInfo.Tag
 
		// 解析label tag
		label := tag.Get("label")
		if label == "" {
			label = fieldInfo.Name + ": "
		}
 
        // 解析uppercase tag，v.Field(i)返回第i个域的值，v.Type().Field(i)返回第i个域的信息
		value := fmt.Sprintf("%v", v.Field(i))
		if fieldInfo.Type.Kind() == reflect.String {
			uppercase := tag.Get("uppercase")
			if uppercase == "true" {
				value = strings.ToUpper(value)
			} else {
				value = strings.ToLower(value)
			}
		}
 
		fmt.Println(label + value)
	}
 
	return nil
}
 
func main() {
	person := Person{
		Name:        "Tom",
		Age:         29,
		Sex:         "Male",
		Description: "Cool",
	}
 
	PrintUseTag(&person)
}
```

运行结果会是

```bash
Person Name: TOM

Age is: 29

Sex is: male

Description: cool
```

实际上，config.go里面出现的tag，在Viper解析配置时我们就会看到上面类似的处理。

