# The latest gossip on BFT consensus

[TOC]



## I.  INTRODUCTION

- State Machine Replication (SMR)方法的关键在于各服务的拷贝启动于同样的初始状态，并且以同样的顺序处理请求(也可称为事务)，从而确保了相互之间的同步。
- 共识(consensus)在SMR方法中的角色是确保所有的服务副本以同样的顺序接收到事务请求。
- 在区块链中，每个节点只跟有限的节点相连，因此，通讯是通过一个基于消息扩散的p2p协议(gossip-based peer-to-peer protocol)，也就是像传播流言那样一传十十传百扩散出去的一种方式。
- Tendermint依照回合(round)顺次执行，每一个回合都有一个专门的提议者(也叫协调者或者领导者)，并且进入下一回合只是正常处理过程的一部分(不只是为了避免提议者有问题。这句与PBFT有关，且留待将来再研究)。

## II.  DEFINITIONS

### A. Model

Gossip communication:如果一个正确的进程 `p` 在时刻`t`收到了某个消息 `m` , 所有正确的进程都会在时刻`max{t, GST}+ ∆`前收到`m`，其中`GST`表示Global Stabilization Time，`∆`表示一个边界。

### B. State Machine Replication

- 复制准则：所有[无错]的副本都按照同一顺序接收和处理消息。所有的(无错)服务拷贝都收到所有的请求，所有的(无错)服务拷贝按照同样的顺序接受到请求。
- 只有客户端提议的请求(事务)才会被处理。事务的校验由(复制的)服务端处理，只处理合法的请求。

### C. Consensus

Tendermint解决状态机复制问题仍然是通过顺序执行共识实例来达到对每一个事务块的一致。考虑区块链系统中的一个拜占庭问题的变种，基于合法性断言的拜占庭共识(Validity Predicate-based Byzantine consensus):

- Agreement:No two correct processes decide on different values。只要进程是正确的，对同一个对象，他们所决定或者说输出的值必然是相同的。

- Termination:All correct processes eventually decide on a value.正确的进程必然会终止并且能够决定或者说输出一个值。

- Validity:A decided value is valid, i.e., it satisfies the predefined predicate denoted valid().决定或者输出的值必然是合法的，也就是说，能够满足预先定义的断言valid()。

断言`valid()`是依应用而定的，对于区块链，一个值如果不包含在区块链中它的上一个值(区块)的哈希值，那它就是不合法的。可见，这里的值并不是指一个通常的数，而是指的一个区块。

## III.  TENDERMINT CONSENSUS ALGORITHM

