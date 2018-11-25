# The latest gossip on BFT consensus

[TOC]



## I.  INTRODUCTION

- State Machine Replication (SMR)方法的关键在于各服务的拷贝启动于同样的初始状态，并且以同样的顺序处理请求(也可称为事务)，从而确保了相互之间的同步。
- 共识(consensus)在SMR方法中的角色是确保所有的服务副本以同样的顺序接收到事务请求。
- 在区块链中，每个节点只跟有限的节点相连，因此，通讯是通过一个基于消息扩散的p2p协议(gossip-based peer-to-peer protocol)，也就是像传播流言那样一传十十传百扩散出去的一种方式。
- Tendermint依照回合(round)顺次执行，每一个回合都有一个专门的提议者(也叫协调者或者领导者)，并且进入下一回合只是正常处理过程的一部分(不只是为了避免提议者有问题。这句与PBFT有关，且留待将来再研究)。

## II.  DEFINITIONS

### A. Model

Gossip communication:如果一个正确的进程 *p* 在时刻*t*收到了某个消息 *m* , 所有正确的进程都会在时刻`max{t, GST}+ ∆`前收到*m*，其中`GST`表示Global Stabilization Time，`∆`表示一个边界。

### B. State Machine Replication

- 复制准则：所有[无错]的副本都按照同一顺序接收和处理消息。所有的(无错)服务拷贝都收到所有的请求，所有的(无错)服务拷贝按照同样的顺序接受到请求。
- 只有客户端提议的请求(事务)才会被处理。事务的校验由(复制的)服务端处理，只处理合法的请求。

### C. Consensus

Tendermint解决状态机复制问题仍然是通过顺序执行共识实例来达到对每一个事务块的一致。考虑区块链系统中的一个拜占庭问题的变种，基于合法性断言的拜占庭共识(Validity Predicate-based Byzantine consensus):

- Agreement:没有两个正确的进程会对不同的值做决定，也就是说所有正确的进程所处理的值必然是同一个。可见，这里的值是一个待决定的对象，而不是决定输出的值。

- Termination:对一个值，正确的进程必然能做出决定,也就是说，必然会停机。

- Validity:要决定的值必然是合法的，也就是说，能够满足预定义的断言`valid()`。断言`valid()`依应用而定的，对于区块链，一个值如果不包含区块链中它的上一个值(区块)的哈希值，那它就是不合法的。这里的合法显然指的是这个值的结构是良构的。

## III.  TENDERMINT CONSENSUS ALGORITHM

- 在某个回合r，当一个正确的进程收到值为v的PROPOSAL的消息，并且收到的PRECOMMIT消息对`id(v)`有`2f+1`的表决权时，就可以对这个v做决定，也就是可以COMMIT
- 在某个回合r,要对v发送PRECOMMIT消息，一个正确的进程必须等待接收PROPOSAL消息以及相应的有`2f+1`投票权的PREVOTE消息，否则就会发送一个值为`nil`的PRECOMMIT消息，比方说超时或者发生某些确定是错误的事件时。也就是说，只有收到超过2/3的PREVOTE才会发送PRECOMMIT，确保每一个回合一个进程只能PRECOMMIT一个值或者`nil`。举个极端的例子，如果没有限定超过2/3，那么就有可能碰到各自占一半投票权的两个PREVOTE，这时候一个正确的进程就会发出两个值各不相同的PRECOMMIT，所以必须超过半数，超过多少合适这只是一个权衡。
- 如果一个进程接受了一个PROPOSAL，则发出值为`id(v)`的PREVOTE消息，否则发出一个值为`nil`的PREVOTE。

每个进程都维护这些变量：

