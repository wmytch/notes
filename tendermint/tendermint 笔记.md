# Tendermint笔记
[TOC]

## 概览
### Core与ABCI

#### 职责

如果要建立一个类比特币的系统
- Tendermint core的职责是
  - 在节点间共享区块和交易
  - 为交易建立一个规范且不可更改的顺序(区块链)

- ABCI App的职责是

  - 维护UTXO数据库

  - 验证交易的加密签名

  - 阻止处理不存在的交易

  - 给客户端提供访问UTXO数据的机制

注意**ABCI是个接口，不是app本身**

#### 消息
- DeliverTx消息是Core通过ABCI向App发送，区块链中的每一个交易都通过这个消息传送，App处理要验证交易，验证的依据是当前状态、应用协议以及加密证书，交易验证通过后更新App状态，比如在一个kv存储中绑定一个值，或者更新UTXO数据库
- CheckTx消息只是用来验证交易。需要验证的交易处于Mempool中，Core向App发送本条消息是附带有需要验证的交易，如DeliverTx消息一样，App只验证这个交易的合法性，并返回验证结果，但是不更新状态。
- Commit消息用来对当前应用状态做个加密运算，运算结果作为一个承诺，放在下一个区块的头部。大概就相当于保存一个快照，可以用来验证一致性。
#### 连接
Core与App之间有3条ABCI连接，一条用来验证在mempool中广播的交易，一条用来给共识引擎运行区块提议，一条用来查询app状态。

### Consensus Overview

- Tendermint是一个BFT共识协议
- 协议的参与者称为校验者
- 校验者轮流提议交易区块，然后对这些区块进行投票
- 区块被提交到链上，每一块都有一个高度(height)
- 如果一个区块提交失败，则进入一个新回合(round)，并且由一个新的校验者重新提议，而高度不变
- 两阶段投票：pre-vote和pre-commit
- 一个回合中，一个区块只有收到了超过2/3的校验者的pre-commit，才会被提交
- 当一个区块收到2/3的pre-vote时，这种情况被称为**polka**
- 每一个pre-commit都必须只针对同回合中的一个polka
- tendermint允许有个超时时间
- 为了保证一个校验者不会在同一高度提交矛盾的区块，引入了一些锁定机制
- 如果一个校验者预提交了一个区块，他就被锁定在了这个区块，那么
  - 他必须预表决他被锁定在的这一块
  - 只有在新的回合中，有一个新块的polka，他才能解锁并预提交这个新块
  - 意思就是如果预提交了一个区块，那么这个校验者就不能再对这一块做预提交了，预表决还是可以的，要再做预提交的动作，只能等下一个新的polka。

### 临时标题

- Byzantine Fault Tolerant State Machine Replication 拜占庭容错状态机拷贝
- hash-linked batches of transactions 许多事务批组成一个链，这个链的每一个元素也就是每一个事务批中有一个字段用来存放前一个元素的hash值，这些许多事务批就通过这样的方式组成一个链
- blocks 区块 上条所说的事务批也就被称为区块
- blockchain 区块链就由这些区块组成
- Height 区块的唯一索引，这个值是唯一并且严格单调的
- 一些被赋予权重的validator或者说验证者成为一个集合，区块就由这个集合里的成员提交
- 这个验证者集合的成员和权重是随时间而变的
- 只要这些验证者中恶意的或者有缺陷的成员不超过1/3，这个区块链就是安全并且可用的的。
- 一个commit指一个带签名的消息的集合，这些消息来自当前验证者集的成员，这些成员的权重之和超过总权重的2/3
- 验证者各自对区块进行提议和表决，这里的表决，就是投赞成票，收到足够的赞成票或者说表决数之后，这一区块就被认为提交了
- 这些表决信息会包含在下一区块中，毕竟当前正在处理的区块已经创建，而区块链当中区块一经创建便不能更改，没有包含在表决信息当中的验证者就被忽略掉了，不论是没有投票赞成还是没有参加表决
- 区块一旦提交，就可以由一个应用来进行一些处理，比方说返回区块中事务的一些结果
- 这些应用也可以返回一些区块之外的信息，比如验证者集合的变化，以及最近状态的加密摘要
- Tendermint 用来对区块链的最近状态进行验证和认证
- 因此，在区块的header部分包含了一些加密的信息作为承诺
- 这些信息包括区块的目录(所包含的事务)，提交这一区块的验证者，以及由应用返回的其他一些结果。需要注意的是应用返回的结果只能包含在下一个区块中，因为应用只有在区块提交之后才会对区块进行处理，而区块一经创建便不可更改
- 而事务结果和验证者集合并不直接包含在区块中，只是一个加密的摘要(merkel树的根)
- 因此，验证一个区块需要一个单独的数据结构来存储这些信息，这些信息称为state
- 区块验证需要访问前一个区块

