---
title: "比特币白皮书中英对照全文翻译（Bitcoin: A Peer-to-Peer Electronic Cash System）"
date: 2015-10-31T10:00:00+08:00
url: /2015/10/bitcoin-whitepaper-cn-en/
draft: false
tags: ["bitcoin", "blockchain", "paper", "security", "distributed-system"]
categories: ["tech"]
description: "比特币白皮书 Bitcoin: A Peer-to-Peer Electronic Cash System 的完整中英对照翻译，共 12 节，含工作量证明难度调整、默克尔树剪枝、简化支付验证，以及攻击者追上诚实链概率的推导与对应的 C 代码。"
---

这是比特币白皮书的完整中英对照翻译。原文 9 页，正文 12 节，加上 8 条参考文献。

体例：每段先列英文原文（引用块），紧接中文译文。公式、算法和概率表为便于阅读做了重排，图按原文内容重绘；专业术语保留英文并附中文，原文的引用编号 [1]–[8] 对应文末参考文献。

<!--more-->

## 论文信息

| 项目 | 内容 |
|------|------|
| 标题 | Bitcoin: A Peer-to-Peer Electronic Cash System |
| 作者 | Satoshi Nakamoto（中本聪） |
| 联系 | satoshin@gmx.com，www.bitcoin.org |
| 发布 | 2008-10-31，密码学邮件列表 metzdowd.com |
| 篇幅 | 9 页，正文 12 节，参考文献 8 条 |
| 核心机制 | 工作量证明、最长链、默克尔树、简化支付验证 |

---

## Abstract · 摘要

> A purely peer-to-peer version of electronic cash would allow online payments to be sent directly from one party to another without going through a financial institution. Digital signatures provide part of the solution, but the main benefits are lost if a trusted third party is still required to prevent double-spending. We propose a solution to the double-spending problem using a peer-to-peer network.

一种纯粹点对点形式的电子现金，可以让在线支付由一方直接发给另一方，而不必经过金融机构。数字签名提供了部分解决方案，但如果仍然需要一个可信第三方来防止**双重支付（double-spending）**，那么主要的好处就丧失了。我们提出一种用点对点网络解决双重支付问题的方案。

> The network timestamps transactions by hashing them into an ongoing chain of hash-based proof-of-work, forming a record that cannot be changed without redoing the proof-of-work. The longest chain not only serves as proof of the sequence of events witnessed, but proof that it came from the largest pool of CPU power.

网络通过把交易哈希进一条持续延伸的、基于哈希的<strong>工作量证明（proof-of-work）</strong>链，来为交易打时间戳，从而形成一条记录——除非重做全部工作量证明，否则这条记录无法被改动。最长链不仅证明了它所见证的事件序列，同时也证明了它来自最大的 CPU 算力池。

> As long as a majority of CPU power is controlled by nodes that are not cooperating to attack the network, they'll generate the longest chain and outpace attackers. The network itself requires minimal structure. Messages are broadcast on a best effort basis, and nodes can leave and rejoin the network at will, accepting the longest proof-of-work chain as proof of what happened while they were gone.

只要多数 CPU 算力掌握在不合谋攻击网络的节点手中，它们就会产生最长链，并把攻击者甩在后面。网络本身需要的结构极少：消息以<strong>尽力而为（best effort）</strong>的方式广播，节点可以随时离开和重新加入网络，并把最长的工作量证明链作为它们离开期间所发生事件的证明。

## 1 Introduction · 引言

> Commerce on the Internet has come to rely almost exclusively on financial institutions serving as trusted third parties to process electronic payments. While the system works well enough for most transactions, it still suffers from the inherent weaknesses of the trust based model. Completely non-reversible transactions are not really possible, since financial institutions cannot avoid mediating disputes. The cost of mediation increases transaction costs, limiting the minimum practical transaction size and cutting off the possibility for small casual transactions, and there is a broader cost in the loss of ability to make non-reversible payments for non-reversible services. With the possibility of reversal, the need for trust spreads. Merchants must be wary of their customers, hassling them for more information than they would otherwise need. A certain percentage of fraud is accepted as unavoidable. These costs and payment uncertainties can be avoided in person by using physical currency, but no mechanism exists to make payments over a communications channel without a trusted party.

互联网上的商业活动，已经几乎完全依赖作为<strong>可信第三方（trusted third party）</strong>的金融机构来处理电子支付。这套系统对大多数交易来说运转得足够好，但它仍然带有基于信任的模型所固有的弱点。**完全不可逆的交易实际上不可能存在**，因为金融机构无法回避对纠纷的调停。调停的成本推高了交易成本，抬高了实际可行的最小交易规模，也切断了小额临时交易的可能性；还有一项更广泛的代价——对于不可逆的服务，失去了使用不可逆支付的能力。一旦存在撤销的可能，对信任的需求就会扩散：商家必须提防自己的客户，向他们索要比原本所需更多的信息。一定比例的欺诈被当作不可避免而接受。这些成本和支付不确定性，在使用实物现金面对面交易时可以避免，但在通信信道上，不存在任何不依赖可信第三方的支付机制。

