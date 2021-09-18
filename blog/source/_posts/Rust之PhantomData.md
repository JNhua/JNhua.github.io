---
title: Rust之PhantomData
date: '2020/10/10 23:48:24'
updated: '2020/11/23 20:06:51'
tags: []
category:
  - Rust
  - Rust基础
mathjax: true
toc: false
abbrlink: a3f13347
---
`PhantomData<T>`是一个零大小类型的标记结构体。
<!--more-->
作用：
1. 并不使用的类型；
2. 型变；
3. 标记拥有关系；
4. 自动trait实现（send/sync）；

# 并不使用的类型

## 生命周期


```rust
struct Slice<'a, T> {
    start: *const T,
    end: *const T,
}
```

然而，因为'a在结构体中未使用，所以它是无界的。在结构定义中禁止无限生命周期和类型。

修改如下：

```rust
use std::marker;

struct Iter<'a, T: 'a> {
    ptr: *const T,
    end: *const T,
    _marker: marker::PhantomData<&'a T>,
}
```

## 类型

```rust
pub struct RetryableSendCh<T, C: Sender<T>> {
    ch: C,
    name: &'static str,
    marker: PhantomData<T>,
}
```

# 标记拥有关系

```rust
struct Vec<T> {
    data: *const T, // *const for variance!
    len: usize,
    cap: usize,
}
```

原生指针不具有所有权语义，drop检查器将认为`Vec<T>`不具有类型T的任何值。这将反过来使得它不需要担心Vec在其析构函数中丢弃任何`T`。但是有可能`T`被提前释放。为了drop检查器认为vec一定拥有了`T`的数据，修改如下：

```rust
use std::marker;

struct Vec<T> {
    data: *const T, // *const for variance!
    len: usize,
    cap: usize,
    _marker: marker::PhantomData<T>,
}
```

# 型变

* 不变

如果不能将一个类型替换为另一个类型，那么这个类型就称之为：**不变**。

* 逆变

可以由其基类替换。

* 协变

可以由其派生类型替换。

## `PhantomData`模式表

这是一张`PhantomData`可以使用的所有方法的表格：

| Phantom type                | `'a`      | `T`                       |
| --------------------------- | --------- | ------------------------- |
| `PhantomData<T>`            | -         | variant (with drop check) |
| `PhantomData<&'a T>`        | variant   | variant                   |
| `PhantomData<&'a mut T>`    | variant   | invariant                 |
| `PhantomData<*const T>`     | -         | variant                   |
| `PhantomData<*mut T>`       | -         | invariant                 |
| `PhantomData<fn(T)>`        | -         | contravariant (*)         |
| `PhantomData<fn() -> T>`    | -         | variant                   |
| `PhantomData<fn(T) -> T>`   | -         | invariant                 |
| `PhantomData<Cell<&'a ()>>` | invariant | -                         |

(*) 如果发生变性的冲突，这个是不变的。

