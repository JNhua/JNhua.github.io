---
title: Rust-trait
copyright: true
date: 2019-09-05 18:00:11
categories:
- Rust
tags:
- Rust
- trait
---

# 行为上对类型的约束

trait是Rust对Ad-hoc（点对点/特别的/临时的）多态的支持。

## 接口抽象

* 接口中可以定义方法，并支持默认实现；
* 接口中不能实现另一个接口，但是接口之间可以继承；
* 同一个接口可以同时被多个类型实现，但不能被同一个类型实现多次；

为不同的类型实现trait，属于一种函数重载，也是Ad-hoc多态。

### 关联类型

```rust
pub trait Add<RHS = Self> {
	type Output;
	fn add(self, rhs: RHS) -> Self::Output;
}
```

Self是每个trait都带有的隐式类型参数，代表实现当前trait的具体类型。实现时，未指明泛型，默认为Self类型。

Output为关联类型。

#### 为u32类型实现Add trait

```rust
impl Add for u32 {
    type Output = u32;
    fn add (self, other: u32) -> u32 { self + other }
}
```

### 孤儿规则

如果要实现某个trait，那么该trait和实现该trait的那个类型至少有一个要在当前crate中定义。

### trait继承

```rust
trait Page{
    fn set_page(&self, p: i32){
        println!("Page Default: 1");
    }
}
trait PerPage{
    fn set_perpage(&self, num: i32){
        println!("Per Page Default: 10");
    }
}
trait Paginate: Page + PerPage{
    fn set_skip_page(&self, num: i32){
        println!("Skip Page : {:?}", num);
    }
}
impl <T: Page + PerPage>Paginate for T{}
struct MyPaginate{ page: i32 }
impl Page for MyPaginate{}
impl PerPage for MyPaginate{}
fn main() {
    let my_paginate = MyPaginate{page: 1};
	my_paginate.set_page(2);
	my_paginate.set_perpage(100);
	my_paginate.set_skip_page(12);
}
```

`impl <T: Page + PerPage>Paginate for T{}` 为拥有Page和PerPage行为的类型实现Paginate。

使用继承，可以不影响之前的代码，加入上面这一行就可以添加新的trait。

## 泛型约束

```rust
use std::ops::Add;
fn sum<T: Add<T, Output=T>>(a: T, b: T) -> T{
    a + b
}
assert_eq!(sum(1u32, 2u32), 3);
assert_eq!(sum(1u64, 2u64), 3);
```

约束sum函数，只有实现了Add这个trait的类型才可以当做参数。

where关键字，可以把泛型中的trait限定移到语句最后。

## 抽象类型

#### trait对象

trait的类型大小在编译期间无法确定，所以trait对象必须使用指针。可以利用引用操作符&或Box<T>来制造一个trait对象。trait对象的结构体包含一个可变data指针和一个可变虚表指针。

作参数时，trait限定是静态分发的，trait对象是动态分发的。

### impl Trait

静态分发的抽象类型impl Trait。目前只可以在输入的参数和返回值两个位置使用。

```rust
fn fly_static(s: impl Fly + Debug) -> bool {
	s.fly()
}
fn can_fly(s: impl Fly+Debug) -> impl Fly {
    if s.fly(){
        //...
    } else {
        //...
    }
    s
}
```

impl Trait只能用于单个参数指定抽象类型。

dyn Trait是与impl Trait相对应的动态分发。

## 标签trait

5个重要的标签：Sized（编译器可确定大小），Unsize（动态大小），Copy（可以按位复制），Send（跨线程安全通信），Sync（线程间安全共享引用）。

# trait对象的生命周期

* trait对象的生命周期默认是'static；
* 如果实现trait的类型包含&'a X 或者 &'a mut X，则默认生命周期是'a；
* 如果实现trait的类型只有T: 'a，则默认生命周期是'a；
* 如果实现trait的类型包含多个类似T:'a 的从句，则生命周期需要明确指定。





