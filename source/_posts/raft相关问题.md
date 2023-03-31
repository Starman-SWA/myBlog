---
title: raft相关问题
date: 2023-03-29 19:13:47
tags:
- 分布式系统
---

# Raft的一致性复制过程

<!-- more -->

1. 客户端向leader发送请求
2. leader向自己的日志增加一条新entry，然后向所有的follower并行发送AppendEntries RPC来复制这条新entry。AppendEntries RPC可以一次性发送多条entry。
3. follower通过AppendEntries RPC中的prevLogIndex和prevLogTerm查找本地是否拥有这条entry的前一条entry，如果有，则删除其后所有冲突的entry，然后添加新entry，向leader发送success响应
4. 当leader接收到来自过半数follower的success响应时，则将这条entry标记为commited，并在随后发送的AppendEntries RPC中将最新的leaderCommit发送给所有follower
5. follower将自己的commitIndex更新为leaderCommit和本地最新index的最小值
6. 所有服务器（包括leader、candidate和follower）定期将本地commitIndex对应的entry应用（apply）到本地状态机上
7. leader将entry应用到本地状态机后，响应客户端

# 当一个leader崩溃时，candidate当选leader的条件

- candidate的日志至少要比过半数的followr更up-to-date，才可以获得过半数投票当选leader：
- ![image-20230328174100075](image-20230328174100075.png)
- 原因如下：当candidate满足这个条件是，它会包含所有已提交的entry：
- ![image-20230328174442878](image-20230328174442878.png)
- up-to-date的比较规则：先比较term，term相同再比较index

# 在一致性复制的不同阶段leader崩溃时，Raft分别如何处理？

## 数据到达leader，但未复制到大多数follower

- 这条entry还没有被提交，剩余的节点会选出一个包含所有已提交日志的leader
- 当leader包含这条未提交的entry时，会对其进行间接提交
  - 如果客户端还在原有的崩溃节点等待，当崩溃节点恢复时，会接收到新leader提交的这条entry，并应用到状态机，从而响应客户端
  - 如果客户端向新的leader再次发送了这条请求，由于请求中含有客户端指定的唯一ID，尽管Raft提交了两条重复的请求，但只会向状态机应用一次
  - 由于这条指令被间接提交后会被应用到每个可用节点的server层，因此**客户端必须重试**。如果客户端不重试，则会有一条错误的指令被执行。也就是说，**一旦用户在客户端发送了一条请求，则这条请求不能够被撤回**。
  - **简而言之，此时的一致性由客户端指定的唯一操作ID保证**

## 数据到达leader，成功复制到大多数follower，但leader未接收到过半数的响应

- 同上

## 数据到达leader，成功接收到了过半数的follower响应，但未更新commitIndex

- 只要未apply，就不会响应客户端。由于commitIndex未更新，因此视为没有commit，同上

## 数据到达leader，成功复制并且更新了commitIndex，但未在本地apply且未将commitIndex发送给大多数follower

- 新leader没有更新commitIndex，视为未commit。本机未apply，未响应客户端。因此不会有问题，同上

## 数据到达leader，成功复制并且更新了commitIndex，已在本地apply，但未将commitIndex发送给大多数follower

- 新leader没有更新commitIndex，视为未commit，但新leader的日志中包含这条entry，会对其间接提交。
- 原本发送请求的客户端接收了正确的响应。间接提交后所有节点的server层也能够正确执行这条指令。

## 数据到达leader，成功复制并且更新了commitIndex，未在本地apply，但已将commitIndex发送给大多数follower

- 新leader会apply这条指令。当客户端在新leader重试时，通过操作ID可以发现这条指令已经被执行了，直接返回success。如果客户端还在原leader等待，当原leader重启时也会apply这条指令并响应客户端

# 网络分区问题

