---
title: Rust语言精要
copyright: true
date: 2019-09-04 10:37:54
categories:
- Rust
- base
tags:
---

# 变量

声明变量关键字：`let`

变量值分为两种类型：

* 可变的（`mut`）
* 不可变

<!-- more -->

变量类型：

* 布尔型 - `bool` 表示 true 或 false
* 无符号整型- `u8` `u32` `u64` `u128` 表示正整数
* 有符号整型 - `i8` `i32` `i64` `i128` 表示正负整数
* 指针大小的整数 - `usize` `isize` 表示内存中内容的索引和大小
* 浮点数 - `f32` `f64`
* 元组（tuple） - `(value, value, ...)` 用于在栈上传递固定序列的值
* 数组 - 在编译时已知的具有固定长度的相同元素的集合
* 切片（slice） - 在运行时已知长度的相同元素的集合
* `str`(string slice) - 在运行时已知长度的文本

可以通过将类型附加到数字的末尾来明确指定数字类型（如 `13u32` 和 `2u8`）。

使用`as`进行类型转换：

```rust
let a = 13u8;
let b = 7u32;
let c = a as u32 + b;
```

## 元组 & 数组

### 数组

```rust
let arr: [i32; 3] = [1, 2, 3];
```

数组初始就固定了长度，即使声明为mut也只能修改索引上的元素，而不是数组本身。

### 元组

```rust
let tuple: (&'static str, i32, char) = ("hello", 5, 'c');
let (x, y ,z) = tuple;
```

当元组只有一个值的时候，需要写成(x, )，这是为了和括号中的其他值进行区分。

# 表达式

```rust
let x = (let y = 6);	//error
```

`let`没有右值语义，Rust中不允许这么用，可以这么写：

```rust

let y = {
	let x = 3;
	x
};
```

# 函数

## 函数

### 多个返回值

函数可以通过**元组**来返回多个值。

元组元素可以通过他们的索引来获取。

支持各种形式的解构，允许我们以符合人类工程学的方式提取数据结构的子片段。

### 返回空

如果没有为函数指定返回类型，它将返回一个空的元组，也称为*单元*。

一个空的元组用 `()` 表示。

## 函数指针

函数指针实现了所有三个闭包 trait（Fn、FnMut 和 FnOnce），所以总是可以在调用期望闭包的函数时传递函数指针作为参数。倾向于编写使用泛型和闭包 trait 的函数，这样它就能接受函数或闭包作为参数。

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

### 函数作为返回值

```rust
fn is_true() -> bool { true }
fn true_maker() -> fn() -> bool { is_true }
fn main() {
    assert_eq!(true_maker()(), true);
}
```

## 闭包

闭包在函数中的应用，常常与`trait`结合。

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

闭包默认会按***引用***捕获变量（在此例中为 `i` ）。如果将此闭包返回，则引用也会跟着返回。而 i 会被销毁，所以引用变为悬垂指针。因此要加上move关键字，移动所有权。

或者如下写法，用`Box<T>`：

```rust
fn return_clo() -> Box<dyn Fn(i32)->i32> {
    Box::new(|x| x+1)
}

fn main() {
    let c = return_clo();
    println!("1 + 1 = {}", c(1));
    println!("1 + 1 = {}", (*c)(1)); //解引用多态
    println!("Hello, world!");
}
```

### 捕获环境值

闭包可以通过三种方式捕获其环境，它们对应函数的三种获取参数的方式，分别是获取所有权、可变借用、不可变借用。这三种捕获值的方式被编码为如下三个Fn trait：
（1）FnOnce消费从周围作用域捕获的变量，闭包周围的作用域被称为其环境。为了消费捕获到的变量，闭包必须获取其所有权并在定义闭包时将其移进闭包。其名称的Once部分代表了闭包不能多次获取相同变量的所有权。
（2）FnMut获取可变的借用值，所以可以改变其环境。
（3）Fn从其环境获取不可变的借用值。
当创建一个闭包时，rust会根据其如何使用环境中的变量来推断我们希望如何引用环境。由于所有闭包都可以被调用至少一次，因此所有闭包都实现了FnOnce。没有移动被捕获变量的所有权到闭包的闭包也实现了FnMut，而不需要对捕获的变量进行可变访问的闭包实现了Fn。

### 自动推导

闭包会为每个参数和返回类型推导一个具体类型，但是不能推导两次。如下错误：