> What is needed is an electronic payment system based on cryptographic proof instead of trust, allowing any two willing parties to transact directly with each other without the need for a trusted third party. Transactions that are computationally impractical to reverse would protect sellers from fraud, and routine escrow mechanisms could easily be implemented to protect buyers. In this paper, we propose a solution to the double-spending problem using a peer-to-peer distributed timestamp server to generate computational proof of the chronological order of transactions. The system is secure as long as honest nodes collectively control more CPU power than any cooperating group of attacker nodes.

所需要的，是一套**基于密码学证明而非信任**的电子支付系统，它让任何两个自愿的当事方能够直接交易，而不需要可信第三方。在计算上无法实际逆转的交易，可以保护卖方免受欺诈；而常规的托管机制也可以很容易地实现，以保护买方。在本文中，我们提出一种使用**点对点分布式时间戳服务器**来为交易的时间先后顺序生成计算证明的方案，以此解决双重支付问题。只要诚实节点作为一个整体掌握的 CPU 算力超过任何合谋的攻击者节点群体，这套系统就是安全的。

## 2 Transactions · 交易

> We define an electronic coin as a chain of digital signatures. Each owner transfers the coin to the next by digitally signing a hash of the previous transaction and the public key of the next owner and adding these to the end of the coin. A payee can verify the signatures to verify the chain of ownership.

我们把一枚电子硬币定义为**一串数字签名**。每一位持有者把硬币转给下一位时，都对"上一笔交易的哈希"加上"下一位持有者的公钥"做数字签名，并把签名附加到硬币末尾。收款人可以验证这些签名，从而验证整条所有权链。

> The problem of course is the payee can't verify that one of the owners did not double-spend the coin. A common solution is to introduce a trusted central authority, or mint, that checks every transaction for double spending. After each transaction, the coin must be returned to the mint to issue a new coin, and only coins issued directly from the mint are trusted not to be double-spent. The problem with this solution is that the fate of the entire money system depends on the company running the mint, with every transaction having to go through them, just like a bank.

问题当然在于：收款人无法验证其中某位持有者是否已经把这枚硬币**重复花掉**过。常见的解决方案是引入一个可信的中心机构，也就是**铸币厂（mint）**，由它检查每一笔交易有没有双重支付。每笔交易之后，硬币必须交回铸币厂以发行一枚新币，只有直接由铸币厂发行的硬币才被信任为没有被重复花费。这个方案的问题在于：整套货币体系的命运取决于运营铸币厂的那家公司，而且每一笔交易都必须经过它——和银行一模一样。

> We need a way for the payee to know that the previous owners did not sign any earlier transactions. For our purposes, the earliest transaction is the one that counts, so we don't care about later attempts to double-spend. The only way to confirm the absence of a transaction is to be aware of all transactions. In the mint based model, the mint was aware of all transactions and decided which arrived first. To accomplish this without a trusted party, transactions must be publicly announced [1], and we need a system for participants to agree on a single history of the order in which they were received. The payee needs proof that at the time of each transaction, the majority of nodes agreed it was the first received.

我们需要一种办法，让收款人知道此前的持有者**没有签署过任何更早的交易**。在我们的目标下，最早的那笔交易才算数，所以后来的双重支付尝试我们并不关心。而确认"某笔交易不存在"的唯一方式，就是**知道所有交易**。在基于铸币厂的模型里，铸币厂知道所有交易，并决定哪一笔先到。要在没有可信第三方的前提下做到这一点，交易必须**公开广播** [1]，并且我们需要一套机制，让参与者就"接收顺序"这唯一一份历史达成一致。收款人需要证据来证明：在每一笔交易发生时，多数节点都同意它是最先收到的。

## 3 Timestamp Server · 时间戳服务器

> The solution we propose begins with a timestamp server. A timestamp server works by taking a hash of a block of items to be timestamped and widely publishing the hash, such as in a newspaper or Usenet post [2-5]. The timestamp proves that the data must have existed at the time, obviously, in order to get into the hash. Each timestamp includes the previous timestamp in its hash, forming a chain, with each additional timestamp reinforcing the ones before it.

我们提出的方案从**时间戳服务器**开始。时间戳服务器做的事情是：取一个待打时间戳的项目区块的哈希，并把该哈希广泛发布出去，比如登在报纸上或发在 Usenet 帖子里 [2-5]。这个时间戳证明了这些数据在当时必定已经存在——显然，只有这样它才可能进入该哈希。每个时间戳都把前一个时间戳包含进自己的哈希里，从而形成一条链，每一个新增的时间戳都在加固它之前的所有时间戳。

