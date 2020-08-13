---
title: write-runtime
copyright: true
date: 2019-09-10 14:53:46
categories:
- BlockChain
- Substrate
- base
tags:
---

# 开始

本地启动substrate节点

```
./target/release/substratekitties purge-chain --dev
./target/release/substratekitties --dev
```

<!-- more -->

如果你要开始一个新项目并希望获得最新版本的 Substrate，你可以通过运行以下命令来构建自己的 Substrate 包：

```
substrate-node-new <project_name><your_name>
substrate-ui-new <project_name>
```

就像之前说过的，该方法的一个缺点就是这些脚本直接从不同的 GitHub 仓库中提取，这意味着在有 breaking changes 时可能会出现不兼容的情况。

使用 [oo7-substrate library](https://github.com/paritytech/oo7/tree/master/packages/oo7-substrate) 库构建的 [Substrate UI](https://github.com/paritytech/substrate-ui) 是另一个可替代 [Polkadot-JS Apps UI](https://github.com/polkadot-js/apps) 的前端界面, 它们都可以用于与你的 Substrate 链进行交互。

构建自己的 UI 时，请记住参考 [Polkadot-JS API 文档](https://polkadot.js.org/api/) 和 [oo7 API 文档](https://paritytech.github.io/oo7/)。

两种编译：

```bash
./build.sh               // Build Wasm
cargo build --release    // Build binary
```

# 创建Module

```bash
substratekitties
|
+-- runtime
    |
    +-- src
        |
        +-- lib.rs
        |
        +-- * substratekitties.rs
```

module文件的位置

## lib.rs操作

在runtime的lib.rs中导入module文件，` mod kitties`

实现其Trait， `impl kitties::Trait for Runtime`

将该module包含在construct_runtime!宏中, `Substratekitties: substratekitties::{Module, Call, Storage}`

## module文件

### decl_storage!

srml/support 有storage相关函数； put ，get，和storage声明带有的getter函数

### decl_module!

处理逻辑的所有入口；函数的第一个参数常常是origin；返回类型是support::dispatch中的Result类型，ok(())或者Err()，不要panic。

ensure_signed(origin)?在system::ensure_signed中；

support::StorageMap

```Rust
//声明
SomeValue get(some_value_getter): map u32 => u32;
MyValue: map T::AccountId => u32;
//插入
<SomeValue<T>>::insert(key, value);
//查询
let my_value = <SomeValue<T>>::get(key);
let also_my_value = Self::some_value_getter(key);
```

结构体

```rust
use parity_codec_derive::{Encode, Decode};
#[derive(Encode, Decode, Default, Clone, PartialEq)] //允许基本地实现某些 trait

#[cfg_attr(feature = "std", derive(Debug))] 
//对 Debug trait 做了同样的事情，但仅在使用“标准”库时启用，即在编译本机二进制文件而不是 Wasm 的时候。

pub struct MyStruct<A, B> {
    some_number: u32,
    some_generic: A,
    some_other_generic: B,
}

MyItem: map T::AccountId => MyStruct<T::Balance, T::Hash>;
```

初始化

```rust
let hash_of_zero = <T as system::Trait>::Hashing::hash_of(&0);
let my_zero_balance = <T::Balance as As<u64>>::sa(0); //as 和 sa 转换
```

导入liballoc库来使用string；

在UI里注册结构体，在 **Settings** app 页面的 **Developer** 部分中，你可以提交包含有自定义 struct 的 JSON 文件或者通过代码编辑器手动添加。

随机数生成

```rust
let sender = ensure_signed(origin)?;
let nonce = <Nonce<T>>::get();
let random_seed = <system::Module<T>>::random_seed();

let random_hash = (random_seed, sender, nonce).using_encoded(<T as system::Trait>::Hashing::hash);

<Nonce<T>>::mutate(|n| *n += 1);
```

***Substrate* Rutime中要先验证再写入**

可以使用映射和计数器模拟列表 代替list；

```rust
AllPeopleArray get(person): map u32 => T::AccountId;
AllPeopleCount get(num_of_people): u32;
let all_people_count = Self::num_of_people();
let new_all_people_count = all_people_count.checked_add(1).ok_or("Overflow adding a new person")?;
//记住加？号

```

使用tuple模拟二维数组

```rust
MyFriendsArray get(my_friends_array): map (T::AccountId, u32) => T::AccountId;
MyFriendsCount get(my_friends_count): map T::AccountId => u32;

```

------

### decl_event!

事件声明，在lib.rs也需要更新；

```rust
pub trait Trait: balances::Trait {
    type Event: From<Event<Self>> + Into<<Self as system::Trait>::Event>;
}

decl_event!(
    pub enum Event<T>
    where
        <T as system::Trait>::AccountId,
        <T as system::Trait>::Hash
    {
        Created(AccountId, Hash),
    }
);

//调用
Self::deposit_event(RawEvent::Created(sender, random_hash));

```

------

公有接口和私有函数，供decl_module! 调用

```rust
impl<T: Trait> Module<T> {
    // Your functions here
}

```

更新前检查值的存在

```rust
ensure!(<MyObject<T>>::exists(object_id));

```

