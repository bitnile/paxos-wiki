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

## 译者注

### 拜占庭将军问题

假设有9位将军投票，其中1名叛徒。8名忠诚的将军中出现了4人投进攻，4人投撤离的情况。这时候叛徒可能故意给4名投进攻的将领送信表示投票进攻，而给4名投撤离的将领送信表示投撤离。这样一来在4名投进攻的将领看来，投票结果是5人投进攻，从而发起进攻；而在4名投撤离的将军看来则是5人投撤离。这样各支军队的一致协同就遭到了破坏。

由于将军之间需要通过信使通讯，叛变将军可能通过伪造信件来以其他将军的身份发送假投票。而即使在保证所有将军忠诚的情况下，也不能排除信使被敌人截杀，甚至被敌人间谍替换等情况。因此很难通过保证人员可靠性及通讯可靠性来解决问题。

参考：[拜占庭将军问题](https://zh.wikipedia.org/wiki/%E6%8B%9C%E5%8D%A0%E5%BA%AD%E5%B0%86%E5%86%9B%E9%97%AE%E9%A2%98)