- *step* 表示内部Tendermint状态机的当前状态，即当前回合算法执行到的阶段
- *lockedValue* 保存最近发出的PRECOMMIT中的值(考虑回合数） 
- *lockedRound* 进程发出的上一个不是`nil`的PRECOMMIT的回合数
- *validValue* 保存最近的可能决定的值
- *validRound* 上一次更新*validValue*的回合

一个正确的进程在回合r发送对`id(v)`的PRECOMMIT之前，设定`lockedValue=v`以及`lockedRound=r`，我们称之为将v锁定在回合r，换句话说，只有在预提交阶段才会进行锁定。一个正确的进程只有在收到`2f+1`的对`id(v)`的PRECOMMIT消息后才能够决定一个值，这就暗示着如果一个值v被相当于`f+1`表决权的正确进程锁定，这个值v就是一个***可能被决定的值***。因此，这个因此跟上面的暗示没关系，在某个回合，一个PROPOSAL的值v,收到了`2f+1`的PREVOKE消息之后，这个值就是一个***可能被决定的值***。

每个进程还保存这些变量，并且将它们与消息关联在一起：

- *h~p~*：当前共识实例，在Tendermint中称为*height*
- *round~p~*：当前回合数

最后，还有一个*decision~p~*，这是一个数组，用来存储决定。

每个回合开始于一个提议者用PROPOSAL消息建议一个值，对每一个height，在首回合提议者对值的选择是任意的，这里的任意当然是在一定框架内的，而不是指随便抛出一个什么值。在后续回合中，只有在*`validValue=nil`*时提议者才会建议一个新值，否则，会一直提议*validValue*，在提议者这里，*`validValue=nil`*表明在PROVOKE阶段这个值就被否定，从而要开始一个新的回合。除了这个*validValue*之外，PROPOSAL消息还包含*validRound*,这样其他进程就可以知道提议者认定*validValue*为一个可能被决定的值的最近一个回合。注意如果一个正确的提议者*p*在PROPOSAL消息中发送了*validValue*和*validRound*，这意味着**进程** *p*在回合*validRound*收到了PROPOSAL，当然这是它自己发出去的，以及相应的对*validValue*的*`2f+1`*个PREVOTE消息。

如果一个正确的进程*p*没有锁定值(*`lockedRound=−1`*)或者锁定了值*v* (*`lockedValue=v`*),并且通过一个外部的*valid*函数对*v*返回*true*,则*p*接受对*v*的提议，发送对*`id(v)`*的PREVOTE消息。

当提议对为*(v, vr≥0)*并且一个正确的进程*p*已经锁定了某个值时，如果*v*是一个更近的可能被决定的值，即*vr>lockedRound~p~*,或者 *lockedValue=v*，则*p*接受*v*。否则*p*就会拒绝这个提议，通过发送一个值为*nil*的PREVOTE的消息。*p*发送*nil*的另一种情况是*`timeoutPropose`*超时，并且在当前回合还没有发送过PREVOTE消息。

如果一个正确的进程收到了值为*v*的PROPOSAL消息和*2f+1*的对*id(v)*的PREVOTE消息，则发送带*id(v)*的PRECOMMIT消息，否则发送PRECOMMIT *nil*，同样的如果本回合尚未发送过PRECOMMIT，并且*`timeoutPrevote`*超时,也会发送PRECOMMIT *nil*。

一个正确的进程，如果收到某个回合*r*的对*v*的PROPOSAL消息，以及*2f+1*的对*id(v)*的PRECOMMIT消息，那么就可以对这个*v*做出决定。为了避免算法堵塞或者无限等待这个条件满足，设置了*`timeoutPrecommit`*，进程在本回合收到任意的*2f+1*的PRECOMMIT消息时触发。如果超时并且本进程还没有能够决定，则开始一个新的回合。如果做出了决定，就开始一个新的共识实例(下一个*height*)。

因为消息扩散通讯机制保证了导致*p*做出决定的PROPOSAL和 *2f+1*的PREVOTE消息肯定能被正确的进程收到，这些进程最终也会做出决定。

## IV.  PROOF OF TENDERMINT CONSENSUS ALGORITHM

**引理1.** *对所有的 f $\ge$ 0，任意两个至少有2f+1表决权的进程集合，至少有一个共同的元素是正确的进程*

**引理2.** *如果 f + 1 的正确进程在回合r~0~锁定了值v(lockedValue=v并且lockedRound=r~0~，那么在所有的回合r $\gt​$ r~0~,它们将发送id(v)或者nile的*PREVOTE*消息。*

[论文原文](https://arxiv.org/pdf/1807.04938.pdf)