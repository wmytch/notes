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

## Viper

那么问题来了，到目前为止只是用缺省值初始化了一个Config对象，我们如果修改了配置文件，程序在哪里读取我们自定义的值呢？就在这里

>err := viper.Unmarshal(conf)

Viper是个应用配置系统，源文件在*github.com/spf13/viper*，跟cobra一样，都不是Tendermint自己的包。

我们看看里边是怎么处理的。会涉及到反射，有一定难度。所以我们先简单了解下反射。

### 反射

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

然后我们再看一个例子，这个例子说明了怎样通过反射修改变量的值。

```go
var x float64 = 3.4
p := reflect.ValueOf(&x)
v := p.Elem()
v.SetFloat(7.1)
```
这么多应该够了，如果不够再回头补，接下来看Unmarshal()函数。
### Unmarshal()

```go
func Unmarshal(rawVal interface{}) error { return v.Unmarshal(rawVal) } 
func (v *Viper) Unmarshal(rawVal interface{}) error {
    err := decode(v.AllSettings(), defaultDecoderConfig(rawVal))
   
    if err != nil {
        return err
    }
   
    v.insensitiviseMaps() 
   
    return nil
}  
```

显然在*ParseConfig()*里面调用的是普通函数版本，而不是方法版本，当然这无关紧要。

```go
var v *Viper   
func init() {
    v = New()
}  
```

一个全局变量*v*及其初始化，注意这里用的是`=`而不是`:=`，因为`v`已经申明为一个全局变量。先简单了解下普通函数版本里边的*v*是什么。

```go
type Viper struct {
    // Delimiter that separates a list of keys
    // used to access a nested value in one go
    keyDelim string

    // A set of paths to look for the config file in
    configPaths []string

    // The filesystem to read config from.
    fs afero.Fs

    // A set of remote providers to search for the configuration
    remoteProviders []*defaultRemoteProvider

    // Name of file to look for inside the path
    configName string
    configFile string
    configType string
    envPrefix  string

    automaticEnvApplied bool
    envKeyReplacer      *strings.Replacer

    config         map[string]interface{}
    override       map[string]interface{}
    defaults       map[string]interface{}
    kvstore        map[string]interface{}
    pflags         map[string]FlagValue
    env            map[string]string
    aliases        map[string]string
    typeByDefValue bool

    onConfigChange func(fsnotify.Event)
}

func New() *Viper {
    v := new(Viper)
    v.keyDelim = "."
    v.configName = "config"
    v.fs = afero.NewOsFs()
    v.config = make(map[string]interface{})
    v.override = make(map[string]interface{})
    v.defaults = make(map[string]interface{})
    v.kvstore = make(map[string]interface{})
    v.pflags = make(map[string]FlagValue)
    v.env = make(map[string]string)
    v.aliases = make(map[string]string)
    v.typeByDefValue = false

    return v
}
```

构造一个对象的方法当然各种各样。这里我们又看到了一种。

接下来调用的是方法版本，里边有两个函数，分别是

> err := decode(v.AllSettings(), defaultDecoderConfig(rawVal))
>
> v.insensitiviseMaps() 

到这里颇为踌躇，到底应该深度优先还是广度优先呢？

我们还是先看下这两个函数

```go
func decode(input interface{}, config *mapstructure.DecoderConfig) error{
    decoder, err := mapstructure.NewDecoder(config)
    if err != nil {
        return err
    }
    return decoder.Decode(input)
}

func (v *Viper) insensitiviseMaps() {
    insensitiviseMap(v.config)
    insensitiviseMap(v.defaults)   
    insensitiviseMap(v.override)   
    insensitiviseMap(v.kvstore)    
} 
```

两个函数都不复杂，并且是有前序关系的，我们就按顺序来看。

先比较第一个函数的签名及调用语句：
```go 
func decode(input interface{}, config *mapstructure.DecoderConfig) error

err := decode(v.AllSettings(), defaultDecoderConfig(rawVal))
```

先看*v.AllSettings()*

```go
// AllSettings merges all settings and returns them as a map[string]interface{}.
func AllSettings() map[string]interface{} { return v.AllSettings() }
func (v *Viper) AllSettings() map[string]interface{} {
    m := map[string]interface{}{}
    // start from the list of keys, and construct the map one value at a time
    for _, k := range v.AllKeys() {
        value := v.Get(k)
        if value == nil {
            // should not happen, since AllKeys() returns only keys holding a value,
            // check just in case anything changes
            continue
        }
        path := strings.Split(k, v.keyDelim)
        lastKey := strings.ToLower(path[len(path)-1])
        deepestMap := deepSearch(m, path[0:len(path)-1])
        // set innermost value
        deepestMap[lastKey] = value
    }
    return m
}
```
*interface{}*的意义就不解释了，*v.AllSettings()*实际返回的是一个map。具体细节暂时放一放。

我们再看*defaultDecoderConfig(rawVal)*

```go
func defaultDecoderConfig(output interface{}) *mapstructure.DecoderConfig {
    return &mapstructure.DecoderConfig{
        Metadata:         nil,
        Result:           output,
        WeaklyTypedInput: true,
        DecodeHook:       mapstructure.StringToTimeDurationHookFunc(),
    }
}
```

返回了一个数据结构，这是能够跟decode的函数签名对的上的，而我们的配置对象成为其中一个域*Result*。

我们已经看到，在decode函数中的

```go
decoder, err := mapstructure.NewDecoder(config)
```

