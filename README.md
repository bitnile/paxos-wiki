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
### 阶段2a *Accept*
如果 Proposer 收到了足够来自法定人数的 Acceptor 的 Promise，它需要给这个提案设定一个值 v。 If any Acceptors had previously accepted any proposal, then they'll have sent their values to the Proposer, who now must set the value of its proposal, v, to the value associated with the highest proposal number reported by the Acceptors, let's call it z.如果没有 Acceptor 接受过大于这个值得提案。那么这个 Proposer 会选择它最开始想要的提议的提案的值。用 x 表示。

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

## 译者注

### 拜占庭将军问题

假设有9位将军投票，其中1名叛徒。8名忠诚的将军中出现了4人投进攻，4人投撤离的情况。这时候叛徒可能故意给4名投进攻的将领送信表示投票进攻，而给4名投撤离的将领送信表示投撤离。这样一来在4名投进攻的将领看来，投票结果是5人投进攻，从而发起进攻；而在4名投撤离的将军看来则是5人投撤离。这样各支军队的一致协同就遭到了破坏。

由于将军之间需要通过信使通讯，叛变将军可能通过伪造信件来以其他将军的身份发送假投票。而即使在保证所有将军忠诚的情况下，也不能排除信使被敌人截杀，甚至被敌人间谍替换等情况。因此很难通过保证人员可靠性及通讯可靠性来解决问题。

参考：[拜占庭将军问题](https://zh.wikipedia.org/wiki/%E6%8B%9C%E5%8D%A0%E5%BA%AD%E5%B0%86%E5%86%9B%E9%97%AE%E9%A2%98)
