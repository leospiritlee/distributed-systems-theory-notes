# 第8章 Paxos：两段提案如何守住唯一决定

**一句话**：`Paxos Made Simple` 用全局唯一且递增的 `proposal number` 与两轮多数派通信，让一个值一旦被选定就不可能被后来的提案覆盖。

**为什么需要它**：第七章说清了「相交」能防什么，却没说清「后来者怎么知道前面已经定了什么」。`Paxos` 的回答是把知识放进编号：新提案者必须先问多数派「你们答应过什么」，再据此决定自己提什么值。

**核心结构**：论文把参与者分成 proposers、acceptors、learners 三类角色（同一进程可兼多角）。Phase 1：`proposer` 选一个 `proposal number` n，向 `majority` 的 acceptors 发 `prepare request`；`acceptor` 若收到编号大于自己承诺过的所有编号，就承诺不再接受更低编号，并回报自己已接受过的最高编号提案。Phase 2：`proposer` 收到多数派响应后，发送 `accept request`；值取「已回报的最高编号提案的值」，若无人回报过才用自己的值。

**关键设计**：安全性由三件事合成——编号全局唯一且递增、`acceptor` 持久化自己的承诺与已接受值、任意两个多数派相交。论文原句：a proposal consists of a `proposal number` and a value. To prevent confusion, we require that different proposals have different numbers. 注意协议本身不需要固定 leader；工程落地之所以普遍用 `Multi-Paxos` 选出稳定 leader，是为了省掉每次重复的 Phase 1。

**拆解**：
### 1. Phase 1 到底在防什么
它不是在「申请许可」，而是在做一次考古：把之前可能已经被多数派接受、但还没人知道已选定的值挖出来。若跳过这一问，一个带着更高编号的新提案就可能写入另一个值，把已选定的决定覆盖掉。

### 2. 为什么值不能自己随便定
若多数派回报过任何已接受提案，新 `proposer` 必须沿用其中编号最高的那个值。这一步让「后来的编号」服务于「早先的决定」，而不是反过来。整条链因此只会收敛到一个值，即便多个 `proposer` 并发、消息乱序、节点重启。

### 3. 活锁与稳定 leader
两个 `proposer` 可能不断用更高编号互相打断对方的 Phase 1，谁都走不到 Phase 2。协议在 `safety` 上仍然正确，只是 `termination` 没保证。综合报告口径把 `Multi-Paxos` 的稳定 leader 视为工程答案：由同一个 `proposer` 连续提案，把两轮通信压成常态一轮。

### 4. 论文与系统之间还有距离
`Paxos Made Simple` 证明的是单决议的共识；真实系统要的是一串决定，于是有 `Multi-Paxos`、成员变更、快照、日志压缩与客户端线性一致读。综合报告口径提醒：算法不是系统，监控、快照、成员变更与升级才是工程量所在。

**今天怎么做**：读一段共识实现时，先找它的「承诺编号」存在哪个字段、是否落盘，再看 prepare 阶段有没有把已接受值带回来。这两个点对了，安全性才有根。

<div class="chapend" markdown="1">

**容易误解的地方**：`Paxos` 不是「投票超过半数就通过」——决定性的是编号单调与 Phase 1 的回填，不是票数本身。另一个误读是拿单决议 `Paxos` 与 `Raft` 比性能；综合报告口径明确说应与 `Multi-Paxos` 比。

**记住这一句**：Phase 1 问「你们答应过什么」，Phase 2 才说「那就定这个」；后来者替早先的决定盖章，而不是盖掉它。

</div>

<!-- chapend -->