**图 1 重绘（左：时间戳链；右：所有权链）**

```
时间戳链                              所有权链

┌──────────────┐                  交易① ─────────────┐
│ Block        │                  ├ Owner 1's Public Key
│ Item Item …  │                  └ Hash ← Owner 0's Signature
│ Hash ←───────┼──┐
└──────────────┘  │               交易② ─────────────┐
┌──────────────┐  │               ├ Owner 2's Public Key
│ Block        │  │               └ Hash ← Owner 1's Signature
│ Item Item …  │  │                                  │
│ Hash         │←─┘ 含前一个 Hash                   │ 用 Owner 1's
└──────────────┘                                     │ Private Key 签名

                                  交易③ ─────────────┐
                                  ├ Owner 3's Public Key
                                  └ Hash ← Owner 2's Signature
                                                     │
                                       用 Owner 2's Private Key 签名
                                       Owner 3's Private Key（用于下一次签名）
```

## 4 Proof-of-Work · 工作量证明

> To implement a distributed timestamp server on a peer-to-peer basis, we will need to use a proof-of-work system similar to Adam Back's Hashcash [6], rather than newspaper or Usenet posts. The proof-of-work involves scanning for a value that when hashed, such as with SHA-256, the hash begins with a number of zero bits. The average work required is exponential in the number of zero bits required and can be verified by executing a single hash.

要在点对点的基础上实现一个分布式时间戳服务器，我们需要使用一种类似 Adam Back 的 Hashcash [6] 的工作量证明系统，而不是报纸或 Usenet 帖子。工作量证明的做法是：扫描寻找一个值，使得对它做哈希（例如用 SHA-256）之后，结果以若干个**零比特**开头。所需的平均工作量随所需零比特数的增加呈**指数级**增长，而验证只需要执行一次哈希。

> For our timestamp network, we implement the proof-of-work by incrementing a nonce in the block until a value is found that gives the block's hash the required zero bits. Once the CPU effort has been expended to make it satisfy the proof-of-work, the block cannot be changed without redoing the work. As later blocks are chained after it, the work to change the block would include redoing all the blocks after it.

对于我们的时间戳网络，工作量证明的实现方式是：不断递增区块中的 **nonce**，直到找到一个值使整个区块的哈希满足所要求的零比特数。一旦 CPU 已经为满足这个工作量证明付出过代价，这个区块就**无法在不重做该工作的情况下被改动**。而随着后续区块被链接在它之后，改动这个区块所需的工作还要包括重做它之后的所有区块。

> The proof-of-work also solves the problem of determining representation in majority decision making. If the majority were based on one-IP-address-one-vote, it could be subverted by anyone able to allocate many IPs. Proof-of-work is essentially one-CPU-one-vote. The majority decision is represented by the longest chain, which has the greatest proof-of-work effort invested in it. If a majority of CPU power is controlled by honest nodes, the honest chain will grow the fastest and outpace any competing chains. To modify a past block, an attacker would have to redo the proof-of-work of the block and all blocks after it and then catch up with and surpass the work of the honest nodes. We will show later that the probability of a slower attacker catching up diminishes exponentially as subsequent blocks are added.

工作量证明还解决了"多数决中如何确定代表权"的问题。如果多数决建立在**一个 IP 地址一票**的基础上，那么任何能够分配大量 IP 的人都能颠覆它。工作量证明本质上是**一台 CPU 一票**。多数决由最长链来代表——因为投入其中的工作量证明最多。如果多数 CPU 算力由诚实节点控制，诚实链将增长得最快，并把任何竞争链甩开。要修改一个过去的区块，攻击者必须重做该区块及之后所有区块的工作量证明，然后还要追上并超过诚实节点所做的工作。我们稍后会证明：随着后续区块不断加入，一个算力更慢的攻击者追上的概率会呈指数级衰减。

> To compensate for increasing hardware speed and varying interest in running nodes over time, the proof-of-work difficulty is determined by a moving average targeting an average number of blocks per hour. If they're generated too fast, the difficulty increases.

为了抵消硬件速度的提升、以及随时间变化的运行节点意愿，工作量证明的**难度**由一个移动平均值决定，其目标是让每小时产生的区块数维持在一个平均水准上。如果区块产生得太快，难度就会上升。

## 5 Network · 网络

> The steps to run the network are as follows:

运行网络的步骤如下：

> 1) New transactions are broadcast to all nodes.

1) 新交易广播给所有节点。

> 2) Each node collects new transactions into a block.

2) 每个节点把新交易收集到一个区块中。

> 3) Each node works on finding a difficult proof-of-work for its block.

3) 每个节点为自己的区块寻找一个满足难度要求的工作量证明。

> 4) When a node finds a proof-of-work, it broadcasts the block to all nodes.

