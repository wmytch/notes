```go
type Request struct {         
    // Types that are valid to be assigned to Value:
    //  *Request_Echo
    //  *Request_Flush
    //  *Request_Info
    //  *Request_SetOption    
    //  *Request_InitChain
    //  *Request_Query
    //  *Request_BeginBlock
    //  *Request_CheckTx
    //  *Request_DeliverTx
    //  *Request_EndBlock
    //  *Request_Commit
    //  *Request_GetSpecTx
    Value                isRequest_Value `protobuf_oneof:"value"`       
    XXX_NoUnkeyedLiteral struct{}        `json:"-"`     
    XXX_unrecognized     []byte          `json:"-"`     
    XXX_sizecache        int32           `json:"-"`                                                                                                                                              
} 
type isRequest_Value interface {
    isRequest_Value()
    Equal(interface{}) bool
    MarshalTo([]byte) (int, error)
    Size() int
}
type Request_GetSpecTx struct {
    GetSpecTx *RequestGetSpecTx `protobuf:"bytes,13,opt,name=get_spec_tx,json=getSpecTx,oneof"`
}
type RequestGetSpecTx struct {
    Hash                 []byte   `protobuf:"bytes,1,opt,name=hash,proto3" json:"hash,omitempty"`
    XXX_NoUnkeyedLiteral struct{} `json:"-"`
    XXX_unrecognized     []byte   `json:"-"`
    XXX_sizecache        int32    `json:"-"`
}

&Request{ 
        Value: &Request_GetSpecTx{&RequestGetSpecTx{Hash:hash}},
    }
 
```

```sequence
cmdGetSpecTx->GetSpecTxSync:hash
GetSpecTxSync->ToRequestGetSpecTx:hash
ToRequestGetSpecTx->Request:hash
Request->Value:hash
Value->Request_GetSpecTx:hash
Request_GetSpecTx->RequestGetSpecTx:hash
RequestGetSpecTx->Request_GetSpecTx:RequestGetSpecTx
Request_GetSpecTx->Value:Request_GetSpecTx
Value->Request:Value
Request->ToRequestGetSpecTx:Request
ToRequestGetSpecTx->GetSpecTxSync:Request
GetSpecTxSync->queueRequest:Request
queueRequest->NewReqRes:Request
NewReqRes->queueRequest:reqres
queueRequest->GetSpecTxSync:reqres
GetSpecTxSync->cmdGetSpecTx:

```

```go
type ReqRes struct {          
    *types.Request            
    *sync.WaitGroup
    *types.Response // Not set atomically, so be sure to use WaitGroup.
                              
    mtx  sync.Mutex
    done bool                  // Gets set to true once *after* WaitGroup.Done().
    cb   func(*types.Response) // A single callback that may be set.
}
type Response struct {
    // Types that are valid to be assigned to Value:
    //  *Response_Exception
    //  *Response_Echo
    //  *Response_Flush
    //  *Response_Info
    //  *Response_SetOption
    //  *Response_InitChain
    //  *Response_Query
    //  *Response_BeginBlock
    //  *Response_CheckTx
    //  *Response_DeliverTx
    //  *Response_EndBlock
    //  *Response_Commit
    //  *Response_GetSpecTx
    Value                isResponse_Value `protobuf_oneof:"value"`
    XXX_NoUnkeyedLiteral struct{}         `json:"-"`
    XXX_unrecognized     []byte           `json:"-"`
    XXX_sizecache        int32            `json:"-"`
}

```

