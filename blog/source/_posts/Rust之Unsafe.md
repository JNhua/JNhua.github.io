---
title: Rust之Unsafe
date: '2020/10/10 23:50:29'
updated: '2020/10/10 23:50:51'
tags: []
category:
  - Rust
  - Rust基础
mathjax: true
---
# Unsafe

<!--more-->
Rust会通过unsafe关键字切换到不安全的Rust。不安全的Rust具有以下超级力量：
（1）解引用裸指针
（2）调用不安全的函数或者方法
（3）访问或修改可变静态变量
（4）实现不安全的trait

注意：unsafe并不会关闭借用检查器或禁用任何其它的Rust安全检查规则，它只提供上述几个不被编译器检查内存安全的功能。unsafe也不意味着块中的代码一定就是不ok的，它只是表示由程序员来确保安全。

# 解引用裸指针

裸指针是可变和不可变的，分别写作`*const T`和`*mut T`。此处的星号不是解引用运算符，而是类型名称的一部分。
裸指针：
（1）允许忽略借用规则，可以同时拥有不可变和可变的指针，或多个指向相同位置的可变指针
（2）不保证指向有效的内存
（3）允许为空
（4）不能实现任何自动清理功能

可以在安全代码中创建裸指针，只是不能在不安全块之外解引用裸指针。

```rust
fn main() {
    let mut num = 5;
    //创建不可变和可变的裸指针可以在安全的代码中，只是不能在不安全代码块之外解引用裸指针
    let r1 = &num as *const i32;
    let r2 = &mut num as *mut i32;

    unsafe {
        println!("r1 is: {}", *r1);
        println!("r2 is: {}", *r2);
    }

    let add = 0x12345usize;
    let _r = add as *const i32;
}
```

# 访问可变静态变量

常量和静态变量的区别：
a、静态变量中的值有一个固定的内存地址（使用这个值总会访问相同的地址），常量则允许在任何被用到的时候复制其数据。
b、静态变量可以是可变的，虽然这可能是不安全的（所以要用unsafe）。

```rust
static mut COUNTER: u32 = 0;
fn add_counter(inc: u32) {
    unsafe {
        COUNTER += inc;
    }
}

fn main() {
    add_counter(3);
    add_counter(3);
    unsafe {
        println!("counter: {}", COUNTER);
    }
}
```

# 不安全的trait

（1）当至少有一个方法中包含编译器不能验证的不变量时，该 trait 是不安全的；
（2）在 trait 之前增加 unsafe 关键字将 trait 声明为 unsafe，同时 trait 的实现也必须标记为 unsafe。