```rust
let example_closure = |x| x;
let s = example_closure(String::from("hello"));
let n = example_closure(5); //报错，尝试推导两次，变成了不同的类型
```

### 与trait结合

```rust
struct Cacher<T> 
    where T: Fn(u32) -> u32
{
    calcuation: T,
    value: Option<u32>,
}

impl<T> Cacher<T>
    where T: Fn(u32) -> u32
{
    fn new(calcuation: T) -> Cacher<T> {
        Cacher {
            calcuation,
            value: None,
        }
    }
}

fn main() {
    let mut c = Cacher::new(|x| x+1);
}
```

# 流程控制

## while

略。

## for

```rust
for x in 0..5{
  printfln!("{}",x);
}
// 0..=5 包含5
```



```rust
let number = 3;
if number{}	//error
```

Rust不能从`number`推断出`bool`值。另，`if`后的判断表达式不需要括号。

## 从块表达式返回值

### if let

```rust
let number = if condition {
    5
} else {
    6
};
```

`if`、`else`返回值的类型必须是相同的。当所有`if`，`else if`块无法匹配时，调用任何一个`else`块，如果无`else`，则返回`()`。因此，此代码中的`if`表达式的`else`块（虽然没有显式写出）返回值为`()`，与`if`块中的`i32`类型不一致，报`E0308`错误。举例如下：

```rust
fn r(n: i32) -> i32 {
    if n > 0 {
        0
    }
    1
}
```

### match let

```rust
let result = match food {
        "hotdog" => "is hotdog",
        // 注意，当它只是一个返回表达式时，大括号是可选的
        _ => "is not hotdog",
};
```

## loop

```rust
let result = loop {
    counter += 1;
    if counter == 10 {
        // loop 可以被中断以返回一个值。
        break counter * 2;
    }
};
//result等于20。
```

避免使用`while true {...}`，使用`loop`。Rust使用LLVM，而LLVM没有表达无限循环的方式，因此在某些时候会出错。如下：

```rust
let x;
while true { x = 1; break; }
println!("{}", x);
```

编译器报错，"use of possibly uninitialised variable"。

## match

`match` 是穷尽的，意为所有可能的值都必须被考虑到。

```rust
match x {
        0 => {
            println!("found zero");
        }
        // 我们可以匹配多个值
        1 | 2 => {
            println!("found 1 or 2!");
        }
        // 我们可以匹配迭代器
        3..=9 => {
            println!("found a number 3 to 9 inclusively");
        }
        // 我们可以将匹配数值绑定到变量
        matched_num @ 10..=100 => {
            println!("found {} number between 10 to 100!", matched_num);
        }
        // 这是默认匹配，如果没有处理所有情况，则必须存在该匹配
        _ => {
            println!("found something else!");
        }
    }
```

# 结构体

Rust中结构体有：具名结构体、元组结构体、单元结构体。

## 具名结构体

```rust
struct SeaCreature {
    animal_type: String,
    name: String,
}
```

## 元祖结构体

```rust
// 这仍然是一个在栈上的结构体
let loc = Location(42, 32);
```

## 单元结构体

```rust
struct Marker;
```

其中，元组结构体只有一个字段时，称之为`New Type`模式。

# 方法（封装特性）

与函数（function）不同，方法（method）是与特定数据类型关联的函数。

**静态方法** — 属于某个类型，调用时使用 `::` 运算符。

**实例方法** — 属于某个类型的实例，调用时使用 `.` 运算符。

```rust
struct SeaCreature {
    noise: String,
}

impl SeaCreature {
    fn get_sound(&self) -> &str {
        &self.noise
    }
}

fn main() {
    let creature = SeaCreature {
      // 静态方法
        noise: String::from("blub"),
    };
  // 实例方法
    println!("{}", creature.get_sound());
}
```

# 枚举

```rust
enum Species {
    Crab,
    Octopus,
    Fish,
    Clam
}
match ferris.species {
        Species::Crab => println!("{} is a crab",ferris.name),
  			_ => xxx,
}
```

`enum` 的元素可以有一个或多个数据类型，从而使其表现得像 C 语言中的*联合*。

当使用 `match` 对一个 `enum` 进行模式匹配时，可以将变量名称绑定到每个数据值。

