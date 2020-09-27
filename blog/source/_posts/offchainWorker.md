---
title: Off-Chain Workers喂价
copyright: true
date: 2020-03-23 16:34:51
categories:
- BlockChain
- Substrate
- base
tags:
---

# OCW喂价

## 什么时候使用OCW

![off-chain-workers-v2](http://image-jennerblog.test.upcdn.net/img/off-chain-workers-v2.png)

例如：长时间运行的任务、外部服务请求（http）、数据的加解密和签名等。

<!-- more -->

## 在Runtime中使用

```rust
// For better debugging (printout) support
use support::{ debug, dispatch };
use system::offchain;
use sp_runtime::transaction_validity::{
  TransactionValidity, TransactionLongevity, ValidTransaction, InvalidTransaction
};
```

在配置 trait 中包括以下关联类型，用于从链下工作机发送已签名和未签名的交易。

```rust
pub trait Trait: timestamp::Trait + system::Trait {
  /// 总的事件类型
  type Event: From<Event<Self>> + Into<<Self as system::Trait>::Event>;
  type Call: From<Call<Self>>;
  type SubmitUnsignedTransaction: offchain::SubmitUnsignedTransaction<Self, <Self as Trait>::Call>;
  type BlockFetchDur = BlockFetchDur;
}
```

在宏 `decl_module！` 模块中，定义 `offchain_worker` 函数。此函数作为链下工作机的入口点，并在每次导入区块后运行。

```rust
decl_module! {
  pub struct Module<T: Trait> for enum Call where origin: T::Origin {

    // --snip--

    fn offchain_worker(block_number: T::BlockNumber) {
      let duration = T::BlockFetchDur::get();

      // Type I task: fetch price
      if duration > 0.into() && block % duration == 0.into() {
        for (sym, remote_src, remote_url) in FETCHED_CRYPTOS.iter() {
          if let Err(e) = Self::fetch_price(block, *sym, *remote_src, *remote_url) {
            debug::error!("Error fetching: {:?}, {:?}: {:?}",
              core::str::from_utf8(sym).unwrap(),
              core::str::from_utf8(remote_src).unwrap(),
              e);
          }
        }
      }

      // Type II task: aggregate price
      <TokenSrcPPMap<T>>::enumerate()
        // filter those to be updated
        .filter(|(_, vec)| vec.len() > 0)
        .for_each(|(sym, _)| {
          if let Err(e) = Self::aggregate_pp(block, &sym) {
            debug::error!("Error aggregating price of {:?}: {:?}",
              core::str::from_utf8(&sym).unwrap(), e);
          }
        });
    } // end of `fn offchain_worker()`
  }
}
```

默认情况下，链下工作机无法直接访问用户密钥（即使在开发环境中），由于安全原因，只能访问应用特定的子密钥（subkeys）。需要在 runtime 顶部定义 `KeyTypeId` 用于将应用特定的子密钥分组，如下所示：

```rust
// 密钥类型ID可以是任何4个字符的字符串
pub const KEY_TYPE: KeyTypeId = KeyTypeId(*b"btc!");

// --snip--

pub mod crypto {
  pub use super::KEY_TYPE;
  use sp_runtime::app_crypto::{app_crypto, sr25519};
  app_crypto!(sr25519, KEY_TYPE);
}
```

和任何其他 pallet 一样，runtime 必须实现 pallet 的配置 trait。进入位于 `runtime/src/lib.rs` 的 runtime `lib.rs`。

```rust
// 定义交易签名人
type SubmitTransaction = system::offchain::TransactionSubmitter<
  offchain_pallet::crypto::Public, Runtime, UncheckedExtrinsic>;

impl price_fetch::Trait for Runtime {
	type Event = Event;
	type Call = Call;
	type SubmitUnsignedTransaction = SubmitPFTransaction;
	type BlockFetchDur = BlockFetchDur;
}
```

在宏 `contrast_runtime!` 中，将所有不同的 pallet 作为 runtime 的一部分。 如果在链下工作机中使用未签名的交易，则添加另外一个参数 `ValidateUnsigned`。需要为此编写自定义验证逻辑。

```rust
construct_runtime!(
  pub enum Runtime where
    Block = Block,
    NodeBlock = opaque::Block,
    UncheckedExtrinsic = UncheckedExtrinsic
  {
    // --snip--

    // 使用未签名交易
    PriceFetch: price_fetch::{Module, Call, Storage, Event<T>, ValidateUnsigned},
  }
);
```

### 在 `service.rs` 中添加密钥（Keys）

使用`KeyTypeId`指定本地密钥库来存储特定于应用的密钥，链下工作机可以访问这些密钥来签署交易。需要通过以下两种方式之一添加密钥。

### 选项 1（开发阶段）：添加第一个用户密钥作为应用的子密钥

在开发环境中，可以添加第一个用户的密钥作为应用的子密钥。更新 `node/src/service.rs` 如下所示。该config的参数需要在启动结点时，加入参数。

```rust
pub fn new_full<C: Send + Default + 'static>(config: Configuration<C, GenesisConfig>)
  -> Result<impl AbstractService, ServiceError>
{
  // --snip--

  // 给Alice clone密钥
  let dev_seed = config.dev_key_seed.clone();

  // --snip--

  let service = builder.with_network_protocol(|_| Ok(NodeProtocol::new()))?
    .with_finality_proof_provider(|client, backend|
      Ok(Arc::new(GrandpaFinalityProofProvider::new(backend, client)) as _)
    )?
    .build()?;

  // 添加以下部分以将密钥添加到keystore
  if let Some(seed) = dev_seed {
    service
      .keystore()
      .write()
      .insert_ephemeral_from_seed_by_type::<runtime::offchain_pallet::crypto::Pair>(
        &seed,
        runtime::offchain_pallet::KEY_TYPE,
      )
      .expect("Dev Seed should always succeed.");
  }
}
```

这样就可以签名交易了。这仅对 **开发阶段** 有利。

### 选项2：通过 CLI 添加应用的子密钥

在更实际的环境中，在设置 Substrate 节点后，可以通过命令行接口添加一个新的应用子密钥。如下所示：

```rust
# 生成一个新帐户
$ subkey -s generate

# 通过RPC提交一个新密钥
$ curl -X POST -vk 'http://localhost:9933' -H "Content-Type:application/json;charset=utf-8" \
  -d '{
    "jsonrpc":2.0,
    "id":1,
    "method":"author_insertKey",
    "params": [
      "<YourKeyTypeId>",
      "<YourSeedPhrase>",
      "<YourPublicKey>"
    ]
  }'
```

新密钥已添加到本地密钥库（keystore）中。

### 未签名交易

使用以下代码，可以将未签名的交易发送回链。

```rust
decl_module! {
  pub struct Module<T: Trait> for enum Call where origin: T::Origin {
    // --snip--

    pub fn record_price(
      origin,
      _block: T::BlockNumber,
      crypto_info: (StrVecBytes, StrVecBytes, StrVecBytes),
      price: u64
    ) -> dispatch::DispatchResult {
      // Ensuring this is an unsigned tx
      ensure_none(origin)?;

      let (sym, remote_src) = (crypto_info.0, crypto_info.1);
      let now = <timestamp::Module<T>>::get();

      // Debug printout
      debug::info!("record_price: {:?}, {:?}, {:?}",
        core::str::from_utf8(&sym).map_err(|_| "`sym` conversion error")?,
        core::str::from_utf8(&remote_src).map_err(|_| "`remote_src` conversion error")?,
        price
      );

      <TokenSrcPPMap<T>>::mutate(&sym, |pp_vec| pp_vec.push((now, price)));

      // Spit out an event and Add to storage
      Self::deposit_event(RawEvent::FetchedPrice(sym, remote_src, now, price));

      Ok(())
    }

    fn offchain_worker(block: T::BlockNumber) {
      // -- snip --
      if let Err(e) = Self::fetch_price(block, *sym, *remote_src, *remote_url){
     			// ...
      }
    }
    
    fn fetch_price<'a>(
        block: T::BlockNumber,
        sym: &'a [u8],
        remote_src: &'a [u8],
        remote_url: &'a [u8]
    ) -> Result<()> {
      debug::info!("fetch price: {:?}:{:?}",
        core::str::from_utf8(sym).unwrap(),
        core::str::from_utf8(remote_src).unwrap()
      );
      let json = Self::fetch_json(remote_url)?;
      let price = match remote_src {
        // -- snip --
      }?;

      // 这里指定下一个区块导入阶段的链上回调函数。
      let call = Call::record_price(
        block,
        (sym.to_vec(), remote_src.to_vec(), remote_url.to_vec()),
        price
      );
      // Unsigned tx
      T::SubmitUnsignedTransaction::submit_unsigned(call)
        .map_err(|_| "fetch_price: submit_unsigned(call) error")
    }
  }
}
```

默认情况下，所有未签名的交易都被视为无效交易。需要在`my_offchain_worker.rs`中添加以下代码段，以显式允许提交未签名的交易。

```rust
decl_module! {
  // --snip--
}

impl<T: Trait> Module<T> {
  // --snip--
}

#[allow(deprecated)]
impl<T: Trait> support::unsigned::ValidateUnsigned for Module<T> {
  type Call = Call<T>;

  fn validate_unsigned(call: &Self::Call) -> TransactionValidity {

    match call {
      Call::record_price(block, input) => Ok(ValidTransaction {
        priority: 0,
        requires: vec![],
        provides: vec![(block, input).encode()],
        longevity: TransactionLongevity::max_value(),
        propagate: true,
      }),
      _ => InvalidTransaction::Call.into()
    }
  }
}
```

添加 `deprecated` 属性，以防止显示警告消息。这是因为这一部分API仍然处于过渡阶段，并将在即将发布的 Substrate 版本中进行更新。请暂时谨慎使用。

了解更多关于 [未签名交易](https://zhuanlan.zhihu.com/p/99809960/conceptual/node/extrinsics.md#unsigned-transactions) 的信息。

### 链上回调函数中的参数

在进行链上回调时，我们的实现会将函数名称及其所有参数值一起哈希。回调将在下次区块导入时被存储和调用。如果我们发现哈希值存在，这意味着之前已经调用了具有相同参数集的函数，那么对于签名交易，如果以更高的优先级调用该函数，则该函数将被替换；对于未签名交易，此回调将被忽略。

如果你的 pallet 定期进行链上回调，并希望它偶尔有重复的参数集，则始终可以从`offchain_worker`函数传入当前区块号外的其他参数。该数字只会增加，并且保证是唯一的。

### 获取外部数据

要从第三方API获取外部数据，请在 `my_offchain_worker.rs` 中使用 `offchain::http` 库，如下所示。

```rust
use sp_runtime::{
  offchain::http,
  transaction_validity::{
    TransactionValidity, TransactionLongevity, ValidTransaction, InvalidTransaction
  }
};

// --snip--

decl_module! {
  pub struct Module<T: Trait> for enum Call where origin: T::Origin {
    // --snip--
    fn offchain_worker(block: T::BlockNumber) {
      match Self::fetch_data() {
        Ok(res) => debug::info!("Result: {}", core::str::from_utf8(&res).unwrap()),
        Err(e) => debug::error!("Error fetch_data: {}", e),
      };
    }
  }
}

impl<T: Trait> Module<T> {
  fn fetch_data() -> Result<Vec<u8>, &'static str> {

    // 指定请求
    let pending = http::Request::get("https://min-api.cryptocompare.com/data/price?fsym=BTC&tsyms=USD")
      .send()
      .map_err(|_| "Error in sending http GET request")?;

    // 等待响应
    let response = pending.wait()
      .map_err(|_| "Error in waiting http response back")?;

    // 检查HTTP响应是否正确
    if response.code != 200 {
      debug::warn!("Unexpected status code: {}", response.code);
      return Err("Non-200 status code returned from http request");
    }

    // 以字节形式收集结果
    Ok(response.body().collect::<Vec<u8>>())
  }
}
```

之后可能需要将结果解析为JSON格式。我们这里有一个在 `no_std` 环境中，使用外部库解析JSON的 [示例](https://link.zhihu.com/?target=https%3A//github.com/jimmychu0807/substrate-offchain-pricefetch/blob/047ad8094dc21e2bced2d055707756265be32e95/node/runtime/src/price_fetch.rs%23L250-L260)。

### 参考文档

- Substrate `im-online 模块`，一个 Substrate 内部的 pallet，使用链下工作机通知其他节点，网络中的验证人在线。