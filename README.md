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

## 译者注

### 拜占庭将军问题

假设有9位将军投票，其中1名叛徒。8名忠诚的将军中出现了4人投进攻，4人投撤离的情况。这时候叛徒可能故意给4名投进攻的将领送信表示投票进攻，而给4名投撤离的将领送信表示投撤离。这样一来在4名投进攻的将领看来，投票结果是5人投进攻，从而发起进攻；而在4名投撤离的将军看来则是5人投撤离。这样各支军队的一致协同就遭到了破坏。

由于将军之间需要通过信使通讯，叛变将军可能通过伪造信件来以其他将军的身份发送假投票。而即使在保证所有将军忠诚的情况下，也不能排除信使被敌人截杀，甚至被敌人间谍替换等情况。因此很难通过保证人员可靠性及通讯可靠性来解决问题。

参考：[拜占庭将军问题](https://zh.wikipedia.org/wiki/%E6%8B%9C%E5%8D%A0%E5%BA%AD%E5%B0%86%E5%86%9B%E9%97%AE%E9%A2%98)