```rust
enum Weapon {
    Claw(i32, Size),
    Poison(PoisonType),
    None
}
...{
  ...
  weapon: Weapon::Claw(2, Size::Small),
}
match ferris.weapon {
  // num_claws 获取 2
    Weapon::Claw(num_claws,size) => {
      ...
  	}
}
```

# 泛型

```rust
// 一个部分定义的结构体类型
struct BagOfHolding<T> {
    item: T,
}

fn main() {
    // 注意：通过使用泛型，我们创建了编译时创建的类型，使代码更大
    // Turbofish 使之显式化
    let i32_bag = BagOfHolding::<i32> { item: 42 };
    let bool_bag = BagOfHolding::<bool> { item: true };
    
    // Rust 也可以推断出泛型的类型！
    let float_bag = BagOfHolding { item: 3.14 };

    let bag_in_bag = BagOfHolding {
        item: BagOfHolding { item: "boom!" },
    };
}

```

## 常用的内置泛型：

```rust
enum Option<T> {
    None,
    Some(T),
}
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`Result` 如此常见以至于 Rust 有个强大的操作符 `?` 来与之配合。

## option

### 用法

Option主要有以下一些用法：

- 初始化值；
- 作为在整个输入范围内没有定义的函数的返回值；
- 作为返回值，用`None`表示出现的简单错误；
- 作为结构体的可选字段；
- 作为结构体中可借出或者是可载入的字段；
- 作为函数的可选参数；
- 代表空指针；
- 用作复杂情况的返回值。

### 值复制方法

```rust
let x = 123u8;
let y: Option<&u8> = Some(&x);
let z = y.copied();
```

copied将引用转换为值。

# Vectors

`Vec` 有一个形如 `iter()` 的方法可以为一个 vector 创建迭代器，这允许我们可以轻松地将 vector 用到 `for` 循环中去。

```rust
fn main() {
    // 我们可以显式确定类型
    let mut i32_vec = Vec::<i32>::new(); // turbofish <3
    i32_vec.push(1);

    // 自动检测类型
    let mut float_vec = Vec::new();
    float_vec.push(1.3);

    // 宏！
    let string_vec = vec![String::from("Hello"), String::from("World")];

    for word in string_vec.iter() {
        println!("{}", word);
    }
}

```

内存细节：

- `Vec` 是一个结构体，但是内部其实保存了在堆上固定长度数据的引用。
- 一个 vector 开始有默认大小容量，当更多的元素被添加进来后，它会重新在堆上分配一个新的并具有更大容量的定长列表。（类似 C++ 的 vector）

## 迭代器

迭代器实现了`Iterator`（`trait`），定义于标准库中。该`trait`定义如下：

```rust
trait Iterator {
	type Item;
	fn next(mut self) -> Option<Self::Item>;
	//省略其它内容
}
```

如果希望迭代可变引用，可以使用`iter_mut`。

# 切片

Slice是对一个数组的引用片段，代表一个指向数组起始位置的指针和数组长度。

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

### utf8

```rust
// &a[3..7]表示螃蟹
let a = "hi 🦀";

// chars[3]表示螃蟹，chars[i]表示一个字符，每个char占4字节
let chars = "hi 🦀".chars().collect::<Vec<char>>();
```

# String

String不能用index访问的原因：

* 避免根据UTF-8编码得到的长度与期望的不一致；
* 根据index访问需要从头遍历（在Rust中需要判断有效字符数量），所以访问的时间复杂度不为`O(1)`。

## String格式

```rust
let haiku: &'static str = "
        I write, erase, rewrite
        - Katsushika Hokusai";
println!("{}", haiku);
    
println!("hello \
world") // notice that the spacing before w is ignored

let a: &'static str = r#"
        <div class="advice">
            Raw strings are useful for some situations.
        </div>
        "#;

// 从文件读取大量字符串
let 00_html = include_str!("00_en.html");
```

# HashMap
## 创建
```rust
HashMap::new();
let scores: HashMap<_, _> = keys.iter().zip(values.iter()).collect(); 
```
## 读取
```rust
let key = String::from("Blue");
let value = ss.get(&key); 
```
## 遍历
`for (key, value) in &ss`
## 更新
```rust
ss.insert(String::from("Blue"), 20);//会将之前Blue对应的值覆盖掉
ss.entry(String::from("Yellow")).or_insert(20); // 没有实体时插入

