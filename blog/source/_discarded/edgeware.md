---
title: Edgeware简介/白皮书中文翻译
categories:
- BlockChain
- Substrate
- Application
tags:
copyright: true
date: 2019-05-21 15:55:33
---



> 笔者刚接触区块链不久，鉴于水平有限，翻译不到位之处还请留言指出，或通过邮件联系，谢谢。

# 简介

Edgeware旨在通过采用完全不同的体系结构来解决可伸缩性问题。Edgeware的token持有者可以使用链上投票来升级网络，然后节点自动下载运行时的新版本。关键决策是在链上做出的，创建一个具有更低协调开销和透明流程的系统，用于决定改进。

<!-- more -->

# 技术细节

Edgeware是一种智能合约区块链，它编译到客户机运行时，是一段WebAssembly (Wasm)代码，可以在Wasm虚拟机中构建和运行或执行。无论哪种方式，在编译Edgeware原生二进制文件时，它都包含一个Wasm虚拟机，可用来执行从网络下载的客户机运行时版本。客户端runtime面向网络代码和Parity Substrate提供的其他组件进行交互。Substrate包括libp2p网络、PBFT协商一致、以及块验证和最终确定性。最后，客户端只负责从网络中下载、执行和验证块。

从这里就可以看出，Edgeware的链从性质上应该与Substrate链是一样的，包括Extrinsics等细节。

## 模块

模块用Rust编写，编译成Wasm，并作为客户端runtime的一部分进行链接。它们定义了Edgeware逻辑的核心，并且能够进行不可预知的图灵完备的计算，因此对模块代码进行仔细的审计和测试是至关重要的。特别重要的是，没有客户端能通过申请一个比blocktime执行时间更长的extrinsic来破坏链。否则，验证器将无法及时调用`author_block`来生成有效块。

Edgeware包含了这些模块，并且不仅仅是这些。其他模块使用的功能模块以及计划中的升级(如`fees`模块)是不包括的：

| Extrinsic |                   描述                    |
| :-------: | :---------------------------------------: |
| Balances  |              为账户保存余额               |
| Consensus |         一系列权威节点被允许出块          |
|   Aura    |         一系列权威节点被允许出块          |
|  Grandpa  |    确保面对错误的验证者还能最终确定块     |
|  Staking  |           掌控成为验证者所需的            |
|  Session  |            在会话间转换验证者             |
| Contract  |    允许可计算gas的智能合约的创建与执行    |
| Treasury  |              线上财富的治理               |
| Signaling | 允许token拥有者使用离线信号来完成提议共识 |
| Democarcy | 允许token持有者通过直接民主的形式通过提案 |
|  Council  |       允许由投票选举的议会通过提案        |
| Identity  |            链上标识的简单原语             |



## 合约

Edgeware智能合约可以用任何兼容wasm的高级语言编写。已经有一些经过良好测试的工具链用于编译C、c++和rust到Wasm，并且正在通过Assemblyscript项目将javascript子集编译到Wasm。随着其他区块链和web基础设施项目进一步开发wasm的工具链， Edgeware还将受益于安全性、可移植性和表达性方面的改进。

合约的功能与它们在Ethereum或其他EVM兼容的区块链上的功能非常相似。任何具有EDG余额的用户都可以部署合约，合约也可以调用其他合约。以EDG计价的gas必须预先购买，并限制计算量。消息调用消耗gas直到交易的限制，任何多余的都将被退还。如果使用了所有gas，计算将终止，并进行回退。

合约还可以通过一组有限的白名单接口与其他Edgeware模块交互，特别是查询`Identity` 和 `Council` 模块中的账户。智能合约系统的其他改进，如并行交易和状态转移，预计将在链推出后跟进。

## 共识

Edgeware推出了一个指定的 Sybil-resistance机械原理的证明。可转换的验证人组被分配来执行和密封块。验证人最初是使用一种集合投币算法来选择的。