## Blockchain--数据结构

### Block

```go
type Block struct {
    Header      Header
    Txs         Data
    Evidence    EvidenceData
    LastCommit  Commit
}
```

- Header 就是Header
- Txs 代表事务，是个Data类型
- Evidence 是一个list，指的是一些不合法的行为，比方说签名不符合的表决
- LastCommit 顾名思义，上一区块的Commit，也就是上一区块的表决数据，如前所述

#### Header

区块头部是一些元数据，关于区块本身、共识、承诺，还有应用返回的结果

```go
type Header struct {
	// basic block info
	Version  Version
	ChainID  string
	Height   int64
	Time     Time
	NumTxs   int64
	TotalTxs int64

	// prev block info
	LastBlockID BlockID

	// hashes of block data
	LastCommitHash []byte // commit from validators from the last block
	DataHash       []byte // Merkle root of transactions

	// hashes from the app output from the prev block
	ValidatorsHash     []byte // validators for the current block
	NextValidatorsHash []byte // validators for the next block
	ConsensusHash      []byte // consensus params for current block
	AppHash            []byte // state after txs from the previous block
	LastResultsHash    []byte // root hash of all results from the txs from the previous block

	// consensus info
	EvidenceHash    []byte // evidence included in the block
	ProposerAddress []byte // original proposer of the block

```

##### Version

包括区块链和应用的协议版本

```go
type Version struct {
    Block   uint64
    App     uint64
}
```

##### BlockID

 `BlockID` 包含区块的两个不同的Merkle树根。第一个，作为区块的主hash，是头部所有域的Merkle树根。第二个，在共识期间用来保护区块的安全传播，是完全序列化的区块切分之后的一个Merkle树根。

```go
type BlockID struct {
    Hash []byte
    Parts PartsHeader
}

type PartsHeader struct {
    Hash []byte
    Total int32   //序列化之后的block切分成了Total这么多part
}
```

##### Time

Time是一个Google.Protobuf.WellKnownTypes.Timestamp的类型，其中包括两个整型数，一个用来表示秒，一个用来表示纳秒，当然都是从Epoch开始

#### Data

是一个事务list的封装

```go
type Data struct {
    Txs [][]byte
}
```

#### Commit

```go
type Commit struct {
    BlockID     BlockID
    Precommits  []Vote
}
```

如前所述，这是前一区块的commit信息

##### Vote

一个表决是一个验证者的签名信息，验证者都是针对某一特定区块而言，也就是说不同的区块参与表决的验证者是不必相同的。

```go
type Vote struct {
	Type             SignedMsgType  // byte
	Height           int64
	Round            int
	Timestamp        time.Time
	BlockID          BlockID
	ValidatorAddress Address
	ValidatorIndex   int
	Signature        []byte
}
```

如前所述，一个Commit中包含了区块的id，以及对这个区块进行了表决的验证者列表，这里说的列表就是通常理解的列表，用slice而不是list来保存。

tendermint的投票包括两轮，所以`vote.Type == 1`就表示是*prevote*轮，而`vote.Type == 2`则表示*precommit*轮

##### Signature