- 假设小分区和大分区均存在leader，均可以接收客户端请求。小分区当中的entry，由于无法append到大多数，因此永远不会被commit；大分区当中的所有entry会被commit和apply。相应地，大分区的客户端请求可以被正确执行，而小分区的客户端请求会一直超时重试。
- 当小分区与大分区合并时，term更高的那个leader会成为整个集群的leader，其日志被复制到整个集群。
- **大分区的leader一定是term更高的**，因为网络分区后，小分区中的票数不足，无法选举产生term更大的leader。所以新leader一定是大分区的leader，不会出现大分区的日志被小分区覆盖，导致已执行的客户端请求丢失的情况。
- 当分区合并后，小分区的leader变为follower，那些一直在超时重试的请求会返回失败。

## CAP理论

- 在多节点系统当中，CAP三个性质只能同时满足两个
  - Consistence一致性：数据在多个副本之间保持一致的特性
  - Availability可用性：服务一直可用（但是不保证获取的数据为最新数据）
  - Network Partitioning分区容错性：网络分区故障时仍然能对外提供满足一致性和可用性的服务，除非整个网络都发生了故障
- raft的网络分区满足了AP特性：当网络分区时，如果要保证服务可用性，则副本之间有可能存在不一致。反之，如果对于副本的一致性有要求，例如金融系统，在网络分区情况下服务是不可用的，必须等到网络分区恢复。

# Raft与一致性

- 有一种说法是，**raft与一致性无关，raft只实现了共识，一致性针对的是服务，取决于上层应用。**然而我认为一致性是服务和raft共同保证的。我的6.824 KV数据库的强一致性由**服务层的唯一操作ID**和**raft层对读操作的共识**共同实现。
- 如果只考虑raft，那么raft也必须为强一致性提供保证，即避免脏读。一种方法是我的读操作共识方法，另一种方法见下一小节。

## 一致性类型

- 最终一致性：写入过程中，客户端可能读到旧值，也可能读到新值

- 线性一致性（强一致）：当其中一个客户端读到了新的数据，所有的客户端都立即能够获取新数据
- **我的raft实现对于读请求必须达成共识，因此实现了强一致性**。因为读请求之前的写请求对应的entry都被commit了，杜绝了“在写的过程中读”这回事

# 如果不想为读操作添加entry，应该怎么避免脏读

- 我的KV数据库实现为所有的读操作添加entry，等到该entry达成共识时才响应客户端，会带来较高的延迟

- 如果不为读操作添加entry，在不引入其他额外措施的情况下会发生脏读（论文第8节最后一段）：客户端读到了旧leader的值，而这个值已经被新leader修改了，但是由于网络分区等原因，旧leader还没有意识到他已经不是leader了。

- 为了防止脏读，需要引入下列两个措施：

  1. leader必须拥有之前所有被提交的日志，leader完整性可以保证leader最终会拥有所有已提交的日志，但在任期刚开始时可能还没拥有。此时需要发送一个no-op的空entry，**以强制提交leader拥有的之前任期未被提交的日志**

  2. leader在响应读请求之前必须确保它仍然是leader，可以通过发送心跳，接收大多数节点的响应来检查

## 为什么需要提交no-op

- 如果leader在其任期开始时拥有之前任期的未被提交的日志，那么这条日志最终会被提交，然而leader在此时向客户端返回的是这条日志未提交的状态，产生了脏读
- 提交no-op可以强制提交这条日志，使leader更新到最新的状态
- **在我的6.824 KV数据库实现当中，由于采用了读共识，因此不需要提交no-op**

# 为什么leader不能直接提交之前任期的entry