为了避免无利害关系的问题，Edgeware 使用 GRANDPA最终确定算法来实现 block finality。链上治理使Edgeware能够优雅地实现改进的共识机制。

5秒的块时间允许足够的网络延迟，同时仍然保持高吞吐量。

## 权益

个人验证者必须将他们的股份担保三个月。Edgeware 维护一组验证人，这些验证人可以在一段时间内保持其领导地位。

一个Edgeware账户可以将他们的余额委托给指定的权益证明(NPoS)系统中的其他验证者，以获得块奖励的一部分。块奖励首先在所有活跃的验证人之间平均分配，然后按照支持该验证人的资产的比例进行分配。因此，Edgeware账户在避免过度集中的同时，会被激励去寻找值得信赖的验证人。

## 资产和账户

资产在基于账户的系统中保持，由ed25519 椭圆曲线的派生密钥来保证。账户和余额转移必须满足这些限制条件，作为跨链的全局变量来维护：

* TotalIssuance：系统内所有的权益数量；
* ExistentialDeposit：保持账户存在的最小资产；
* TransferFee：交易的手续费；
* CreationFee：创建账户的手续费。至少和ReclaimRebate 一样大；
* FreeBalance：给定账户的自由资产，用来决定在合约执行环境中的资产。当这个资产低于'ExistentialDeposit'时，'当前账户'会被删除；
* ReservedBalance：给定账户的可以被削减的外部预留资产，但最后被削减；
* TransactionBaseFee：交易费用；基础部分；
* TransactionByteFee：交易费用；每字节部分；

## Edgeware Token - EDG

最初有50亿(50亿) EDG token被铸造出来，可以被小数点后18位整除。最初，通货膨胀被设置为158 EDG每块。这意味着第一年的通货膨胀率大约为997,220,160 EDG，或者略低于20%。

EDG铸币总量将逐年保持不变，导致通货膨胀率呈反通货膨胀趋势，第二年的年通货膨胀率大约下降到16.6%。此外，全系统投票可能会进一步增加或减少通货膨胀。

关于Edgeware的第一批提议之一是将通货膨胀与特定的参与率或安全率联系起来，通货膨胀上下浮动以达到预期的总持股量。指定的提案将通过修改通货膨胀率的runtime升级进行表决和执行。

在Edgeware上，EDG 有多种功能。EDG 授权持有人同时拥有下注和表决权，以及将该权利委托给另一个帐户的能力。EDG 还用于支付Wasm 智能合约模块中的状态更改。

## 互操作性

Edgeware 有一个通过Polkadot 实现跨链互操作性的具体路线图，Polkadot 是Parity 正在用Substrate 开发的跨链桥接网络。Polkadot 将连接区块链，包括Edgeware和以太坊，允许资产被锁在一个网络上，在另一个网络上管理。

从长远来看，用户将能够同时运行以太网和Edgeware dapps ——也许可以将以太网用作高风险金融应用的稳定网络，将Edgeware 用作需要高吞吐量和一级身份的应用的渐进网络（PW）。

# 网络启动

将通过向以太网持有者锁定 Edgeware 代币来启动 Eegeware 网络，锁定将在积极参与者之间协调激励措施。锁定投放（lockdrop) 是一种空投，锁定是指代币持有者在一个网络上用智能合约锁定他们的代币一段可变化的时期。在我们的例子中，以太持有者可以通过 ***singnal*** 来锁定他们的代币，从而表示他们希望参与锁定或锁定的时间短至三个月，长至一年，而锁定时间的延长则对应于立即收到的按比例增加的 Edgeware 代币。

出于以下原因，此过程是在 PoS 网络中分发代币的更公平和更安全的方法：

