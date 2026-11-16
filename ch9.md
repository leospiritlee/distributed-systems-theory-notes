# 第9章 Raft：把共识拆成三个可理解的子问题

**一句话**：`Raft` 用强 leader、随机超时选举与连续日志，把 `consensus` 拆成 `leader election`、`log replication`、`safety` 三块，以可理解性为设计目标换取状态空间的大幅收缩。

**为什么需要它**：`Paxos` 的正确性证明漂亮，但把单决议协议变成可维护的 `replicated log` 系统，中间要补的工程细节非常多。`Raft` 论文开宗明义：In Search of an Understandable Consensus Algorithm，它把「让人能改对」当成与「算法正确」同等重要的目标。

**核心结构**：论文原句——`Raft` is a `consensus` algorithm for managing a `replicated log`. It produces a result equivalent to (multi-)`Paxos`, and it is as efficient as `Paxos`, but its structure is different from `Paxos`。它把关键元素分为 `leader election`、`log replication`、`safety`，并「enforces a stronger degree of coherency to reduce the number of states that must be considered」。节点只有三种状态，写请求全部经 leader，日志条目按序复制，收到多数派确认后才推进 `commit index` 并应用到 `state machine`。

**关键设计**：`term` 是 `Raft` 的逻辑时钟。论文原句：Terms act as a `logical clock` [14] in `Raft`, and they allow servers to detect obsolete information such as stale leaders. 每个 `term` 至多一个 leader，随机化选举超时减少撞车；日志强制连续、禁止乱序提交，这换来简单，也放弃了 `Multi-Paxos` 允许的乱序提交吞吐。综合报告口径把这一条列为二者最实质的差别。

**拆解**：
### 1. 强 leader 简化了什么
所有写都从 leader 进入，日志只从 leader 流向 follower，于是「日志该按什么顺序」不再需要协商，只需要保证 leader 完整。代价是 leader 成为可用性与吞吐的焦点，选举期间无法推进，这正是第四章 `liveness` 只在部分同步下的具体体现。

### 2. term 如何识别过期信息
每次选举产生新 `term`，节点间交换消息时带上编号；编号小的一方立刻退位或拒绝。它不度量真实时间，只提供单调的版本号——第二章的 `logical clock` 在这里第二次出现，用途从「排序事件」变成「识别陈旧 leader」。

### 3. 日志连续与安全性
`Raft` 要求被提交的日志条目此前所有条目也一致，靠「条目含 `term` + 索引」的匹配规则让冲突日志被覆盖。禁止乱序提交使状态机只需按一个下标推进；综合报告口径指出 `Multi-Paxos` 允许乱序提交、吞吐更高但复杂得多。

### 4. 客户端那一侧还有一段
论文专门讨论了客户端如何找到 leader、如何支持 linearizable semantics。综合报告口径归纳过一类真实事故：所谓「一致性读」只是返回「自认为是 leader」的节点本地状态，而 `Raft` 本身不保证 leader 排他，于是出现陈旧读。补法是 `ReadIndex`、`lease` 或 `fencing token`。

**今天怎么做**：若必须自己实现共识，先问团队能不能在三年后还改对这套代码；能选现成系统（`etcd`、`Consul`、`ZooKeeper`）就不要自己写。若要读源码，按选举、复制、安全三块分三次读，别从日志匹配规则开始。

<div class="chapend" markdown="1">

**容易误解的地方**：`Raft` 与 `Paxos` 的比较对象应是 `Multi-Paxos` 而非单决议 `Paxos`，二者容错与性能等价，差别在工程复杂度。另一个误读是把 `term` 当成时间戳；它只是单调递增的任期编号，不提供跨节点真实时间。

**记住这一句**：`Raft` 的创新不在更强的保证，而在把状态空间收紧到人类能推理的大小。

</div>

<!-- chapend -->

# 分布式系统理论精读