4) 当一个节点找到工作量证明后，它把该区块广播给所有节点。

> 5) Nodes accept the block only if all transactions in it are valid and not already spent.

5) 节点只在区块中的所有交易都有效、且尚未被花费时才接受该区块。

> 6) Nodes express their acceptance of the block by working on creating the next block in the chain, using the hash of the accepted block as the previous hash.

6) 节点通过在链上创建下一个区块来表达对该区块的接受——以被接受区块的哈希作为"前一个哈希"。

> Nodes always consider the longest chain to be the correct one and will keep working on extending it. If two nodes broadcast different versions of the next block simultaneously, some nodes may receive one or the other first. In that case, they work on the first one they received, but save the other branch in case it becomes longer. The tie will be broken when the next proof-of-work is found and one branch becomes longer; the nodes that were working on the other branch will then switch to the longer one.

节点总是认为**最长链**是正确的，并持续在其上扩展。如果两个节点同时广播了下一个区块的不同版本，有些节点可能先收到其中一个。此时它们会先在自己先收到的那个上工作，但会保存另一个分支，以防它变得更长。当下一个工作量证明被找到、某个分支变得更长时，平局就被打破；原本在另一分支上工作的节点会切换到更长的那条。

**图 2 重绘（区块结构）**

```
┌───────────────────────┐      ┌───────────────────────┐
│ Block                 │      │ Block                 │
│ ├ Prev Hash           │←─────┤ ├ Prev Hash           │
│ ├ Nonce               │      │ ├ Nonce               │
│ └ Tx Tx …             │      │ └ Tx Tx …             │
└───────────────────────┘      └───────────────────────┘
```

> New transaction broadcasts do not necessarily need to reach all nodes. As long as they reach many nodes, they will get into a block before long. Block broadcasts are also tolerant of dropped messages. If a node does not receive a block, it will request it when it receives the next block and realizes it missed one.

新交易的广播不必到达所有节点。只要它们到达足够多的节点，不久就会被放进某个区块。区块广播同样能容忍丢消息：如果一个节点没有收到某个区块，它会在收到下一个区块、意识到自己漏掉了一个之后，去请求补发。

## 6 Incentive · 激励

> By convention, the first transaction in a block is a special transaction that starts a new coin owned by the creator of the block. This adds an incentive for nodes to support the network, and provides a way to initially distribute coins into circulation, since there is no central authority to issue them. The steady addition of a constant of amount of new coins is analogous to gold miners expending resources to add gold to circulation. In our case, it is CPU time and electricity that is expended.

按照约定，区块中的**第一笔交易**是一笔特殊交易，它创建一枚新硬币，归该区块的创建者所有。这为节点支持网络提供了激励，也提供了一种把硬币初始分发到流通中的方式——因为不存在发行它们的中心机构。新硬币以一个恒定的数量稳定增加，这类似于金矿矿工消耗资源把黄金投入流通。而在我们的情形里，被消耗的是 CPU 时间和电力。

> The incentive can also be funded with transaction fees. If the output value of a transaction is less than its input value, the difference is a transaction fee that is added to the incentive value of the block containing the transaction. Once a predetermined number of coins have entered circulation, the incentive can transition entirely to transaction fees and be completely inflation free.

激励也可以由**交易手续费**来提供。如果一笔交易的输出值小于其输入值，差额就是手续费，它会被加到包含该交易的区块的激励值上。一旦预先确定数量的硬币进入流通，激励就可以完全转为手续费，并且**完全不产生通货膨胀**。

> The incentive may help encourage nodes to stay honest. If a greedy attacker is able to assemble more CPU power than all the honest nodes, he would have to choose between using it to defraud people by stealing back his payments, or using it to generate new coins. He ought to find it more profitable to play by the rules, such rules that favour him with more new coins than everyone else combined, than to undermine the system and the validity of his own wealth.

激励也许有助于鼓励节点保持诚实。如果一个贪婪的攻击者能够集结比所有诚实节点更多的 CPU 算力，他将不得不在两种用途之间选择：用它来欺诈他人——把自己付出去的钱偷回来；或者用它来生成新硬币。他应当会发现，**按规则行事**——这些规则给他的新硬币比所有其他人加起来还多——比破坏这套系统及其自身财富的有效性更有利可图。

## 7 Reclaiming Disk Space · 回收磁盘空间

> Once the latest transaction in a coin is buried under enough blocks, the spent transactions before it can be discarded to save disk space. To facilitate this without breaking the block's hash, transactions are hashed in a Merkle Tree [7][2][5], with only the root included in the block's hash. Old blocks can then be compacted by stubbing off branches of the tree. The interior hashes do not need to be stored.

