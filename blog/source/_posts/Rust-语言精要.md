---
title: Rust语言精要
copyright: true
date: 2019-09-04 10:37:54
categories:
- Rust
tags:
- Rust
---

# 函数指针 & 闭包

## 函数指针

### 函数作为参数

```rust
pub fn math(op: fn(i32, i32) -> i32, a: i32, b: i32) -> i32 {
    op(a, b)
}
fn sum(a: i32, b:i32) -> i32 {
    a + b
}
fn main() {
    let a =2;
    let b =3;
    assert_eq!(math(sum, a, ,b), 5);
}
```

<!-- more -->

### 函数作为返回值

```rust
fn is_true() -> bool { true }
fn true_maker() -> fn() -> bool { is_true }
fn main() {
    assert_eq!(true_maker()(), true);
}
```

## 闭包

闭包在函数中的应用，常常与trait结合。

### 闭包作为参数

```rust
fn closure_math<F: Fn() -> i32>(op: F) -> i32 {
    op()
}
fn main() {
    let a = 2;
    let b = 3;
    assert_eq!(closure_math(|| a + b ), 5);
}
```

### 闭包作为返回值

```rust
fn two_times_impl() -> impl Fn(i32) -> i32 {
    let i = 2;
    move |j| j * i
}
fn main() {
    let result = two_times_impl();
    assert_eq!(result(2), 4);
}
```

闭包默认会按引用捕获变量（在此例中为 i ）。如果将此闭包返回，则引用也会跟着返回。而 i 会被销毁，所以引用变为悬垂指针。因此要加上move关键字，移动所有权。

# 元组 & 数组

## 数组

```rust
let arr: [i32; 3] = [1, 2, 3];
```

数组初始就固定了长度，即使声明为mut也只能修改索引上的元素，而不是数组本身。

## 元组

```rust
let tuple: (&'static str, i32, char) = ("hello", 5, 'c');
let (x, y ,z) = tuple;
```

当元组只有一个值的时候，需要写成(x, )，这是为了和括号中的其他值进行区分。

# 模式匹配

## match

与 switch / case 类似。

## if let / while let

在某些场合替代match，简化代码。

# 切片

Slice是对一个数组的引用片段，代表一个指向数组起始位置的指针和数字长度。

Slice是一种新的类型，而不是简单的一个引用而已。

```rust
let arr: [i32;2]=[1,2];
assert_eq!((&arr).len(), 2);
assert_eq!(&arr.len(), &2);
```

## str（字符串切片）

字符串常量是&'static str。str字符串是固定长度的，String字符串是可变长度的。

`let hello_world = "Hello, World!";` 声明了一个 `&str`类型；

`let hello_world: &'static str = "Hello, world!";` hello_world 与 字符串常量一样。

# 结构体

Rust中结构体有：具名结构体、元组结构体、单元结构体。

其中，元组结构体只有一个字段时，称之为New Type模式。

# trait

在Rust中，trait是唯一的接口抽象方式。Rust中没有继承，贯彻的是组合优于继承和面向接口编程的思想。

```rust
struct Duck;
struct Pig;
trait Fly {
    fn fly(&self) -> bool;
}
impl Fly for Duck {
    fn fly(&self) -> bool {
        true
    }
}
impl Fly for Pig {
    fn fly(&self) -> bool {
        false
    }
}
fn fly_static<F: Fly>(s: F) -> bool {
    s.fly()
}
fn fly_dyn(s: &dyn Fly) -> bool {
    s.fly()
}

fn main() {
    let pig = Pig;
    assert_eq!(fly_static::<Pig>(pig), false);
    let duck = Duck;
    assert_eq!(fly_static::<Duck>(duck), true);
    assert_eq!(fly_dyn(&Pig), false);
    assert_eq!(fly_dyn(&Duck), true);
}
```

fly_static是静态分发，fly_dyn是动态分发。

# 注释

## 文档注释

内部支持Markdown标记，也支持对文档中的示例代码进行测试，可以用rustdoc生成HTML文档。

* /// ：生成库文档，用于函数或结构体的说明；
* //! ：生成库文档，用于说明整个模块的功能；

# println!宏

* `println!("{}", 2)`，nothing表示Display；
* `println!("{:?}", 2)`，？表示Debug；
* o代表八进制，x/X表示十六进制，b表示二进制；
* p代表指针；
* e/E表示指数；