ED25519签名，64个字节的裸数据

#### EvidenceData

```go
type EvidenceData struct {
    Evidence []Evidence
}
```

##### Evidence

注意，这个`Evidence`是`[]`后面那个`Evidence`

`Evidence`是个接口，所以：

```go
// amino name: "tendermint/DuplicateVoteEvidence"
type DuplicateVoteEvidence struct {
	PubKey PubKey
	VoteA  Vote
	VoteB  Vote
}
```

Evidence保存的是有冲突的表决，所谓冲突并不是赞成和否决的冲突，因为在Vote列表中并不存在否决提议的验证者，而是Vote中的一些数据的不一致，比方说签名，要记得一个公钥实际上唯一标识了一个验证者。

## Blockchain -- Validation

### Header

#### Version

```go
block.Version.Block == state.Version.Block
block.Version.App == state.Version.App
```

#### ChainID

```go
len(block.ChainID) < 50
```

#### Height

```go
block.Header.Height > 0
block.Header.Height == prevBlock.Header.Height + 1
```

#### Time

```go
block.Header.Timestamp >= prevBlock.Header.Timestamp + 1 ms
block.Header.Timestamp == MedianTime(block.LastCommit, state.LastValidators)
```

时间戳必须是单调的，并且是加权的中值，注意不是平均值。另外，一个表决的时间戳必须至少比要表决的那一区块的时间戳大一毫秒。

另外：

```go
if block.Header.Height == 1 {
    block.Header.Timestamp == genesisTime
}
```

事实上有这么个关系：

```go
if block.Height == 1 { //姑且先忽略这里的block跟上面的block的关系,只要注意genesisTime的来源就行  
        genesisTime := state.LastBlockTime
    	...
}
```

#### NumTxs

```go
block.Header.NumTxs == len(block.Txs.Txs)
```

回想一下之前的定义：

```go
type Block struct {
    Header      Header
    Txs         Data
    Evidence    EvidenceData
    LastCommit  Commit
}
type Data struct {
    Txs [][]byte
}
type Header struct {
	...	
	NumTxs   int64
	TotalTxs int64
    ...
}
```

#### TotalTxs

```go
block.Header.TotalTxs == prevBlock.Header.TotalTxs + block.Header.NumTxs
```

所以`TotalTxs`是个累加值，表示了区块链中所有的事务的数量。对于第一个区块，显然有`block.Header.TotalTxs = block.Header.NumberTxs`

#### LastBlockID

```go
type Header struct {
	...
	// prev block info
	LastBlockID BlockID
    ...
}
type BlockID struct {
    Hash []byte
    Parts PartsHeader
}

type PartsHeader struct {
    Hash []byte
    Total int32
}


prevBlockParts := MakeParts(prevBlock, state.LastConsensusParams.BlockGossip.BlockPartSize)
block.Header.LastBlockID == BlockID {
    Hash: SimpleMerkleRoot(prevBlock.Header),
    PartsHeader{
        Hash: SimpleMerkleRoot(prevBlockParts),
        Total: len(prevBlockParts),
    },
}
```

自然，第一区块的 `block.Header.LastBlockID == BlockID{}`.

另外，`state.LastConsensusParams`可能会被应用改变。

**还有别的内容，并不费解，所以直接看原文档即可**

## State

```go
type State struct {
    Version     Version
    LastResults []Result
    AppHash []byte

    LastValidators []Validator
    Validators []Validator
    NextValidators []Validator

    ConsensusParams ConsensusParams
}
```

### Result

```go
type Result struct {
    Code uint32
    Data []byte
}
```

### Validator

```go
type Validator struct {
    Address     []byte
    PubKey      PubKey
    VotingPower int64
}
```

### ConsensusParams

```go
type ConsensusParams struct {
	BlockSize
	Evidence
	Validator
}

type BlockSize struct {
	MaxBytes        int64
	MaxGas          int64
}

type Evidence struct {
	MaxAge int64
}

type Validator struct {
	PubKeyTypes []string
}
```