一旦一枚硬币的最新交易被足够多的区块埋住，它之前的那些已花费交易就可以被丢弃，以节省磁盘空间。为了在做到这一点时不破坏区块自身的哈希，交易被组织进一棵**默克尔树（Merkle Tree）** [7][2][5] 中做哈希，只有根哈希被包含在区块的哈希里。之后旧区块可以通过剪掉树的分支来压缩，中间层的哈希无需保存。

**图 3 重绘（默克尔树与剪枝）**

```
交易被哈希进一棵默克尔树            从区块中剪除 Tx0-2 之后

Block Header (Block Hash)          Block Header (Block Hash)
├ Prev Hash                        ├ Prev Hash
├ Nonce                            ├ Nonce
└ Root Hash                        └ Root Hash
      │                                  │
    Hash01 ── Hash23                   Hash23
      │          │                       │
   Hash0 Hash1  Hash2 Hash3           Hash2  Hash3
     │     │      │     │               │      │
    Tx0   Tx1    Tx2   Tx3             Tx2    Tx3
```

> A block header with no transactions would be about 80 bytes. If we suppose blocks are generated every 10 minutes, 80 bytes * 6 * 24 * 365 = 4.2MB per year. With computer systems typically selling with 2GB of RAM as of 2008, and Moore's Law predicting current growth of 1.2GB per year, storage should not be a problem even if the block headers must be kept in memory.

不含任何交易的区块头大约是 **80 字节**。假设每 10 分钟产生一个区块，那么 80 字节 × 6 × 24 × 365 = **每年 4.2 MB**。2008 年时计算机通常配备 2GB 内存，而摩尔定律预计当年以每年 1.2GB 的速度增长——即便必须把区块头全部保留在内存里，存储也不应该成为问题。

## 8 Simplified Payment Verification · 简化支付验证

> It is possible to verify payments without running a full network node. A user only needs to keep a copy of the block headers of the longest proof-of-work chain, which he can get by querying network nodes until he's convinced he has the longest chain, and obtain the Merkle branch linking the transaction to the block it's timestamped in. He can't check the transaction for himself, but by linking it to a place in the chain, he can see that a network node has accepted it, and blocks added after it further confirm the network has accepted it.

不运行一个完整的网络节点，也有可能验证支付。用户只需要保存最长工作量证明链的**区块头副本**——他可以通过向网络节点查询来获得，直到确信自己拿到的是最长链——并获取把该交易与其被打上时间戳的区块联系起来的**默克尔分支**。他无法自己检查这笔交易，但通过把它链接到链上的某个位置，他可以看到某个网络节点已经接受了它；而在此之后新增的区块则进一步确认网络已经接受了它。

> As such, the verification is reliable as long as honest nodes control the network, but is more vulnerable if the network is overpowered by an attacker. While network nodes can verify transactions for themselves, the simplified method can be fooled by an attacker's fabricated transactions for as long as the attacker can continue to overpower the network. One strategy to protect against this would be to accept alerts from network nodes when they detect an invalid block, prompting the user's software to download the full block and alerted transactions to confirm the inconsistency. Businesses that receive frequent payments will probably still want to run their own nodes for more independent security and quicker verification.

因此，只要诚实节点控制着网络，这种验证就是可靠的；但如果网络被攻击者的算力压倒，它就更脆弱。网络节点可以自行验证交易，而简化方法只要在攻击者能持续压倒网络期间，就可能被其伪造的交易欺骗。防范这一点的一种策略是：接受网络节点在检测到无效区块时发出的警报，促使用户软件下载完整区块和相关联的被警报交易，以确证这处不一致。接收频繁付款的商家，多半仍然会希望运行自己的节点，以获得更独立的安全性和更快的验证。

**图 4 重绘（最长工作量证明链上的默克尔分支）**

```
Block Header ─┐   Block Header ─┐   Block Header
Merkle Root   │   Merkle Root   │   Merkle Root
Prev Hash     │   Prev Hash     │   Prev Hash
Nonce         │   Nonce         │   Nonce
              │                 │        │
              └─────────────────┘     Hash23 ── Hash2   ← 默克尔分支
                                            │
                                           Tx3          ← 待验证交易
```

## 9 Combining and Splitting Value · 价值的合并与拆分

> Although it would be possible to handle coins individually, it would be unwieldy to make a separate transaction for every cent in a transfer. To allow value to be split and combined, transactions contain multiple inputs and outputs. Normally there will be either a single input from a larger previous transaction or multiple inputs combining smaller amounts, and at most two outputs: one for the payment, and one returning the change, if any, back to the sender.

尽管逐枚处理硬币是可行的，但为一笔转账中的每一分钱都单独做一笔交易会非常笨重。为了让价值可以被拆分和合并，交易包含**多个输入和多个输出**。通常，要么是来自一笔更大金额的前序交易的单个输入，要么是合并若干较小金额的多个输入；而输出最多两个：一个是付款，另一个是找回给发送方的**找零**（如果有的话）。