* 非零的机会成本：为了参与 lockdrop ，个人放弃了 ETH 在 lockdrop 期间的机会成本，例如能够以复合贷款的方式放贷。这不是一个长期 ETH 持有者的问题！由于这个原因，长期持有 ETH 最符合EDG lockdrop 。EDG 代币的生成将允许代币持有者参与分配给 Edgeware 参与者的所有权利，成为验证人或对网络提案进行投票。
* 下行保护：在 lockdrop 结束时，参与者将分别访问两个生产力代币 ETH和EDG 。Edgeware是一种新的链，在发生恶意攻击或利用时，lockdrop 参与者仍然保留他们的 ETH 。
* 容易参与：将 ETH 发送到我们 lockdrop 合约的单个交易允许一个人得到他或她自己的 EDG 代币。任何帐户都可以从硬件或软件钱包(例如Trezor、Metamask等)执行此操作。此外，任何 ETH 持有人都可以参加。
* 经济安全：在新的PoS 链上加强安全性。Ethereum 从最初的群众销售和 PoW 奖励启动宁静版本。同样的，Edgeware 也可以利用已经广泛分布的以太坊持有者来引导Edgeware链的经济安全。
* 合约安全：当调用锁时，会生成一个单独的合约，其中包含timelocked Ethereum，从而降低单个合约中包含的价值。

Edgeware持有者可以投票决定将区块奖励或分发的额外部分分配给其他社区，比如DOT持有者。

## Lockdrop合约

Lockdrop 合约是一个简单的双功能智能合约。当使用下面指定的接口调用函数lock时，将创建一个新的合约来保存timelocked Ethereum。Lockdrop 合同将接受“lock drops ”两周，在初始锁定期后两周接受"lock signals"。下面讨论前两周的 lockdrop 时期。

|    名称     | 数据类型 |                            描述                            |
| :---------: | :------: | :--------------------------------------------------------: |
|    term     |   Term   |                 明确ETH可被锁定时长的枚举                  |
| edgewareKey |  bytes   |          在创世，指定的Edgeware账户，会被分配EDG           |
| isValidator |   bool   | 允许个人选择是否这个Edgeware账户在启动时想要成为一个验证人 |

注意，当isValidator 为“是”时，整个验证时间内总的Edgeware 资产将作为验证资产。在Edgeware 上，锁的长度分别为3个月、6个月和12个月，锁定ETH的奖金分别为0%、10%和40%。

合约本身并不持有全部被锁定的以太坊，从而降低了恶意攻击的潜在价值。相反，当调用 lock 时，会创建一个单独的合约来保存timelocked Ethereum。lockdrop 和lock 合约非常简单。特别是，在没有跳转的情况下完成调用之前，lock 合约只有45条指令。这些合同将得到正式验证，并接受第三方审计。

在最初的两周之后，锁信号将被接受。个人将能够使用类似于 carbon 投票的函数调用，在有效的零月锁定时间内接收 EDG 。信号对冷钱包和硬件钱包都是开放的。在这些钱包里，有相当一部分代币是留给长期使用以太坊的人的。似乎长期参与者会是在Edgeware 上提供有价值的贡献者，因此他们的参与应该受到激励。为了正确地执行一个信号，在删除周期结束时对主以太链进行快照，以防止系统攻击或双锁，个人可以将资金从一个地址转移到另一个地址，并尝试对代币发出两次信号。

注意，如果个人通过信号参与，他们将在 post-Edgeware 网络启动12个月后收到他们的EDG 。此外，对于从一个钱包发出的ETH 信号，当时收到的 EDG 金额将被扣除40%。此外，发信号者将不能成为活跃的验证人。

| 锁定时长（月） | 方法   | 比例  | 接受EDG             |
| -------------- | ------ | ----- | ------------------- |
| 0              | Signal | 0.60x | post 网络启动12月后 |
| 3              | Drop   | 1.00x | 网络启动            |
| 6              | Drop   | 1.10x | 网络启动            |
| 12             | Drop   | 1.40x | 网络启动            |

在EDG 代币被锁定的一年内，在网络启动时接收 EDG 的个人将能够通过下注和参与其他网络活动获得代币。