// 根据旧值更新
for word in text.split_whitespace() {
    let count = map.entry(word).or_insert(0);
    *count += 1;
}
```

# 所有权

所有权是Rust的特性。所有权解决了堆栈分配与回收问题。

## 内存分配

| 语言     | 内存回收机制                                                 |
| -------- | ------------------------------------------------------------ |
| 其他语言 | GC会跟踪声明的变量，当它不再被使用时，自动清除。如果没有GC，程序员负责在恰当的时候释放这段申请的内存。 |
| Rust     | 使用RAII(“资源获取即初始化”)，在变量`invalid`时，调用`drop`回收。 |

## Move，Copy，Clone

### Move

```rust
let s1 = String::from("hello");
let s2 = s1;
```

`s1`移动到了`s2`，不仅仅是`shallow copy`，`s1`还被置为`invalid`了。栈上的s1对象和s2对象进行按位浅拷贝，堆上数据不变。

将所有者作为参数传递给函数时，其所有权将移交至该函数的参数。 在一次**移动**后，原函数中的变量将无法再被使用。在**移动**期间，所有者的堆栈值将会被复制到函数调用的参数堆栈中。

Rust不会自动创建“深拷贝”，需要自己用`Clone()`。但是，如果实现了`Copy`的`trait`，那么值会被复制入栈。

### Copy

实现Copy的类型在堆上没有资源，值完全处于栈上。浅拷贝后，源与目标对象都可以访问，是独立的数据。为了`#[derive(Copy, Clone)]`工作，成员也必须实现`Copy`。

> 在派生语句中的Clone是需要的，因为Copy的定义类似这样:pub trait Copy:Clone {}，即要实现Copy需要先实现Clone。clone方法应该和Copy语义相容，等同于按位拷贝。

Copy与Drop不能同时存在，原因见下面。

### Drop

变量在离开作用范围时，编译器会自动销毁变量，如果变量类型有`Drop` trait，就先调用`Drop::drop`方法，做资源清理，一般会回收heap内存等资源，然后再收回变量所占用的stack内存。如果变量没有`Drop` trait，那就只收回stack内存。

如果类型实现了`Copy` trait，在copy语义中并不会做deep copy，那就会出现两个变量同时拥有一个资源（比如说是heap内存等），在这两个变量离开作用范围时，会分别调用`Drop::drop`方法释放资源，这就会出现double free错误。所以Copy与Drop不能同时存在。

### Clone

根据不同的类型，实现不同的复制语义。例如对于Box类型，执行“深拷贝”。对于Rc类型，仅仅把引用计数加1。

## 释放

释放是分级进行的。删除一个结构体时，结构体本身会先被释放，紧接着才分别释放相应的子结构体并以此类推。

内存细节：

- Rust 通过自动释放内存来帮助确保减少内存泄漏。
- 每个内存资源仅会被释放一次。

## 引用

引用默认也是不可变的。可变引用才可以修改被引用的值。已经被引用的变量，其所有权不可以被移动。

### 可变引用(&mut)

* 可变引用只能出现一次，避免数据竞争。
* 已有不可变引用，可变引用就不能再出现。

### 解引用

使用 `&mut` 引用时, 你可以通过 `*` 操作符来修改其指向的值。 你也可以使用 `*` 操作符来对所拥有的值进行拷贝（前提是该值可以被拷贝）。

操作符"`.`"可以自动解引用：

```rust
let f = Foo { value: 42 };
let ref_ref_ref_f = &&&f;
println!("{}", ref_ref_ref_f.value);
```

### 解引用多态与可变性交互

解引用多态有如下三种情况：

* 当 T: Deref<Target=U> 时从 &T 到 &U。
* 当 T: DerefMut<Target=U> 时从 &mut T 到 &mut U。
* 当 T: Deref<Target=U> 时从 &mut T 到 &U。（注意：此处反之是不可能的）

# 生命周期

生命周期的主要目标是避免悬垂引用，大部分时候是可以隐含并且被推断的。

## 显式生命周期

尽管 Rust 不总是在代码中将它展示出来，但编译器会理解每一个变量的生命周期并进行验证以确保一个引用不会有长于其所有者的存在时间。 同时，函数可以通过使用一些符号来参数化函数签名，以帮助界定哪些参数和返回值共享同一生命周期。 生命周期注解总是以 `'` 开头，例如 `'a`，`'b` 以及 `'c`。