## Byzantine Consensus Algorithm

### Terms
- 直接与某个节点连接的节点称为*peers*.
- 一次共识过程会进行多个 *回合(rounds)*.
- `NewHeight`, `Propose`, `Prevote`, `Precommit`,  `Commit` 代表一个回合当中状态机的状态，也称之为`RoundStep` 或者 "step"。通俗的说所谓状态就是某个点上的时空特性，状态会相互转化，转化又把状态联系起来，这么一种概念称为状态机.
- 一个节点必然位于一个给定的`(height，round，step)`或者说  `(H,R,S)`, 也可以省略掉***步***简化成 `(H,R)` 
- *prevote* 或者 *precommit* 表示广播一个prevote Vote数据或者一个首次precommit Vote数据
- 验证者收到广播的vote数据之后，就会进行表决，也就是签名。在`(H,R)`上的一个表决或者说vote，就是在相应的Vote数据块的Signature字段填上内容 。要注意的是一个vote对应一个验证者，共识过程会收集这些vote
- *+2/3*  “超过 2/3"
- *1/3+*  "1/3 或者更多"
- A set of +2/3 of prevotes for a particular block or `<nil>` at `(H,R)` is called a *proof-of-lock-change* or *PoLC* for short. 这一条不知道啥意思，先保留。大概指的是处于`(H,R)`状态的或者是一个特定的区块，或者什么都没有(`<nil>`)，有一个集合，里边收集了超过2/3的预表决，这个集合就称为**proof-of-lock-change**或者**PoLC**,也就是**锁定变更证明**？理解一下，因为区块不可更改，所以超过2/3的表决记录就可以证明这一区块没有被更改？

### State Machine Overview

```go
NewHeight -> (Propose -> Prevote -> Precommit)+ -> Commit -> NewHeight ->...
```

`(Propose -> Prevote -> Precommit)`就是一个回合(round)，`+`表示可以有多个回合。会有多个回合的原因大概有：

- 指定的提交参与者不在线
- 指定的提交者提交的区块非法
- 指定提交者提交的区块没有及时发布
- 在 `Precommit`前没有及时收到足够的预表决票数。或者即使收集到了2/3的预表决，但是其中有至少一个验证者的签名是`<nil>`或者干了些见不得人的事情，也就是说至少有一个验证者的vote数据是非法的。
- 参与预提交的验证者投票权重达不到2/3。

### Background Gossip
- 节点传播`PartSet` ，也就是把当前回合提议的区块分成多个子块来传播
- 节点传播prevote/precommit表决。一个节点 `NODE_A` 位于`NODE_B`之前，它向 `NODE_B` 发送当前回合的prevotes或者precommits ，然后`NODE_B`'可以进一步转发 
- 如果有PoLC (proof-of-lock-change)提议，则节点传播对这个提议的prevote
- Nodes gossip to nodes lagging in blockchain height with block commits for older blocks.这个不确定啥意思，大概是在区块链上如果发现height并不是当前height的区块，节点也会传播
- 节点有时候会传播 `HasVote` 消息以告知peers自己已经拥有的vote
- 节点向所有的邻居广播自己的当前状态，但不会进一步传播
### Proposals

每个回合的提议都由指定提议者签名并公布。提议者由一个确定且没有歧义的轮转算法根据其投票权占比挑选出来。 在 `(H,R)` 上的一个提议由一个区块和一个可选部分组成，这个可选部分是最近的PoLC回合，有`PoLC-Round < R`，如果这个建议者知道这个回合存在的话。这意味着网络允许节点在安全的时候解锁以确保liveness property(这里指表决最终一定会返回一个正确的结果)。

### State Machine Spec

#### Common exit conditions

- 收到+2/3 预提交 -->  `Commit(H)`
- 在 `(H,R+x)`收到+2/3的预表决. --> `Prevote(H,R+x)`
- 在任意 `(H,R+x)`收到+2/3的预提交. -->  `Precommit(H,R+x)`