代币分发机制的提议——drops 和 signals ——应该以更公平的方式，既激励积极参与者，又扩大分配。

## 创世区块

在启动后，个人将能够单独验证锁定或发出信号事件的以太坊总量，方法是查找两个关键事件，并在 Edgeware 上计算相应的计算平衡。

我们希望在 lockdrop 中分发90%的初始 Edgeware 代币。为了引导治理并证明共识算法在对抗条件下有效，需要公平而广泛的代币分发。Edgeware 是一个最初由Commonwealth Labs 开发和管理的项目。其他代币将分发给Commonwealth Labs(4.5%)，其他协助项目的合作伙伴，如 Parity Technologies (3.0%)，以及预留给最初的开源贡献者、社区参与者和测试网用户(2.5%)的额外代币。

# 治理

绝大多数(估计80%)的实用代币和智能合约区块链都表明了它们转向链上和分散化治理的意图。然而，在撰写本文时，很少有链上治理的成功实现。

迄今已推出的少数系统主要是链上国债、委托权益证明(DPoS)系统和直接代币加权投票模型。直接代币加权的链上流程通常很容易被链上的大型经济持有者捕获，如区块生产者、挖矿团队和交易所。此外，现在很少有治理系统具有任何级别的积极使用。交流、对话和投票受到同样的非直观界面的限制，这阻碍了密码货币的整体增长。

Edgeware 旨在成为开发有效的在线管理的先锋网络。通过迭代产品接口、投票系统和其他管理原语，Edgeware 可以加速核心技术的实现和部署，如分片、权益证明、高效的 SNARK 实现和 runtime 更改，以比其他区块链更快的速度实现技术进步。通过使用明确定义的链上治理流程，Edgeware 网络升级将更易于协调，部署速度更快。

## 治理模块

Edgeware 治理的核心功能是在几个可扩展的模块中实现的：

* Signaling：非绑定轮询是现有区块链治理过程的重要组成部分。用户还应该能够创建和分发民意调查，以表示对不同建议、策略和方向的兴趣。
* Identity：Edgeware 需要一个优等的标识原语，这样人们就可以在治理过程中彼此标识。
* Democracy：民主模块最初限制对提议的 binary 投票的投票。该模块允许委托投票——提高分配给投票的总权益。未来的升级应该允许用户在许多不同的提案投票方法中进行选择(例如，binary 或 rank-choice )。
* Council：Edgeware 允许投票者提名一组在网络上拥有特定权利的账户。委员会成员可以对提议产生异议。
* Treasury：这个模块允许个人对新生成的 EDG 的分配进行投票。

在Edgeware 的后续迭代中，将包含对附加治理功能的支持。其他功能的开发将由链上金库提供资金，包括：

* Secure voting：用户应该能够匿名投票，以防止购买选票的攻击，这需要在协议层实现加密原语。
* DAOs：随着 Edgeware 生态系统的发展，将会出现团队在网络初始治理结构之外进行协调的需求。从长期来看，我们希望通过用户构建和管理 DAOs 来实现这一点，这允许在不更改核心治理的情况下探索其他决策结构。例如，DAOs 可能遵循决策过程，包括简单的一人一票、委托投票、二次投票和更复杂的模型。

注意，进一步的治理改进可以在合约或者模块层次上实现。

### Signaling

信号模块允许用户投票创建非绑定投票。非绑定轮询是现有区块链治理过程的重要组成部分，以前以一种特别的方式进行管理。用户应该天生能够创建和分发民意调查，以表示对不同的建议、策略和方向的兴趣。

个人可以在许多不同的提案投票方法中进行选择(例如 binary 或 rankchoice )，此外还允许委托投票——提高分配给投票的总股份。一旦选择了一个方向，该委员会或民主模块可以创建一个有约束力的链上公投。

### Identity

