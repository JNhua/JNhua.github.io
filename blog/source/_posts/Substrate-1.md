---
title: Substrate基础_github项目的README
date: 2019-04-19 16:57:26
categories: 
- Substrate
- base
tags: Substrate
copyright: true
---

# 简介

Substrate是下一代区块链革新框架。

# 概述

Substrate的核心由三个技术组成：[WebAssembly](https://webassembly.org/), [Libp2p](https://libp2p.io/) 和 GRANDPA 共识。关于GRANDPA共识，查看此[定义](https://hackmd.io/Jd0byWX0RiqFiXUVC78Bdw?view#GRANDPA)，[介绍](https://medium.com/polkadot-network/grandpa-block-finality-in-polkadot-an-introduction-part-1-d08a24a021b5)和[形式规范](https://github.com/w3f/consensus/blob/master/pdf/grandpa.pdf)。它既是一个构建新链的库，又对于区块链客户端来说是一个“万能钥匙”，能够与任何基于Substrate的链进行同步。

<!-- more -->

Substrate的链有三个不同特性使得他们成为“下一代”（区块链）：动态、自定义状态转移功能；轻客户端功能性；一种具有快速块生成和自适应、确定性的先进共识算法。STF，用Wasm编码，作为“runtime”。这个定义了`execute_block`功能，并且可以从权益算法、交易语义、日志记录机制和替换自身任何一部分过程或区域链状态（“治理”）来指定所有（链）。因为runtime是完全动态的，所有这部分在任何时候可以被替换或者被更新。一个Substrate链是一个“活的组织”。

更多关于Substrate的资料请看：  [Substrate是什么？](https://jnhua.github.io/2019/04/20/what-is-Substrate/)

# 使用

Substrate仍然是一个早期项目，然而它已经被用来当做Polkadot的主要项目，所以使用它也是一个重要的任务。特别地，你应该有完备的区块链概念和基础密码学知识。比如对header, block, client, hash, transaction 和signature 这些术语足够熟悉。目前你将需要能够做任何事情的Rust工作知识（不过最终的目标，是不再需要这些）。

Substrate被设计成有三种用法：

1. 平凡的：通过运行Substrate二进制文件` substrate ` 并且以包括当前演示的runtime的一个创世区块来设置它。在这里，你仅仅构建Substrate，设置了一个JSON文件和启动你自己的链。这提供给你最少的定制灵活性，非常基础地允许你去改变各种runtime模块的创始参数：balances, staking, block-period, fees 和 governance。
2.  模块化：通过从Substrate Runtime Module Library借用模块到一个新的runtime，可能改变或者重设的Substrate客户端的出块逻辑。这使在你自己链的逻辑上很大程度上的自由度，允许你改变数据格式，增删模块，关键是能增加你自己的模块。很多可以被改变而不需要触碰到出块逻辑（因为它是通用的）。如果在这种方法中，这时存在着的Substrate二进制文件可以被用作出块和同步。如果出块逻辑需要被改变，那么一个新的、改变了的出块二进制代码必须被构建成为独立项目并且被验证者所使用。这就是Polkadot转发链如何被构建的，并且能够满足所有接近中期的情况。
3.  通用的：整个Substrate Runtime Module Library可以被忽略，并且整个runtime是拼凑设计和实现的。如果希望的话，这个可以被除了Rust的语言完成，只要该语言可以面向WebAssembly。如果runtime可以被制作成根据已有出块逻辑编译的，那么你可以简单地从你的Wasm二进制文件构建一个新的创世区块，然后用已有的基于Rust的Substrate客户端启动你的区块链。如果不是，那么你将需要去改变客户端的出块逻辑。对现在的大部分项目这可能是一个没用的选项，但是为Substrate范例提供了可以长期有效升级路径的完整自由度。

## Substrate基础

Substrate是一个区块链平台，有着完全通用的状态转移功能。这是说，它同时具有关于基础数据结构的标准和约定（特别是关于Runtime Module Library）。概括地说，这些核心数据类型对应于按照实际非议的标准+trait+s和通用的按照惯例+struct+s。

```c
Header := Parent + ExtrinsicsRoot + StorageRoot + Digest
Block := Header + Extrinsics + Justifications
```

## Extrinsics

Extrinsics在Substrate中是来自“外部世界”的信息碎片，这些信息被链上区块包含。你也许会想“啊，那意思是交易”：事实上，不是的。Extrinsics分成两大类，只有其中之一是交易。另一部分是固有的。这两部分的不同是：交易被签名和在网上传播，可以被认为是自身有用的。这符合在比特币或者以太坊中被你称为的交易模型。

与此同时，固有部分，不被传播也不被签名。它们代表描述环境但是不要求任何东西来验证它的数据，例如签名。相反，他们被认为是“真实的”仅仅是因为足够多的验证者已经同意它们是合理的。

举个例子，固有时间戳设置块的当前时间戳。这不是Substrate的固定部分，但是作为Substrate Runtime Module Library的一部分，可以根据需要来使用。没有签名可以从根本上证明，以与签名可以“证明”花费某些特定资金的愿望完全相同的方式，在给定时间来出块。相反，每个验证人的作用是确保他们在同意候选块有效之前将时间戳设置为合理的值。

其他例子包括Polkadot中的“parachain-heads” extrinsic和用于Substrate Runtime Module Library的“note-missed-proposal” extrinsic来决定和惩罚或者使离线验证人失效。

## Runtime 和 API

Substrate链都有runtime。runtime是一个Wasm二进制代码，包括了一些接口。一些接口被要求作为基础Substrate规范的一部分。其他的仅仅是惯例和被要求是Substrate客户端的默认实现使其能够出块。

如果你想要用Substrate开发一条链，你会需要去实现`Core`的trait。这个`Core`trait生成一个API，包含与你的runtime进行交互的最小必要功能。一个特殊的宏被提供：`impl_runtime_apis!`，帮助你实现runtime的API traits。所有runtime API trait实现需要在一次调用`impl_runtime_apis!`后被完成。所有参数和返回值需要实现[`parity-codec`](https://crates.io/crates/parity-codec)来变得可编码和可解码。

这里是一个Polkadot API作为PoC-3实现的一部分：

```rust
impl_runtime_apis! {
	impl client_api::Core<Block> for Runtime {
		fn version() -> RuntimeVersion {
			VERSION
		}

		fn execute_block(block: Block) {
			Executive::execute_block(block)
		}

		fn initialize_block(header: <Block as BlockT>::Header) {
			Executive::initialize_block(&header)
		}
	}
	// ---snip---
}
```

## 固有的Extrinsics

Substrate Runtime Module Library包括时间戳和缩减功能。如果使用，这些功能依赖于通过固有的extrinsics传递的“可信”外部信息。Substrate引用出块客户端软件，希望能够使用整理数据调用的runtime API（在引用Substrate出块客户端的情况下，这仅仅是当前时间戳以及处于离线状态的节点），以便返回相应的extrinsics来准备包容。如果要在修改的rumtime中使用新的固有extrinsic类型，那么这个功能（及其参数类型）将会改变。

## 出块逻辑

在Substrate中，区块链同步和出块之间存在一个主要区别（“authoring”是比特币中所谓的“mining”更通用的术语）。第一种情况可能被称为“完整节点”（或“轻节点” -  Substrate支持两者）：出块必然需要同步节点，因此，所有出块客户端必须能够同步。然而，反之则不然。出块节点不在“同步节点”中的主要功能有三个：交易队列逻辑，固有交易认知和BFT共识逻辑。 BFT共识逻辑是作为Substrate的核心元素提供的，可以忽略，因为它只在SDK下的`authorities()` API条目中公开。

Substrate中的交易队列逻辑被设计为尽可能通用，允许runtime通过`initialize_block` 和 `apply_extrinsic`调用来表示哪些交易适合包含在块中。但是，优先级和替换策略等更微妙的方面目前必须表示为“硬编码”，作为区块链出块代码的一部分。也就是说，Substrate的交易队列参考实现应该足以用于初始链实现。

固有的外在部分在某种程度上是通用的，并且按照惯例，外部函数的实际构造被委托给runtime中的“软代码”。如果链中需要额外的外部信息，则需要更改出块逻辑以将其提供到runtime，并且运行时的`inherent_extrinsics`调用将需要使用此额外信息以构造任何其他外部事务包含在块中。  

---

以下部分略。

原文链接：https://github.com/paritytech/substrate/blob/master/README.adoc 





