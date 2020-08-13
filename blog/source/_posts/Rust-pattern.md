---
title: Rust_pattern
copyright: true
date: 2020-08-12 16:24:15
categories:
- Rust
- base
tags:
---

# 模式

模式由如下内容组成：
（1）字面值
（2）解构的数组、枚举、结构体或者元组
（3）变量
（4）通配符
（5）占位符

<!-- more -->

可能用到模式的位置：

* match
* if let
* while let
* for
* let 
* 函数参数

# 反驳

模式有两种：refutable（可反驳的）和 irrefutable（不可反驳的）。能匹配任何传递的可能值的模式被称为是不可反驳的。对值进行匹配可能会失败的模式被称为可反驳的。

不可反驳的：函数、let语句、for循环。原因：因为通过不匹配的值程序无法进行有意义的工作。

if let和while let表达式被限制为只能接受可反驳的模式，因为它们的定义就是为了处理有可能失败的条件。

# 语法

## 匹配

* 匹配字面值

  ```rust
  match x {
      1 => println!("one"),
      _ => println!("anything"),
  }
  ```

* 匹配命名变量

  ```rust
  let y = 10;  //1处
  match x {
      Some(50) => println!("Got 50"),
      Some(y) => println!("Matched, y = {:?}", y), //此处的y和上面1处的y不一样，此处是引入的变量y覆盖之前的y
      _ => println!("Default case, x = {:?}", x),
  }
  ```

* 多个模式

  ```rust
  match x {
      1 | 2 => println!("one or two"),
      _ => println!("anything"),
  }
  ```

* 范围

  ```rust
  match x {
      'a'..='j' => println!("early ASCII letter"),
      'k'..='z' => println!("late ASCII letter"),
      _ => println!("something else"),
  }
  ```

## 解构

* 结构体

  ```rust
  let p = Point { x: 0, y: 7 };
      let Point { x: a, y: b } = p;
      assert_eq!(0, a);
      assert_eq!(7, b);
     //let Point { x, y } = p;   //创建了同名的变量，可以简写
  ```

* 枚举

* 元组

## 忽略

* 使用`_`忽略整个值或部分值；
* 用`..`忽略剩余值，必须是无歧义的（一般只用一个）。

## 其他

* 匹配守卫：匹配守卫是一个指定于match分支模式之后的额外的if条件，它必须满足才能选择此分支。

* 绑定：`@`运算符允许我们在创建一个存放值的变量，并且测试这个变量的值是否匹配模式。

  ```rust
  match msg {
      Message::Hello { id: id_variable @ 3..=7 } => {  //创建id_variable 存放id的值，同时测试值是否在3到7的范围
          println!("Found an id in range: {}", id_variable)
      },
    _ => {}
  }
  ```