```rust
// 参数 foo 和返回值共享同一生命周期
fn do_something<'a>(foo: &'a Foo) -> &'a i32 {
    return &foo.x;
}

// foo_b 和返回值共享同一生命周期
// foo_a 则拥有另一个不相关联的生命周期
fn do_something<'a, 'b>(foo_a: &'a Foo, foo_b: &'b Foo) -> &'b i32 {
    println!("{}", foo_a.x);
    println!("{}", foo_b.x);
    return &foo_b.x;
}

// 静态变量的范围也可以被限制在一个函数内
static mut SECRET: &'static str = "swordfish";

// 字符串字面值拥有 'static 生命周期
let msg: &'static str = "Hello World!";
```

## 隐式生命周期

三条规则确定不需要生命周期注解：

* 第一条规则是：每一个是引用的参数都有它自己的生命周期参数。
* 第二条规则是：如果只有一个输入生命周期参数，那么它被赋予所有输出生命周期参数：`fn foo<'a>(x: &'a i32) -> &'a i32`。 
* 第三条规则是：在struct的impl语句中，如果方法有多个输入生命周期参数，不过其中之一因为方法的缘故为 `&self` 或 `&mut self`，那么 `self` 的生命周期被赋给所有输出生命周期参数。第三条规则使得方法更容易读写，因为只需更少的符号。
```rust
use std::str::FromStr;
pub struct Wrapper<'a>(&'a str);

impl<'a> FromStr for Wrapper<'a> {
    type Err = ();
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        Ok(Wrapper(s))
    }
}
```

在上述例子中，fn from_str函数显然是符合第二条规则，也就是说入参s: &str的生命周期被赋予为输出的生命周期。但是，输出参数中的Self对应的类型为结构体Wrapper，而Wrapper是有生命周期的限制的，此时编译器不知道如何判断，因此报错。

## 结构体生命周期

如果结构体成员含有引用类型，则需要显式指定生命周期。如下：

```rust
struct StuA<'a> {
    name: &'a str,
}
```

相应的，在方法中，也需要声明结构体的生命周期：

```rust
impl<'b> StuA<'b> {
  // 隐式生命周期第二条规则
    fn do_something(&self) -> i32 {
        3
    }

    fn do_something2(&self, s: &str) -> &str{
      // 隐式生命周期第三条规则
    //相当于fn do_something2<'b>(&'b self, s: &str) -> &'b str{
      // self.name与self生命周期相同，✅
        self.name
    }

  // 返回值生命周期与s相同，而不是self，所以需要显式指定
    fn do_something3<'a>(&self, s: &'a str) -> &'a str{
        s
    }
}
```



# Trait（多态）

在Rust中，trait是唯一的接口抽象方式。Rust中没有继承，贯彻的是组合优于继承和面向接口编程的思想。