> It should be noted that fan-out, where a transaction depends on several transactions, and those transactions depend on many more, is not a problem here. There is never the need to extract a complete standalone copy of a transaction's history.

需要注意的是，<strong>扇出（fan-out）</strong>在这里不成问题——即一笔交易依赖若干笔交易，而那些交易又依赖更多交易。这里从来不需要抽出一份完整独立的交易历史副本。

**图 5 重绘（传统隐私模型与新隐私模型）**

```
传统隐私模型                        新隐私模型

  Identities ──┐                     Identities ──┐
               │                                  │
  Transactions ┼──→ 可信第三方          Transactions ┼──→ 公开
               │                                  │
  Counterparty ┘
               │
             Public

  身份与交易都被藏在可信第三方背后      身份与交易都可见，
  （Public 看不到）                     靠公钥匿名来获得隐私
```

（原文图注：传统隐私模型把交易信息限制在参与方与可信第三方之间；新隐私模型把交易本身公开，但让身份保持匿名。）

## 10 Privacy · 隐私

> The traditional banking model achieves a level of privacy by limiting access to information to the parties involved and the trusted third party. The necessity to announce all transactions publicly precludes this method, but privacy can still be maintained by breaking the flow of information in another place: by keeping public keys anonymous. The public can see that someone is sending an amount to someone else, but without information linking the transaction to anyone. This is similar to the level of information released by stock exchanges, where the time and size of individual trades, the "tape", is made public, but without telling who the parties were.

传统银行模型通过把信息访问权限限制在交易参与方和可信第三方之间，来达到一定程度的隐私。而**公开宣告所有交易**这一必要做法排除了这种手段；不过隐私仍然可以通过在另一处切断信息流来保持：**让公钥保持匿名**。公众可以看到某人向另一个人发送了一定金额，但没有把该交易与任何人关联起来的信息。这类似于证券交易所披露信息的程度：每笔交易的时间和规模——也就是"**行情带（tape）**"——是公开的，但不会说明交易双方是谁。

> As an additional firewall, a new key pair should be used for each transaction to keep them from being linked to a common owner. Some linking is still unavoidable with multi-input transactions, which necessarily reveal that their inputs were owned by the same owner. The risk is that if the owner of a key is revealed, linking could reveal other transactions that belonged to the same owner.

作为一道额外的防火墙，**每笔交易都应使用一对新密钥**，以防止它们被关联到同一个所有者。在多输入交易中，某种程度的关联仍然不可避免——这类交易必然暴露出它们的输入属于同一个所有者。风险在于：一旦某个密钥的所有者身份被揭露，关联分析就可能暴露出属于同一所有者的其他交易。

## 11 Calculations · 计算

> We consider the scenario of an attacker trying to generate an alternate chain faster than the honest chain. Even if this is accomplished, it does not throw the system open to arbitrary changes, such as creating value out of thin air or taking money that never belonged to the attacker. Nodes are not going to accept an invalid transaction as payment, and honest nodes will never accept a block containing them. An attacker can only try to change one of his own transactions to take back money he recently spent.

我们考虑这样一个场景：攻击者试图以比诚实链更快的速度生成一条替代链。即使他做到了这一点，也不会让系统对任意改动敞开大门——比如凭空创造价值，或者拿走从来不属于攻击者的钱。节点不会接受一笔无效交易作为支付，诚实节点也永远不会接受包含这类交易的区块。攻击者能做的，只是试图修改他自己的一笔交易，把自己最近花出去的钱拿回来。

> The race between the honest chain and an attacker chain can be characterized as a Binomial Random Walk. The success event is the honest chain being extended by one block, increasing its lead by +1, and the failure event is the attacker's chain being extended by one block, reducing the gap by -1.

诚实链与攻击者链之间的竞赛，可以用一个<strong>二项式随机游走（Binomial Random Walk）</strong>来刻画。成功事件是诚实链延长一个区块，把领先优势增加 +1；失败事件是攻击者的链延长一个区块，把差距减少 -1。

> The probability of an attacker catching up from a given deficit is analogous to a Gambler's Ruin problem. Suppose a gambler with unlimited credit starts at a deficit and plays potentially an infinite number of trials to try to reach breakeven. We can calculate the probability he ever reaches breakeven, or that an attacker ever catches up with the honest chain, as follows [8]:

攻击者从给定落后量追上来的概率，类似于一个**赌徒破产问题（Gambler's Ruin）**。假设一个拥有无限信用的赌徒从落后开始，并且可以进行潜在无限次的尝试以试图回到不亏不赚。我们可以计算他最终回到盈亏平衡的概率，也就是攻击者最终追上诚实链的概率，如下 [8]：