- 论文中的Figure 8进行了解释：
- ![Figure 8](https://baixiangcpp.oss-cn-shanghai.aliyuncs.com/blog/raft-paper/raft-8.png)
- 简而言之，leader只能在提交本任期日志的过程中，通过日志匹配规则“间接提交”之前任期的日志。如果直接提交之前任期的日志，则这条日志有可能被覆盖。
- 图中方框中的数字表示日志的任期，带黑框框的节点为leader。并且需要注意每个leader的任期：(c)中的S1为term=4，(d)中的S5为term=5
- 如果允许leader直接提交之前任期的entry，则第一步：(c)中S1向大多数节点复制并提交了term=2的日志，然后崩溃了。第二步：(d)中S5的日志比大多数节点都要up-to-date，被成功选为leader，然后用自己的蓝色日志覆盖了黄色日志。
- 如果不允许直接提交，则第一步：(c)提交了term=4的红色日志，并且间接提交了term=2的黄色日志，变为(e)。此时，S5的日志没有比大多数节点up-to-date，**无法被选为leader**。

# Raft试题

- [Raft 作者亲自出的 Raft 面试题！（含答案） - 掘金 (juejin.cn)](https://juejin.cn/post/6902221596765716488)

## 1

### 问题

- 下面的每张图都显示了一台 Raft 服务器上可能存储的日志（日志内容未显示，只显示日志的 index 和任期号）。考虑每份日志都是独立的，下面的日志可能发生在 Raft 中吗？如果不能，请解释原因。
- a：![img](https://pic3.zhimg.com/80/v2-dc0978de8bdd645cf355a7ab6ec82ce2_720w.webp)
- b：![img](https://pic2.zhimg.com/80/v2-0b5aa71e876ff9b9a0235c9d3c84a0b1_720w.webp)
- c：![img](https://pic3.zhimg.com/80/v2-de6ef98b2ad6c0d3ee2b89bfb983fa2a_720w.webp)
- d：![img](https://pic2.zhimg.com/80/v2-a64e57a90f28c62652807f10483b365d_720w.webp)

### 解答

- a：不可能，一旦服务器接收到term=3的日志，会拒绝所有term<3的请求，不可能再添加term=2的日志
- b：可能，日志的term号单调递增，一个term可能有多个日志
- c：可能，一个term并不一定有新的日志
- d：不可能，服务器在将一个entry附加到日志当中的时候，会保证前一个entry与leader匹配，因此不可能在某一个旧的index上有缺失的entry

## 2

### 问题

- 下图显示了一个 5 台服务器集群中的日志（日志内容未显示）。哪些日志记录可以安全地应用到状态机？请解释你的答案。
- ![img](https://pic3.zhimg.com/80/v2-1540c1ed479e0f358ebbc42692770876_720w.webp)

### 解答

- ~~index=1、2、3、4、5。因为这些日志被复制到了大多数（3个以上）节点~~
- 除此之外，我们还要考虑哪些日志会被新的leader截断
- 首先考虑哪些节点可以成为leader。根据up-to-date的规则，先比较term，再比较index，因此S2、S5都有可能成为leader。因为S2比S3、S4、S5更up-to-date，S5比S3、S4更up-to-date。当S2或S5成为leader时，index>=3的日志有可能被截断。因此，可以被安全应用的日志只有Index=1、2

## 3

### 问题

- 考虑下图，它显示了一个 6 台服务器集群中的日志，此时刚刚选出任期 7 的新 Leader（日志内容未显示，只显示日志的 index 和任期号）。对于图中每一个 Follower，给定的日志是否可能在一个正常运行的 Raft 系统中存在？如果是，请描述该情况如何发生的；如果不是，解释为什么。
- ![img](https://pic2.zhimg.com/80/v2-47103e12ffe306582d5df0ae2f851721_720w.webp)

### 解答

- (a)：不可能，当前leader有3个term为3的日志，而term为3的leader将日志发送给(a)时会把之前term为2的日志截断
- (b)：不可能，因为term为5的日志之前的日志与现leader矛盾，原因同(a)
- (c)：可能，~~这个节点与term为6的leader在同一个分区~~。这个节点可能是term为6的leader，并且在一个单独的分区
- (d)：不可能，term号不可能递减
- (e)：可能，~~term为1的leader向该节点多发送了4个entry，但与现leader不在一个网络分区，然后崩溃了~~。这个节点可能是term为1的leader，并且在一个单独的分区

## 4

### 问题

- 假设硬件或软件错误破坏了 Leader 为某个特定 Follower 存储的 `nextIndex` 值。这是否会影响系统的安全？请简要解释你的答案。

### 解答

- 不会影响，因为有prevLogIndex和prevLogTerm来匹配follower和leader中相同的entry。nextIndex的变动不会影响匹配的正确性，只要匹配失败，nextIndex就会继续减小，直至成功。极端情况下，当nextIndex减小到0的时候，leader也能够将整个日志发送给follower

## 5

### 问题

- 假设你实现了 Raft，并将它部署在同一个数据中心的所有服务器上。现在假设你要将系统部署到分布在世界各地的不同数据中心的每台服务器，与单数据中心版本相比，多数据中心的 Raft 需要做哪些更改？为什么？

### 解答

- 选举超时和心跳超时需要延长时间，否则会因为网络传输时间变长导致频繁的领导变更

## 6

### 问题

- 每个 Follower 都在其磁盘上存储了 3 个信息：当前任期（`currentTerm`）、最近的投票（`votedFor`）、以及所有接受的日志记录（`log[]`）。

- a. 假设 Follower 崩溃了，并且当它重启时，它最近的投票信息已丢失。该 Follower 重新加入集群是否安全（假设未对算法做任何修改）？解释一下你的答案。

- b. 现在，假设崩溃期间 Follower 的日志被截断（truncated）了，日志丢失了最后的一些记录。该 Follower 重新加入集群是否安全（假设未对算法做任何修改）？解释一下你的答案。

### 解答

- a：不安全，它有可能会在同一个term之内将票投给另外一个candidate，造成同一个term有两个leader，且这两个leader无法通过互相识别term来退位
- ~~b：安全，尽管这个follower会丢失一些已经被提交的日志，但是这些日志会被leader重新发送，最终它会重新拥有所有日志。并且由于这个follower的日志不是最up-to-date的，它也不可能通过成为新的leader来覆盖其他节点的日志。因此是安全的~~
- b：不安全。截断会使得一个已提交的日志没有被存储在多数派上，此时，没有这条日志的节点就能够满足up-to-date条件而被选举为leader，并将其他follower中的这条日志都覆盖掉，导致一个index被提交了不同的值

## 7

### 问题

- 如[视频](https://link.zhihu.com/?target=https%3A//www.youtube.com/watch%3Fv%3DYbZ3zDzDnrw)中所述，即使其它服务器认为 Leader 崩溃并选出了新的 Leader 后，（老的）Leader 依然可能继续运行。新的 Leader 将与集群中的多数派联系并更新它们的任期，因此，老的 Leader 将在与多数派中的任何一台服务器通信后立即下台。然而，在此期间，它也可以继续充当 Leader，并向尚未被新 Leader 联系到的 Follower 发出请求；此外，客户端可以继续向老的 Leader 发送请求。我们知道，在选举结束后，老的 Leader 不能提交（commit）任何**新的**日志记录，因为这样做需要联系选举多数派中的至少一台服务器。但是，老的 Leader 是否有可能执行一个成功 `AppendEntries RPC`，从而完成在选举开始前收到的旧日志记录的提交？如果可以，请解释这种情况是如何发生的，并讨论这是否会给 Raft 协议带来问题。如果不能发生这种情况，请说明原因。

### 解答

- 有可能，在新的leader选举出来之前，老的leader向多数派append了日志并提交，然后还没有来得及将新的commitIndex发送给follower就断开连接了。但是并不会给Raft协议带来问题，因为多数派当中只有日志最up-to-date的节点才可以被选为新的leader，因此，新leader一定包含这条被提交日志。

## 8

### 问题

- 在配置变更过程中，如果当前 Leader 不在 C-new 中，一旦 C-new 的日志记录被提交，它就会下台。然而，这意味着有一段时间，Leader 不属于它所领导的集群（Leader 上存储的当前配置条目是 C-new，C-new 不包括 Leader）。假设修改协议，如果 C-new 不包含 Leader，则使 Leader 在其日志存储了 C-new 时就立即下台。这种方法可能发生的最坏情况是什么？

### 解答

- 成员变更还没看，挖坑待填

# 其他问题

## 为什么follower需要随机超时

- 为了防止多个follower同时变为candidate，造成split vote