在互联网上，电子邮件地址已经成为用户身份和身份验证的实际标准。它们是普遍可互操作的，并且可以跨服务进行识别。目前还没有在区块链上出现如此广泛采用的身份识别系统。最近才在现实世界中部署的区块链应用程序的几个领域从某种形式的持久用户标识中受益匪浅，其中包括贷款、协议治理或管理网络。对于贷款协议，身份是必要的，因为担保不足的贷款是不可行的，而且有风险，除非贷款人或承销商知道他们合作者的身份。对于协议治理，身份是构建社会共识的必要条件。

虽然有些人可能选择完全使用假名，但 Edgeware 网络上的参与者可能会发现，将身份或声明链接到同名地址是有用的。通过将一个 opt-in 身份解决方案构建到Edgeware (一个优等的原语)中，Edgeware 将成为一个理想的平台，用于分布式的金融、治理和其他 dApp 。

理想情况下，身份标准将为前面提到的例子提供服务，同时具有电子邮件登录的易用性和通用寻址性。Edgeware 身份识别系统实现了这一功能，详细内容见 `Identityrecord` ：

| 名称          | 数据类型                 | 描述                                                         |
| ------------- | ------------------------ | ------------------------------------------------------------ |
| account       | AccountId                | 身份关联的特定Edgeware账户                                   |
| identity      | Vec<u8>                  | 一个对应身份的字节数组。可以被加密和解密。                   |
| stage         | IdentityStage            | 一个表示提案生命周期的枚举——被注册，被证实或被验证。         |
| expiration    | BlockNumber              | 可选的。身份成为有效的区块。                                 |
| proof         | Attestation              | 为一个身份发送的证明。                                       |
| verifications | <Vec<(AccountId, bool)>> | 一个数组，具有相应的元组帐户和bool 。显示第三方个人是否可以验证身份和账户是实际关联的。 |
| metadata      | MetadataRecord           | 一个可选字段，允许人们在 Edgeware 上添加头像、标语或显示名称。这允许系统上有一个人可读的标识符。 |

标识模块中的公共函数允许帐户注册、验证或验证该标识。账户firstregisteran身份。然后，个人通过与外部证据相联系来验证这一点。第三方现在可以验证这个链接的身份是否有效，无论认证是否有效。验证必须来自特定选定的验证者，而不一定是任何第三方个人。初始验证器集是edgeware网络上的验证器。验证标识的工作可以委托给守护进程来管理此过程。

身份模块中的公共功能允许账户注册，证明或验证该身份。 帐户第一次注册身份。 此后，个人通过链接到外部证据来证实。 第三方现在可以验证该链接身份是否有效，投票证明该证明是否有效。证据必须来自特定选择的验证者，而不一定只是第三方个人。 初始验证人集是 Edgeware 网络上的验证人。可以将验证身份的工作委托给守护进程来管理此过程。

注册和证明的发送者必须相同，有助于减轻错误的证明。因此，区块链作为身份登记证明的真实单一来源。验证者应该只查询存储在链中的证据，以确定它是否有效，而不是证明的任何其他形式或表示。不同设置的证明示例：

* Github：标识是Github 用户名。 用户名注册后，注册商应在用户名账户下发布Github Gist证明，并附带 Edgeware 注册链接。
* Ethereum：身份是以太坊公钥或地址。 注册用户名注册后，注册商应将0值交易发送到刻录地址，该刻录地址表示其对应的 Edgeware 账户的公钥，并提交交易哈希作为证明。在检查时，验证者应该有足够的证据证明，如果以太坊帐户的所有者也不拥有收件人 Edgeware 账户（在以太坊上作为目标地址代表），那么他们就不会发出这样的交易。

最初，验证人集是具有最小0.1％ EDG 代币的账户集。 身份验证的投票是代币加权的。 Edgeware 这样做是为了有组织地建立一个一人一票政府用于系统的其他方面。

### Democracy

