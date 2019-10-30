# paxos-wiki

Paxos算法是莱斯利·兰伯特（英语：Leslie Lamport，LaTeX中的“La”）于1990年提出的一种基于消息传递且具有高度容错特性的一致性算法。本文是对维基百科上[Paxos算法文献](https://en.wikipedia.org/wiki/Paxos_(computer_science))的翻译。

## 假设条件

为了简化 Paxos 的介绍, 先明确以下假设和定义。

### Processors

- Processors 以任意速度运行。
- Processors 可能会遇到故障。
- 具有稳定存储的 Processors 在失败后可以重新加入协议（遵循崩溃-恢复故障模型）。question
- 不会发生拜占庭式故障。question

### Network

- 每个 Processor 可以将消息发送到其它任何 Processor。
- 消息是异步发送的，可能要花很长时间才能发出。
- 消息可能会丢失、乱序或重复。
- 消息在发送过程中没有损坏（即没发生拜占庭式故障）。

### Processor的数量

通常，使用 n=2F+1 个 Processor 可以在 F 个 Processor 同时发生故障时依然保持共识算法的正常运行：换句话说，非故障的 Processor 数量必须大于故障的 Processor 数量。However, using reconfiguration, a protocol may be employed which survives any number of total failures as long as no more than F fail simultaneously[citation needed].question

## 角色

在 Paxos 中，Processor 的行为取决于它的角色：client、acceptor、proposer、learner 和 leader。在典型实现中，同一个 Processor 可以扮演多个角色，这不会影响协议的正确性——合并角色通常能改善延迟并减少消息数量。
- Client: 客户端向分布式系统发出请求，然后等待响应。例如，对分布式文件服务器中文件的写请求。
- Acceptor (Voters): The Acceptors act as the fault-tolerant "memory" of the protocol. Acceptors are collected into groups called Quorums. Any message sent to an Acceptor must be sent to a Quorum of Acceptors. Any message received from an Acceptor is ignored unless a copy is received from each Acceptor in a Quorum.question
- Proposer: 向 Acceptors 提出 Client 的请求，并在冲突发生的时候，起到冲突调节的作用。
- Learner: 在协议中充当备份的角色。一旦 Client 的请求被 Acceptors 同意了，Learner 将执行请求并将响应发送给 Client。为了提高可用性，可以添加多个 Learner。
- Leader: Paxos requires a distinguished Proposer (called the leader) to make progress. Many processes may believe they are leaders, but the protocol only guarantees progress if one of them is eventually chosen. If two processes believe they are leaders, they may stall the protocol by continuously proposing conflicting updates. However, the safety properties are still preserved in that case.question

### 法定人数

法定人数通过确保那些存活的 processor 仍保留结果来保证 Paxos 的一致性。法定人数是 Acceptors 的子集，因此任意两个子集（即任意两组法定人数）至少有一个成员是共享的。通常，法定人数是 Acceptors 中的多数派，例如，给定一组 Acceptors {A，B，C，D}，法定人数可以是任意三个 Acceptors: {A，B，C}，{A，C，D}，{A，B，D} ，{B，C，D}。同样，可以将任意权重分配给 Acceptors，此时，法定人数可以定义为权重大于所有 Acceptors 总权重一半的任意子集。

### 提案编号和内容

Each attempt to define an agreed value v is performed with proposals which may or may not be accepted by Acceptors. Each proposal is uniquely numbered for a given Proposer. So, e.g., each proposal may be of the form (n, v), where n is the unique identifier of the proposal and v is the actual proposed value. The value corresponding to a numbered proposal can be computed as part of running the Paxos protocol, but need not be.question

## Basic Paxos
Basic Paxos 是 Paxos 协议族中最基本的一种协议。Basic Paxos 的每一个实例（或 “操作”）都决定了一个输出值。这个协议分几个轮次进行。一个成功的轮次有两个阶段：阶段1（分为 a 和 b 两个部分）和阶段2（分为 a 和 b 两个部分）。参见下面对各阶段的描述。我们假设一个异步模型，一个 Processor 可能在其中一个阶段而另一个 Processor 可能在另一个阶段。

### 阶段1
#### 阶段1a：*Prepare*
一个 Proposer 创建了一条消息，我们把这条消息称 “Prepare” 消息，并确认一个数 n，注意，n 不是要提议的值，and maybe agreed on question，而只是一个数字，它由 Proposer 唯一标识此初始消息（发送给 Acceptors）。而 n 必须比这个 Proposer 在之前创建的任何 *Prepare* 消息的标识号都要大。接着，它发送这个带有 n 的 *Prepare* 消息给 [Acceptors](#角色) 的 [法定人数](#法定人数)。注意，*Prepare* 消息只包含了 数字 n （也就是说，它没有包含提案的值，提案的值通常用 v 来表示）。Proposer 决定哪些 Acceptor 在法定人数中。如果无法与 Acceptors 中的多数派进行通信，则 Proposer 不会进行 Paxos。

#### 阶段1b：*Promise*
任何一个 Acceptor 都在等待接收来自任意一个 Proposer 的 *Prepare* 消息，Acceptor 必须查看刚刚收到的 *Prepare* 消息中的确认数字 n。这里有两种情况：

* 如果 n 大于 Acceptor 在之前从任何一个 Proposer 接收到的消息标识号，则 Acceptor 必须向 Proposer 返回一条消息，我们称这个消息为 “Promise”，用来忽略将来所有标识号小于 n 的提案。如果 Acceptor 在过去的某个时候接受了提案，那么他必须在对 Proposer 的回复中包含先前的提案标识号 m 和相应的接收值 w。

* 否则（指的是 n 不大于 Acceptor 在之前从任何一个 Proposer 接收到的消息标识号），Acceptor 可以忽略这个提案。在这种情况下，Paxos 不会进行。但是，为了优化起见，发送一个拒绝响应（[Nack](https://en.wikipedia.org/wiki/NAK_(protocol_message))）告诉 Proposer 它将停止尝试与 n 建立共识。

### 阶段2
#### 阶段2a *Accept*
如果 Proposer 收到了足够来自法定人数的 Acceptor 的 Promise，它需要给这个提案设定一个值 v。 ~~If any Acceptors had previously accepted any proposal, then they'll have sent their values to the Proposer, who now must set the value of its proposal, v, to the value associated with the highest proposal number reported by the Acceptors, let's call it z.~~ 如果任何 Acceptors 以前接受过任何提案，那么它们会将提案内容发送给 Proposer，Proposer 现在必须将其提案的内容 v 设置为与 Acceptors 报告的最高的提案编号关联的内容 z。~~如果没有 Acceptor 接受过大于这个值得提案。那么这个 Proposer 会选择它最开始想要的提议的提案的值。用 x 表示。~~ 如果到目前为止没有任何一个 Acceptor 接受了提案，那么 Proposer 可以选择它最初想要的提案 x。

Proposer 发送一个带有提案值 v 和提案数字 n 的 *Accept* 消息（n，v）给具有法定人数的 Acceptor（n 和之前发送给 Acceptor 的 *Prepare* 消息中的标识号是相同的）。所以，这个*Accept* 消息又可以表示为 （n，v=z）或者在之前没有 Acceptor 接收值得情况下，（n，v=x） 

这个 *Accept* 消息应该被翻译为 “请求”，表示为“请接受该提议！”

#### 阶段2b *Accepted*
如果一个 Acceptor 接收了一个来自 Proposer 的 *Accept* 消息 （n，v），当且仅当它尚未承诺（在Paxos协议的 阶段1b 中）仅考虑标识号大于 n 的提议时，它才必须接受它。

如果 Acceptor 尚未承诺（在 阶段1b 中）仅考虑标识号大于 n 的提案，则应将（刚收到的  *Accept* 消息）的值 v 注册为（协议的）接收值，并发送一个 *Accepted* 消息给 Proposer 和每个 Learner（通常是 Proposer 本身）</br>
否则，它将忽略这个 *Accept* 消息或请求

注意，一个 Acceptor 可以接收多个提案。所以可能会发生以下情况：当另一个 Proposer 不知道要确定的新值时，使用一个更高的标识号 n 来开始一个新的轮次。在这种情况下，即使 Acceptor 在早先接收了另外一个标识号，接受者仍然会承诺并且稍后接收新的（标识号更大的）提案。在某些故障情况下，这些提案甚至可能会有不同的值。但是 Paxos 协议会保证 Acceptor 最终会在一个值中达成共识。

### 轮次失败的情况
当多个 Proposer 发送冲突的 *Prepare* 消息，或者 Proposer 未能接收到法定人数的承诺或接收回复，该轮次会失败。在这些情况下，会开始另一个带有更高的提案标识号的轮次。

### Paxos 能应用在领导者选举中
请注意，当 Acceptor 接收了一个请求，他也会承认 Proposer 的领导。因此，Paxos 也能够用来选举一个节点集群的领导者。


### Basic Paxos 的图形表示
下面的流程图表示 Basic Paxos 协议应用的几种情况。这几种情况会说明 Basic Paxos 协议如何应对分布式系统中的一些组件 question 的故障。

注意：在首次提出提案时， *Promise*消息 中返回的值为 “null”（因为在这个轮次之前，没有 Acceptor 接受过该值）

#### Basic Paxos 的成功情况
在下图中，有一个 client，一个 Proposer， 三个 Acceptor（即法定人数为 3）和两个 Learner（由2条垂直线表示）。该图表示第一轮成功的情况（即网络中没有进程失败）。

```
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |<---------X--X--X       |  |  Promise(1,{Va,Vb,Vc})
   |         X--------->|->|->|       |  |  Accept!(1,V)
   |         |<---------X--X--X------>|->|  Accepted(1,V)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
```

question

#### Basic Paxos 的错误情况
The simplest error cases are the failure of an Acceptor (when a Quorum of Acceptors remains alive) and failure of a redundant Learner. In these cases, the protocol requires no "recovery" (i.e. it still succeeds): no additional rounds or messages are required, as shown below (in the next two diagrams/cases).

#### Acceptor 失败的 Basic Paxos
在下图中，法定人数中的其中一个 Acceptor 失败，所以法定认识变成了 22，在这种情况下，Basic Paxos 仍然可以成功

```
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |          |  |  !       |  |  !! FAIL !!
   |         |<---------X--X          |  |  Promise(1,{Va, Vb, null})
   |         X--------->|->|          |  |  Accept!(1,V)
   |         |<---------X--X--------->|->|  Accepted(1,V)
   |<---------------------------------X--X  Response
   |         |          |  |          |  |
```

#### 多个 Learner 失败的 Basic Paxos
在这种情况下，会有一个（或多个） Learn 失败，但是 Basic Paxos 协议仍然能成功
```
Client Proposer         Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |<---------X--X--X       |  |  Promise(1,{Va,Vb,Vc})
   |         X--------->|->|->|       |  |  Accept!(1,V)
   |         |<---------X--X--X------>|->|  Accepted(1,V)
   |         |          |  |  |       |  !  !! FAIL !!
   |<---------------------------------X     Response
   |         |          |  |  |       |
```

#### 一个 Proposer 失败的 Basic Paxos
In this case, a Proposer fails after proposing a value, but before the agreement is reached. Specifically, it fails in the middle of the Accept message, so only one Acceptor of the Quorum receives the value. Meanwhile, a new Leader (a Proposer) is elected (but this is not shown in detail). Note that there are 2 rounds in this case (rounds proceed vertically, from the top to the bottom).

```
Client  Proposer        Acceptor     Learner
   |      |             |  |  |       |  |
   X----->|             |  |  |       |  |  Request
   |      X------------>|->|->|       |  |  Prepare(1)
   |      |<------------X--X--X       |  |  Promise(1,{Va, Vb, Vc})
   |      |             |  |  |       |  |
   |      |             |  |  |       |  |  !! Leader fails during broadcast !!
   |      X------------>|  |  |       |  |  Accept!(1,V)
   |      !             |  |  |       |  |
   |         |          |  |  |       |  |  !! NEW LEADER !!
   |         X--------->|->|->|       |  |  Prepare(2)
   |         |<---------X--X--X       |  |  Promise(2,{V, null, null})
   |         X--------->|->|->|       |  |  Accept!(2,V)
   |         |<---------X--X--X------>|->|  Accepted(2,V)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
```

#### 多个 Proposer 冲突的 Basic Paxos
如果有多个 Proposer 认为自身是 Leader 的时候，这种情况是最复杂的。举个例子，当前的 Leader 可能失败后恢复，但是此时其他的 Proposer 已经选举了新 Leader。而恢复后的 Leader 仍不知道选举了新 leader，而试图开启一个和当前的 Leader 冲突的轮次。在下图中，展示了 4 种未成功的轮次，但其实有可能一直失败下去。

```
Client   Leader         Acceptor     Learner
   |      |             |  |  |       |  |
   X----->|             |  |  |       |  |  Request
   |      X------------>|->|->|       |  |  Prepare(1)
   |      |<------------X--X--X       |  |  Promise(1,{null,null,null})
   |      !             |  |  |       |  |  !! LEADER FAILS
   |         |          |  |  |       |  |  !! NEW LEADER (knows last number was 1)
   |         X--------->|->|->|       |  |  Prepare(2)
   |         |<---------X--X--X       |  |  Promise(2,{null,null,null})
   |      |  |          |  |  |       |  |  !! OLD LEADER recovers
   |      |  |          |  |  |       |  |  !! OLD LEADER tries 2, denied
   |      X------------>|->|->|       |  |  Prepare(2)
   |      |<------------X--X--X       |  |  Nack(2)
   |      |  |          |  |  |       |  |  !! OLD LEADER tries 3
   |      X------------>|->|->|       |  |  Prepare(3)
   |      |<------------X--X--X       |  |  Promise(3,{null,null,null})
   |      |  |          |  |  |       |  |  !! NEW LEADER proposes, denied
   |      |  X--------->|->|->|       |  |  Accept!(2,Va)
   |      |  |<---------X--X--X       |  |  Nack(3)
   |      |  |          |  |  |       |  |  !! NEW LEADER tries 4
   |      |  X--------->|->|->|       |  |  Prepare(4)
   |      |  |<---------X--X--X       |  |  Promise(4,{null,null,null})
   |      |  |          |  |  |       |  |  !! OLD LEADER proposes, denied
   |      X------------>|->|->|       |  |  Accept!(3,Vb)
   |      |<------------X--X--X       |  |  Nack(4)
   |      |  |          |  |  |       |  |  ... and so on ...
```

 ## Multi-Paxos

A typical deployment of Paxos requires a continuous stream of agreed values acting as commands to a distributed state machine. If each command is the result of a single instance of the Basic Paxos protocol, a significant amount of overhead would result.

如果 Leader 比较稳定，就没必要再进行阶段一了。因此，对于将来具有相同领导者的协议的实例，可以跳过阶段一。

To achieve this, the round number I is included along with each value which is incremented in each round by the same Leader. Multi-Paxos reduces the failure-free message delay (proposal to learning) from 4 delays to 2 delays.

### Multi-Paxos 中消息流的图形表示

#### 正常情况下的 Multi-Paxos

在下图中，只显示了 Basic-Paxos 协议的一个实例，其中有一个初始领导者（一个提议者）。注意，Multi-Paxos 由 Basic-Paxos 协议的几个实例组成。question

```
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  | --- First Request ---
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(N)
   |         |<---------X--X--X       |  |  Promise(N,I,{Va,Vb,Vc})
   |         X--------->|->|->|       |  |  Accept!(N,I,V)
   |         |<---------X--X--X------>|->|  Accepted(N,I,V)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
```

where V = last of (Va, Vb, Vc).

#### 可忽略阶段一的 Multi-Paxos

在这种情况下，Basic-Paxos 协议的子序列实例（由I+1表示）使用相同的领导者，所以包含了 Prepare 和 Promise 子阶段的阶段一将被忽略。注意，Leader 应当是稳定的，即不应崩溃或更换。

```
Client   Proposer       Acceptor     Learner
   |         |          |  |  |       |  |  --- Following Requests ---
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Accept!(N,I+1,W)
   |         |<---------X--X--X------>|->|  Accepted(N,I+1,W)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
```

#### 角色合并的 Multi-Paxos question

Multi-Paxos 的一个常见部署是将 Proposers、Acceptors 和 Learners 的角色合并为 Servers。所以，最后只有 Clients 和 Servers。下图代表 Basic-Paxos 协议的第一个“实例”，即当 Proposer、Acceptor 和 Learner 的角色合并为单个角色（称为 Server）时。

```
Client      Servers
   |         |  |  | --- First Request ---
   X-------->|  |  |  Request
   |         X->|->|  Prepare(N)
   |         |<-X--X  Promise(N, I, {Va, Vb})
   |         X->|->|  Accept!(N, I, Vn)
   |         X<>X<>X  Accepted(N, I)
   |<--------X  |  |  Response
   |         |  |  |
```

#### 当角色合并且 Leader 稳定时的 Multi-Paxos

如果 Leader 相同，则可以跳过阶段一。

```
Client      Servers
   X-------->|  |  |  Request
   |         X->|->|  Accept!(N,I+1,W)
   |         X<>X<>X  Accepted(N,I+1)
   |<--------X  |  |  Response
   |         |  |  |
```

## 译者注

### 拜占庭将军问题

假设有9位将军投票，其中1名叛徒。8名忠诚的将军中出现了4人投进攻，4人投撤离的情况。这时候叛徒可能故意给4名投进攻的将领送信表示投票进攻，而给4名投撤离的将领送信表示投撤离。这样一来在4名投进攻的将领看来，投票结果是5人投进攻，从而发起进攻；而在4名投撤离的将军看来则是5人投撤离。这样各支军队的一致协同就遭到了破坏。

由于将军之间需要通过信使通讯，叛变将军可能通过伪造信件来以其他将军的身份发送假投票。而即使在保证所有将军忠诚的情况下，也不能排除信使被敌人截杀，甚至被敌人间谍替换等情况。因此很难通过保证人员可靠性及通讯可靠性来解决问题。

参考：[拜占庭将军问题](https://zh.wikipedia.org/wiki/%E6%8B%9C%E5%8D%A0%E5%BA%AD%E5%B0%86%E5%86%9B%E9%97%AE%E9%A2%98)