#### Propose Step (height:H,round:R)

- 进入：提议者提交一个`(H,R)`的区块

- 结束：
  - `timeoutProposeR`  -->  `Prevote(H,R)` 

  - 收到一个区块提议后并且所有的预表决都在 `PoLC-Round` -->  `Prevote(H,R)` 

  - 上面的Common exit conditions

#### Prevote Step (height:H,round:R)

进入 `Prevote`的每一个验证者都广播其预表决的表决。

- 如果在 `LastLockRound`之后验证者被锁定在一个区块，但是现在有一个在`PoLC-Round` 回合的PoLC，且`LastLockRound < PoLC-Round < R`, 则解锁
- 如果验证者仍被锁定在一个区块上，则将这个区块预表决(锁定了这个验证者的那一块)
- 如果没有被锁，并且来自 `Propose(H,R)`的区块是好的，则预表决这一区块
- 否则，如果这个提议是无效或者没有按时收到，则预提交`<nil>`

`Prevote` 结束于:

 - 收到超过+2/3的验证者的预表决，不论这个预表决是对某一区块还是仅仅跳了了某一块或者只是提交了`<nil>` --> `Precommit(H,R)`
 - 在 `timeoutPrevote`后收到任意的+2/3预表决 --> `Precommit(H,R)` 
 -  上面的 common exit conditions

#### Precommit Step (height:H,round:R)

进入`Precommit`后，每个验证者广播其预提交表决

- 如果在 `(H,R)`，对某一个区块`B`，验证者有一个PoLC,则将it (re)locks (or changes lock to) and precommits则将 `B`(重新) 锁定或者将锁转向`B`,并对其预提交，然后设置 `LastLockRound = R`. 
- 否则，如果验证者对在 `(H,R)` 对 `<nil>`有一个PoLC，则将其解锁并预提交`<nil>`.
- 否则，保持锁不变并预提交 `<nil>`，对 `<nil>`的预提交表示”这个回合我没有看到PoLC，但是我确实收到了+2/3的预表决并且可以等一会“ 

预表决结束于:

- 对 `<nil>`收到+2/3的预提交 --> `Propose(H,R+1)` 
- 在 `timeoutPrecommit`收到任意的+2/3的预提交 --> `Propose(H,R+1)` 
- 上面的 common exit conditions

#### Commit Step (height:H)

- Set `CommitTime = now()`
- Wait until block is received. --> goto `NewHeight(H+1)`

#### NewHeight Step (height:H)

- Move `Precommits` to `LastCommit` and increment height.
- Set `StartTime = CommitTime+timeoutCommit`
- Wait until `StartTime` to receive straggler commits. --> goto `Propose(H,0)`

### Proofs

#### Proof of Safety

Assume that at most -1/3 of the voting power of validators is byzantine.
If a validator commits block `B` at round `R`, it's because it saw +2/3
of precommits at round `R`. This implies that 1/3+ of honest nodes are
still locked at round `R' > R`. These locked validators will remain
locked until they see a PoLC at `R' > R`, but this won't happen because
1/3+ are locked and honest, so at most -2/3 are available to vote for
anything other than `B`

这的意思大概是，假定在所有的验证者中拜占庭节点不超过1/3，那么在对回合R表决时，如果一个验证者做了提交，那说明它看到了有+2/3的验证者做了预提交，也就是说有2/3以上的表决权重通过了这个回合的表决并且是可信的。从另一方面看，这些已经在回合`R`做了预提交的验证者，自然已经开始了新一个回合`R'`的表决，因为拜占庭节点不超过1/3,那么在这个回合进行表决的诚实节点自然就超过了1/3。这些处于回合`R'`或者说锁定在`R'`的节点只有在他们看到了一个在回合`R'`的PoLC才会结束这个回合的表决，也就是解锁，然而这不可能，也就是不可能出现一个在回合`R'`的PoLC，因为这时候还有+1/3的诚实节点锁定在回合`R`,