民主模块在投票过程中创建和移动不同的具有约束力的公民投票。公民投票允许以root用户身份在链上执行任意的外部调用。这允许在不执行 runtime 升级的情况下执行具有重要意义的动作（例如，设置平衡，修改时间戳等）。该模块目前仅支持binary 大部分通过投票，除非由理事会成员提交（其中包括绝大多数拒绝和简单多数通过）。个人投票通过代币比例和锁定时间调整来计算 binary 选择投票。

首先，所有投票将限制在两周的固定期限内。在未来，民主模块将扩展到许多投票类型，他们是：

* Binary：传统的 是/否 投票；
* Multi-Option：个人可以多种选择；
* Commit-Reveal：通过发布投票权来承诺特定投票的计划。

在投票决议 TallyType 的改革中，更改权衡投票的方法。 它们列举如下：

* Coin-Weight：个人账户投票的投票是每投币一个硬币 —— 一枚硬币相当于一票；
* Lock-time weight：账户以特定的锁定持续时间投票，其中较长的时间对应于更多的投票权。 例如，一个投票可能被锁定一到两周，后者的时间对应于两倍的投票权增加；
* One-persion-one-vote：一个账户或身份可以投一票。在 Edgeware 上，一人一票可以引用一组特定的经过验证的身份，例如经过验证的Github帐户。

最初，所有关于 Edgeware 的公投都将采用硬币加权和锁定时间加权。一个重要的注意事项，对于硬币加权投票，用户不能进行“部分基金投票” —— 用户只能在锁定期间投票并锁定所有资金用于给定账户，或者根本不投票。锁定时间加权允许具有较小代币资产的个人账户在管理过程中行使更多权力。在决议通过后，在实施任何变更之前会有一段延迟 —— 最初设定为两周。

此外，民主模块允许委派投票，即直接和代议制民主之间的中间点。在投票结束之前的任何时候，账户可以投票的方式与该账户可能委派的账户的投票方式不同。自愿代表团有可能增加政治参与，减少选举过程中的战略激励，并改善政治委托 —— 代理问题。委托可以递归。Edgeware 可以缓解攻击，恶意个人可以将委托树的深度操作到极端深度。Edgeware 将代表团限制为五个，而不是强制所有节点统计选票以遍历深层代表树。 另外，循环授权被阻止。

民主将通过增加对支持排名选择和匿名投票的不同投票类型的支持来扩展，其中账户直接链接到他们的投票。或者通过添加不同的 TallyingType 例如二次投票。

### Council

分散式系统为在线和主动参与提供了一个有问题的场景，往往导致低投票率。因此，理事会成员对仲裁，所需票数以及肯定和非肯定投票之间的相对差异提出异议。Edgeware 理事会模块将进行扩展，允许24个个人账户在固定期限为12个月的理事会成员的网络中拥有一些特殊权利。 网络启动时，理事会成员人数将为6人。

理事会在同行基础上投票，即没有硬币加权投票。仲裁异议允许理事会改变所需的有效绝对多数，以便在提案通过后可以更容易或更难以通过支持或反对它的不清晰的多数投票权。如果所有理事会成员投票支持提案，那么所需的非理事会成员票数就会被收取。 反之亦然。当理事会投票并且不止一个成员表示异议时，仲裁就会产生负面意见。

### Treasury

虽然验证人通常是其他区块链中唯一受到激励的利益相关者，但 Edgeware 建议使用区块奖励来激励所有利益相关者。区块通胀将被分配给一个链上金库，将由具体的金库提案分配。

最初，链上金库将由50％的区块奖励资助。Edgeware 代币持有者可以分配资金到：

* Core Technology：实现缩放技术，例如 runtime 改进，分片和脱链扩展；
* Goverance：治理系统，包括链上身份以及用于组织和协调核心开发人员工作的工具；
* Developer exprience：用于开发，调试和测试智能合约的成熟工具链；
* User experience：钱包和用户体验原语（例如，JavaScript库）使分散的应用程序简单易用。
* Ecosystem support：通过现场活动吸引开发人员，最终用户和其他利益相关者。