[trait相关详情](https://munan.tech/2019/09/05/Rust-trait/)

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

# 内存布局

Rust 程序有 3 个存放数据的内存区域：

- **数据内存** - 对于固定大小和**静态**（即在整个程序声明周期中都存在）的数据。 例如 “Hello World”字面值常量，该文本的字节只能读取，因此它们位于该区域中。 编译器对这类数据做了很多优化，由于位置已知且固定，因此通常认为编译器使用起来非常快。
- **栈内存** - 对于在函数中声明为变量的数据。 在函数调用期间，内存的位置不会改变，因为编译器可以优化代码，所以栈数据使用起来非常快。
- **堆内存** - 对于在程序运行时创建的数据。 此区域中的数据可以添加、移动、删除、调整大小等。由于它的动态特性，通常认为它使用起来比较慢， 但是它允许更多创造性的内存使用。当数据添加到该区域时，我们称其为**分配**。 从本区域中删除 数据后，我们将其称为**释放**。

## 结构体内存对齐

对齐规则：

* 每种类型都有一个数据对齐属性。在X86平台上u64和f64都是按照32位对齐的。
* 一种类型的大小是它对齐属性的整数倍，这保证了这种类型的值在数组中的偏移量都是其类型尺寸的整数倍，可以按照偏移量进行索引。需要注意的是，动态尺寸类型的大小和对齐可能无法静态获取。
* 结构体的对齐属性等于它所有成员的对齐属性中最大的那个。Rust会在必要的位置填充空白数据，以保证每一个成员都正确地对齐，同时整个类型的尺寸是对齐属性的整数倍。
* 不保证数据填充和成员顺序，编译器可能进行优化。

```rust
struct A {
    a: u8,
    b: u32,
    c: u16
}
```

按照前3条规则，A的大小应该为12字节，而实际上编译后可能只有8字节。

# 指针

原生指针：

- `*const T` - 指针常量。
- `*mut T` - 可变指针。

取得指针所指地址内的数据，需要在`unsafe{...}`中，因为不能保证该原生指针指向有效数据。



## 智能指针

智能指针通常使用结构体实现。智能指针区别于常规结构体的显著特征在于其实现了Deref和Drop trait。

* Deref trait允许智能指针结构体实例表现的像引用一样，这样就可以编写既用于引用，又用于智能指针的代码。
* Drop trait允许我们自定义当智能指针离开作用域时执行的代码。

### Box

`Box`将数据从栈上移动到堆，栈上存放指向堆数据的指针。

```rust
pub struct Box<T: ?Sized>(Unique<T>);

struct Ocean {
    animals: Vec<Box<dyn NoiseMaker>>,
}
let ocean = Ocean {
        animals: vec![Box::new(ferris), Box::new(sarah)],
};
```

适用于：

* 当有一个在编译时未知大小的类型，而又需要在确切大小的上下文中使用这个类型值的时候；（举例子：在一个list环境下，存放数据，但是每个元素的大小在编译时又不确定）；
* 当有大量数据并希望在确保数据不被拷贝的情况下转移所有权的时候；
* 当希望拥有一个值并只关心它的类型是否实现了特定trait而不是其具体类型时。

### Rc

引用计数指针，将数据从栈上移动到堆。允许其他`Rc`指针**不可变引用**同一个数据。单线程。

```rust
pub struct Rc<T: ?Sized> {
    ptr: NonNull<RcBox<T>>,
    phantom: PhantomData<RcBox<T>>,
}

let heap_pie = Rc::new(Pie);
let heap_pie2 = heap_pie.clone();

heap_pie2.eat();
heap_pie.eat();
// all reference count smart pointers are dropped now
// the heap data Pie finally deallocates
```

### RefCell

一个智能指针容器。可变与不可变引用都可以，引用规则与之前一样。单线程。

```rust
pub struct RefCell<T: ?Sized> {
    borrow: Cell<BorrowFlag>,
    value: UnsafeCell<T>,
}

fn main() {
    // RefCell validates memory safety at runtime
    // notice: pie_cell is not mut!
    let pie_cell = RefCell::new(Pie{slices:8});
    {
        // but we can borrow mutable references!
        let mut mut_ref_pie = pie_cell.borrow_mut();
        mut_ref_pie.eat();
        mut_ref_pie.eat();
        // mut_ref_pie is dropped at end of scope
    }
    // now we can borrow immutably once our mutable reference drops
     let ref_pie = pie_cell.borrow();
     println!("{} slices left",ref_pie.slices);
}
```

#### 内部可变性

```rust
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}
```

可以拥有一个表面上不可变的List，但是通过`RefCell<T>`中提供内部可变性方法来在需要时修改数据的方式。

### 弱引用

```rust
fn main() {
    let a = Rc::new(Cons(5, RefCell::new(Weak::new())));
  // 1, a strong count = 1, weak count = 0
	// 1, a tail = Some(RefCell { value: (Weak) })
    let b = Rc::new(Cons(10, RefCell::new(Weak::new())));
    if let Some(link) = b.tail() {
        *link.borrow_mut() = Rc::downgrade(&a);
    }
  // 2, a strong count = 1, weak count = 1
	// 2, b strong count = 1, weak count = 0
	// 2, b tail = Some(RefCell { value: (Weak) })
    if let Some(link) = a.tail() {
        *link.borrow_mut() = Rc::downgrade(&b);
    }
  // 3, a strong count = 1, weak count = 1
	// 3, b strong count = 1, weak count = 1
	// 3, a tail = Some(RefCell { value: (Weak) })
}
```

特点：
（1）弱引用通过`Rc::downgrade`传递Rc实例的引用，调用`Rc::downgrade`会得到`Weak<T>`类型的智能指针，同时将weak_count加1（不是将strong_count加1）。
（2）区别在于 `weak_count` 无需计数为 0 就能使 Rc 实例被清理。只要`strong_count`为0就可以了。
（3）可以通过`Rc::upgrade`方法返回`Option<Rc<T>>`对象。

### Mutex

智能指针容器，可变与不可变引用都可以。可以用来编排多核CPU线程任务。

## 内部可变性

组合智能指针：`Rc<Vec<Foo>>`，`Rc<RefCell<Foo>>`, `Arc<Mutex<Foo>>`。

## 比较

`RefCell<T>`/`Rc<T>` 与 `Mutex<T>`/`Arc<T> `的相似性

（1）`Mutex<T>`提供内部可变性，类似于RefCell；

（2）`RefCell<T>`/`Rc<T>`是非线程安全的，而`Mutex<T>`/`Arc<T>`是线程安全的。

# 面向对象

## 对象

结构体、枚举。

## 封装

在Rust中，使用pub关键字来标记模块、类型、函数和方法是公有的，默认情况下一切都是私有的。

## 继承

Rust不支持继承。但是Rust可以通过trait进行行为共享。

## trait对象

1、trait对象动态分发
（1）对泛型类型使用trait bound编译器进行的方式是单态化处理，单态化的代码进行的是静态分发（就是说编译器在编译的时候就知道调用了什么方法）。
（2）使用 trait 对象时，Rust 必须使用动态分发。编译器无法知晓所有可能用于 trait 对象代码的类型，所以它也不知道应该调用哪个类型的哪个方法实现。为此，Rust 在运行时使用 trait 对象中的指针来知晓需要调用哪个方法。

2、trait对象要求对象安全
只有 对象安全（object safe）的 trait 才可以组成 trait 对象。trait的方法满足以下两条要求才是对象安全的：

- 返回值类型不为 Self（例如`Clone`不能作为对象安全的trait对象）
- 方法没有任何泛型类型参数

# 高级特性

## 类型别名

类型别名的主要用途是减少重复。

```rust
type Result<T> = std::result::Result<T, std::io::Error>;//result<T, E> 中 E 放入了 std::io::Error
pub trait Write { 
    fn write(&mut self, buf: &[u8]) -> Result<usize>; 
    fn flush(&mut self) -> Result<()>; 
}
```

## 从不返回的never type

Rust 有一个叫做 `!` 的特殊类型。在类型理论术语中，它被称为 empty type，因为它没有值。我们更倾向于称之为 never type。在函数不返回的时候充当返回值。

```rust
loop { 
        let mut guess = String::new(); 
        io::stdin().read_line(&mut guess) .expect("Failed to read line"); 
        let guess: u32 = match guess.trim().parse() { 
            Ok(num) => num, 
            Err(_) => continue, //continue 的值是 !。
            //当 Rust 要计算 guess 的类型时，它查看这两个分支。
            //前者是 u32 值，而后者是 ! 值。
            //因为 ! 并没有一个值，Rust 决定 guess 的类型是 u32
            };         

        println!("You guessed: {}", guess); 
    } 
```

说明：never type 可以强转为任何其他类型。允许 match 的分支以 continue 结束是因为 continue 并不真正返回一个值；相反它把控制权交回上层循环，所以在 Err 的情况，事实上并未对 guess 赋值。

## 动态类型

动态大小类型（dynamically sized types），有时被称为 “DST” 或 “unsized types”，这些类型允许我们处理只有在运行时才知道大小的类型。

### str

```rust
// 错误代码
// let s1: str = "Hello there!"; 
// let s2: str = "How's it going?";
// 正确代码为：
let s1: &str = "Hello there!"; 
let s2: &str = "How's it going?";
```

&str 则是 两个 值：str 的地址和其长度。这样，&str 就有了一个在编译时可以知道的大小：它是 usize 长度的两倍。也就是说，无论字符串是多大，&str的大小我们总是知道的。
因此，引出动态大小类型的黄金规则：必须将动态大小类型的值置于某种指针之后。如：Box 或 Rc、&str等。

### trait

每一个 trait 都是一个可以通过 trait 名称来引用的动态大小类型。为了将 trait 用于 trait 对象，必须将他们放入指针之后，比如 &Trait 或 Box（Rc 也可以）。

### Sized trait

为了处理 DST，Rust 用Sized trait 来决定一个类型的大小是否在编译时可知。这个 trait 自动为编译器在编译时就知道大小的类型实现。

