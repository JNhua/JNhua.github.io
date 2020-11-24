---
title: Rust包管理
date: '2020/10/10 23:27:00'
updated: '2020/11/23 20:00:12'
tags: []
category:
  - Rust
  - Rust基础
mathjax: true
abbrlink: 484acaf
---
# 包管理（Cargo）
## cargo 命令创建包
`cargo new xxx --lib` 创建一个名为xxx的包；
<!--more-->
`cargo new xxx` 或者 `cargo new xxx --bin` 创建一个名为xxx的可被编译为可执行文件的包。

## 使用第三方包

1. 在`Cargo.toml`中的`[dependencies]`下加入包的依赖；
2. 在需要引入的文件头部加入` extern crate` 包名; 之后才可以`use` 包（`Rust 2015`）。在`2018`中，直接可以用`use xxx`。

Note：`Cargo`默认把连字符替换为下划线。

## Cargo文件格式

### [package]表配置

描述的都是与包相关的元数据，比如包名，作者等。数组使用[]，多段文字使用"""。

`build = "buidl.rs"` 指定构建脚本； `workspace = ".."` 指定工作空间为父目录。

### [badges]表配置

云端的持续集成服务。

### [workspace]表配置

`members = ["bench", "regex-capi"]`	指定子包。

### [dependencies]表配置

配置依赖文件。

### [features]表配置

条件编译功能相关。在代码中对应`#[cfg(featrue = "xxx")]`。

### [lib]表配置

表示最终编译目标库的信息：name，crate-type， path， test， bench等。

### [ [test] ]表配置

用两个中括号，表示数组。

### [profile]表配置

自定义rustc编译配置。

# 模块系统

如果存在与文件名同名的目录，则该目录下的模块都是该文件的子模块。

![1](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029110452.png)

`read_func.rs`

```rust
pub mod static_kv;  //pub关键字使得可以在main.rs中：use read_func::static_kv
pub fn read_kv () {
}
pub fn rw_mut_kv() -> Result<(), String> {
}
```

Rust会通过`mod`关键字去当前模块的子模块中寻找`static_kv`模块。模块名字可以被以下表述：1.模块名.rs；2.与模块名相同的文件夹，并且该文件夹包含`mod.rs`。

`read_func/static_kv.rs`

```rust
use lazy_static::lazy_static;
use std::collections::HashMap;
use std::sync::RwLock;
```

`main.rs`

```rust
mod read_func;
use crate::read_func::{read_kv , rw_mut_kv};	//因为之前声明了pub
fn main() {
}
```

第一行mod引入模块。

第二行的`crate`可以用`self`代替，代表当前的`crate`，以`main.rs`为起点寻找当前相对路径下的`read_func`模块。如果是第三方包，就不需要写`crate`前缀。

## 模块间的关系

### 模块嵌套

```rust
mod sound {
    mod instrument {
        mod woodwind {
            fn clarinet() {
                // 函数体
            }
        }
    }
    mod voice {
    }
}
fn main() {
}
```

树形结构：

```
crate
└── sound
    ├── instrument
    │   └── woodwind
    └── voice
```

私有性规则有如下：

* 所有项（函数、方法、结构体、枚举、模块和常量）默认是私有的。
* 可以使用 pub 关键字使项变为公有。
* 不允许使用定义于当前模块的子模块中的私有代码。
* 允许使用任何定义于父模块或当前模块中的代码。

```rust
mod sound {
    pub mod instrument {    //加上Pub后可用路径访问instrument
        pub fn clarinet() {    //加上pub后可以用该函数
            // 函数体
        }
    }
}
fn main() {
    // 绝对路径
    crate::sound::instrument::clarinet();
    // 相对路径
    sound::instrument::clarinet();
}
```

也可以使用 ***super*** 开头来构建相对路径。这么做类似于文件系统中以 .. 开头：该路径从 **父** 模块开始而不是当前模块。

```rust
mod instrument {
    fn clarinet() {
        super::breathe_in();
    }
}
fn breathe_in() {
    // 函数体
}
```

`clarinet `函数位于` instrument` 模块中，所以可以使用 `super` 进入 `instrument` 的父模块，也就是根 `crate`。从这里可以找到 `breathe_in`。使用`super`相对路径可以方便的进行扩展，而不用更改路径来调用。

`sound`模块放入`sound.rs`文件中，调用方式不变。

### 重新导出

```rust
pub use crate::sound::instrument;
```

可以简化外部调用的导出路径（外部调用：`use xxx::instrument; `），也不需要对外暴露模块（`sound`）。

一般重新导出放在`lib.rs`中。`main.rs`结合`lib.rs`的形式，是二进制包的最佳实践。

## 可见性

```rust
pub mod outer_mod {
    pub(self) fn outer_mod_fn() {}
     
    pub mod inner_mod {
        // 在Rust 2018 edtion 模块系统必须使用use导入
        use crate::outer_mod::outer_mod_fn;
        // 对外层模块 `outer_mod` 可见
        pub(in crate::outer_mod)  fn outer_mod_visible_fn() {}
        // 对整个crate可见
        pub(crate) fn crate_visible_fn() {}
        // `outer_mod` 内部可见
        pub(super) fn super_mod_visible_fn() {
            // 访问同一模块的函数
            inner_mod_visible_fn();
            // 使用use导入了outer_mod
            outer_mod_fn();
        }
        // 仅在`inner_mod`可见
        pub(self) fn inner_mod_visible_fn() {}
    }
     
    pub fn foo() {
        inner_mod::outer_mod_visible_fn();
        inner_mod::crate_visible_fn();
        inner_mod::super_mod_visible_fn();
     
        // 不能使用inner_mod 的私有函数
        // inner_mod::inner_mod_visible_fn();
    }
}
fn bar() {
    // 该函数对整个crate可见
    outer_mod::inner_mod::crate_visible_fn();
 
    // 该函数只对outer_mod可见
    // outer_mod::inner_mod::super_mod_visible_fn();
 
    // 该函数只对outer_mod可见
    // outer_mod::inner_mod::outer_mod_visible_fn();
     
    // 通过foo函数调用内部细节
    outer_mod::foo();
}
fn main() { bar() }
```

关于pub：

* 如果不显示使用pub，则函数或者模块可见性默认为私有的；
* pub，可以对外暴露公共接口；
* pub(crate)，对整个crate可见；
* pub(in Path)，其中Path是模块路径，表示可以通过此Path路径来限定可见范围；
* pub(self) / pub(in self)，只限当前模块可见；
* pub(super) / pub(in super)，当前模块和父模块中可见。

*Note*：trait中关联类型和Enum中变体的可见性，会随着trait和Enum的可见性而变化。但是结构体中的字段需要单独使用pub来改变可见性。