## 治理界面

Commonwealty Labs 开发了一个用于管理分布式网络的界面。该界面将支持对提案进行讨论和投票，创建和管理与Web 2.0社交媒体帐户相关联的身份，并在不同的通信渠道中跟踪有关 Edgeware 提案的讨论。

Commonwealth Labs还可以运行Slack，Riot或Discord聊天。Edgeware 生态系统中的参与者之间相互通信。由于 Edgeware 遵循标准的Parity Substrate API，用户可以替代地使用标准的 Substrate 库来手动访问链，阅读治理提案的状态，并在本地签署交易以向网络提交投票。

## 治理过程

Edgeware 利益相关者将能够通过链式信号在网络路线图上进行协作，通过举行非正式投票来包含特定项目和提案。这个规划阶段是链上治理过程的自然前身，正式投票是为了资助和实现同意的功能。

建立强大的进行链上升级的程序将是确保网络安全性的重要步骤。所有网络升级都应由多个独立方进行审核和测试，并且还需要大量的EDG持有人批准。

## 实验平台

从长远来看，Edgeware 的发展和成功需要涉及大量承诺的利益相关者，包括验证者，投票硬币持有者，应用程序开发人员和核心技术开发人员。我们预计，随着时间的推移，这些类别的利益相关者之间的协调的权威机构将会出现。

然而，为了加速这一过程，Edgeware 将为利益相关者群体提供一套工具来试验新的治理形式，例如不同的投票模型或决策制定流程。原生治理工具允许 DAOs 有组织地出现，以测试不同治理系统的有效性。

例如，虽然个人和链下公司将成为金库资金的第一个接收者，但 DAOs 可能会出现以协调虚假网络中的工作参与者，可以在 Edgeware 网络上建立身份和跟踪记录。 或者，随着匿名投票系统的部署，应该可以使用可链接的环签名来轮询受信任的开发人员、投资人或专家组，以建立围绕关键网络升级和治理决策的氛围。

通过该实验过程发现的成功模式可以在进一步的迭代中重复使用和改进。它们可以在Layer 2协议，公共产品基金 DAOs 以及其他项目上重复使用，这些项目都是在软件链本身上孵化的。

# 总结

分布式计算现在处于早期开发阶段，应该以快速迭代周期和雄心勃勃的路线图开发。通过渐进式升级策略创建一个新的受管理链，可以通过正确的激励措施实现这一目标。

作为一个循环的自我改进系统，Edgeware 可以投资，开发和部署网络升级，以支持网络的性能和可用性。这使得 Edgeware 成为现有区块链的新颖替代品。它协调激励，以便快速发展和开放的治理流程创造一个良性循环，治理的早期成功反映在网络、社区和生态系统的价值中。

# 参考文献

[1] Vitalik Buterin.Governance, Part2: Plutocracy Is Still Bad. URL: [vitalik.ca/general/2018/03/28/plutocracy.html](<https://vitalik.ca/general/2018/03/28/plutocracy.html>).

[2] Byzantine Finality Gadgets.2018. URL: https://github.com/w3f/consensus/blob/master/pdf/grandpa.pdf.

[3] Dillon  Chen.What’s in a Lockdrop.  2018. URL:https://medium.com/commonwealth-labs/whats-in-a-lockdrop-194218a180ca.

[4] Parity Substrate.URL:https://github.com/paritytech/substrate.

[5] Drew  Stone.Anonymous Voting: A Design Survey.  2018.URL: [https ://medium.com/commonwealth-labs/anonymous- voting- a- design-survey-12de869dc97f](https://medium.com/commonwealth-labs/anonymous-voting-a-design-survey-12de869dc97f) .

[6] Dr. Gavin Wood. Polkadot: Vision for a Hetergeneous Multi-Chain Frame-work. URL:https://polkadot.network/PolkaDotPaper.pdf.