```
p  = 诚实节点找到下一个区块的概率
q  = 攻击者找到下一个区块的概率
qz = 攻击者从落后 z 个区块处最终追上的概率

qz = 1           若 p ≤ q
qz = (q / p)^z   若 p > q
```

> Given our assumption that p > q, the probability drops exponentially as the number of blocks the attacker has to catch up with increases. With the odds against him, if he doesn't make a lucky lunge forward early on, his chances become vanishingly small as he falls further behind.

在我们假定 p > q 的前提下，随着攻击者需要追赶的区块数增加，这个概率**呈指数级下降**。由于胜算对他不利，如果他没有在早期幸运地猛冲一大步，那么随着他落后得越来越多，机会就变得微乎其微。

> We now consider how long the recipient of a new transaction needs to wait before being sufficiently certain the sender can't change the transaction. We assume the sender is an attacker who wants to make the recipient believe he paid him for a while, then switch it to pay back to himself after some time has passed. The receiver will be alerted when that happens, but the sender hopes it will be too late.

现在我们来考虑，一笔新交易的收款人需要等待多久，才能足够确信发送方无法更改这笔交易。我们假定发送方是一个攻击者，他希望让收款人在一段时间内相信他已经付款，然后在一段时间之后再把它改为付给自己。到那时收款人会收到警报，但发送方希望那已经太晚了。

> The receiver generates a new key pair and gives the public key to the sender shortly before signing. This prevents the sender from preparing a chain of blocks ahead of time by working on it continuously until he is lucky enough to get far enough ahead, then executing the transaction at that moment. Once the transaction is sent, the dishonest sender starts working in secret on a parallel chain containing an alternate version of his transaction.

收款人在签名前不久生成一对新密钥，并把公钥交给发送方。这可以防止发送方提前准备一条区块链——通过持续工作直到幸运地领先得足够远，然后在那一刻才执行这笔交易。一旦交易发出，不诚实的发送方就开始秘密地在一条**并行链**上工作，那条链里包含他这笔交易的一个替代版本。

> The recipient waits until the transaction has been added to a block and z blocks have been linked after it. He doesn't know the exact amount of progress the attacker has made, but assuming the honest blocks took the average expected time per block, the attacker's potential progress will be a Poisson distribution with expected value:

收款人会一直等到该交易被加进一个区块、并且其后又链接了 z 个区块。他并不知道攻击者已经取得了多少进度，但假设诚实区块所花的时间就是平均预期每区块时间，那么攻击者可能的进度将服从一个**泊松分布（Poisson distribution）**，其期望值为：

```
λ = z · (q / p)
```

> To get the probability the attacker could still catch up now, we multiply the Poisson density for each amount of progress he could have made by the probability he could catch up from that point:

要得到攻击者此刻仍能追上的概率，我们把"他可能取得的每一种进度"的泊松密度，乘以"他从那一点能追上"的概率：

```
P = Σ(k=0..∞) [ λ^k · e^(−λ) / k! ] × ( (q/p)^(z−k) 若 k ≤ z  否则 1 )
```

> Rearranging to avoid summing the infinite tail of the distribution...

重排一下，以避免对分布的无限尾部求和……

```
P = 1 − Σ(k=0..z) [ λ^k · e^(−λ) / k! ] × ( 1 − (q/p)^(z−k) )
```

> Converting to C code...

转换成 C 代码……

```c
#include <math.h>
double AttackerSuccessProbability(double q, int z)
{
    double p = 1.0 - q;
    double lambda = z * (q / p);
    double sum = 1.0;
    int i, k;
    for (k = 0; k <= z; k++)
    {
        double poisson = exp(-lambda);
        for (i = 1; i <= k; i++)
            poisson *= lambda / i;
        sum -= poisson * (1 - pow(q / p, z - k));
    }
    return sum;
}
```

> Running some results, we can see the probability drop off exponentially with z.

跑一些结果出来，可以看到该概率随 z 呈指数级衰减。

**q = 0.1（攻击者掌握 10% 算力）**

| z（落后区块数） | P（追上的概率） |
|---:|---:|
| 0 | 1.0000000 |
| 1 | 0.2045873 |
| 2 | 0.0509779 |
| 3 | 0.0131722 |
| 4 | 0.0034552 |
| 5 | 0.0009137 |
| 6 | 0.0002428 |
| 7 | 0.0000647 |
| 8 | 0.0000173 |
| 9 | 0.0000046 |
| 10 | 0.0000012 |

**q = 0.3（攻击者掌握 30% 算力）**

| z（落后区块数） | P（追上的概率） |
|---:|---:|
| 0 | 1.0000000 |
| 5 | 0.1773523 |
| 10 | 0.0416605 |
| 15 | 0.0101008 |
| 20 | 0.0024804 |
| 25 | 0.0006132 |
| 30 | 0.0001522 |
| 35 | 0.0000379 |
| 40 | 0.0000095 |
| 45 | 0.0000024 |
| 50 | 0.0000006 |

