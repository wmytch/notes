### 关于`Import "github.com/gogo/protobuf/gogoproto/gogo.proto" was not found or had errors`  
实际上这个错误并不是找不到`gogo.proto`,而是找不到其中`import`的`google/protobuf/descriptor.proto`.
所以需要指定这个文件所在的目录。  
假定`descriptor.proto`文件在`$GOPATH/src/google/protobuf/`下(否则需要用`-I`指定一个路径，这个路径可以是相对的也可以是绝对的，但必须是`google/protobuf/`的父目录且从当前目录可达),`types.proto`文件在目录`$GOPATH/src/abci-test/types/`下,那么在`$GOPATH/src/`目录下执行命令：`protoc  --go_out=. abci-test/types/types.proto`就会在`$GOPATH/src/abci-test/types/`下生成相应的`types.pb.go`文件。至于其他所需要`import`的文件的路径也类似处理。
另外如果此时不在`$GOPATH/src/`下执行这条命令,生成`types.pb.go`会在一个奇怪的地方,不要问我为什么。