> Solving for P less than 0.1%...

求解使 P 小于 0.1% 的条件……

| q（攻击者算力占比） | z（需要的确认数） |
|---:|---:|
| 0.10 | 5 |
| 0.15 | 8 |
| 0.20 | 11 |
| 0.25 | 15 |
| 0.30 | 24 |
| 0.35 | 41 |
| 0.40 | 89 |
| 0.45 | 340 |

## 12 Conclusion · 结论

> We have proposed a system for electronic transactions without relying on trust. We started with the usual framework of coins made from digital signatures, which provides strong control of ownership, but is incomplete without a way to prevent double-spending. To solve this, we proposed a peer-to-peer network using proof-of-work to record a public history of transactions that quickly becomes computationally impractical for an attacker to change if honest nodes control a majority of CPU power. The network is robust in its unstructured simplicity. Nodes work all at once with little coordination. They do not need to be identified, since messages are not routed to any particular place and only need to be delivered on a best effort basis. Nodes can leave and rejoin the network at will, accepting the proof-of-work chain as proof of what happened while they were gone. They vote with their CPU power, expressing their acceptance of valid blocks by working on extending them and rejecting invalid blocks by refusing to work on them. Any needed rules and incentives can be enforced with this consensus mechanism.

我们提出了一套不依赖信任的电子交易系统。我们从"由数字签名构成的硬币"这一惯常框架出发，它提供了对所有权强有力的控制，但如果没有防止双重支付的手段就不完整。为解决这一点，我们提出一个使用工作量证明来记录交易公开历史的点对点网络；只要诚实节点控制着多数 CPU 算力，攻击者想要改动这些记录，在计算上很快就会变得不切实际。这套网络以其**无结构的简洁**而稳健：节点们几乎不需要协调就同时工作；它们不需要被识别身份，因为消息并不路由到任何特定位置，只需尽力投递即可；节点可以随意离开和重新加入网络，并把工作量证明链作为它们离开期间所发生事情的证明。它们用 CPU 算力投票——通过在有效区块上继续扩展来表达接受，通过拒绝在其上工作来表达对无效区块的拒绝。任何必要的规则与激励，都可以由这套共识机制来强制执行。

---

## References · 参考文献

> [1] W. Dai, "b-money," http://www.weidai.com/bmoney.txt, 1998.

[1] W. Dai，"b-money"，http://www.weidai.com/bmoney.txt，1998。

> [2] H. Massias, X.S. Avila, and J.-J. Quisquater, "Design of a secure timestamping service with minimal trust requirements," In 20th Symposium on Information Theory in the Benelux, May 1999.

[2] H. Massias、X.S. Avila、J.-J. Quisquater，"Design of a secure timestamping service with minimal trust requirements"，第 20 届比荷卢信息论研讨会，1999 年 5 月。

> [3] S. Haber, W.S. Stornetta, "How to time-stamp a digital document," In Journal of Cryptology, vol 3, no 2, pages 99-111, 1991.

[3] S. Haber、W.S. Stornetta，"How to time-stamp a digital document"，《密码学杂志》第 3 卷第 2 期，第 99-111 页，1991。

> [4] D. Bayer, S. Haber, W.S. Stornetta, "Improving the efficiency and reliability of digital time-stamping," In Sequences II: Methods in Communication, Security and Computer Science, pages 329-334, 1993.

[4] D. Bayer、S. Haber、W.S. Stornetta，"Improving the efficiency and reliability of digital time-stamping"，载于 Sequences II: Methods in Communication, Security and Computer Science，第 329-334 页，1993。

> [5] S. Haber, W.S. Stornetta, "Secure names for bit-strings," In Proceedings of the 4th ACM Conference on Computer and Communications Security, pages 28-35, April 1997.

[5] S. Haber、W.S. Stornetta，"Secure names for bit-strings"，第 4 届 ACM 计算机与通信安全会议论文集，第 28-35 页，1997 年 4 月。

> [6] A. Back, "Hashcash - a denial of service counter-measure," http://www.hashcash.org/papers/hashcash.pdf, 2002.

[6] A. Back，"Hashcash - a denial of service counter-measure"，http://www.hashcash.org/papers/hashcash.pdf，2002。

> [7] R.C. Merkle, "Protocols for public key cryptosystems," In Proc. 1980 Symposium on Security and Privacy, IEEE Computer Society, pages 122-133, April 1980.

[7] R.C. Merkle，"Protocols for public key cryptosystems"，1980 年 IEEE 计算机学会安全与隐私研讨会论文集，第 122-133 页，1980 年 4 月。

> [8] W. Feller, "An introduction to probability theory and its applications," 1957.

[8] W. Feller，"An introduction to probability theory and its applications"（概率论及其应用导论），1957。
