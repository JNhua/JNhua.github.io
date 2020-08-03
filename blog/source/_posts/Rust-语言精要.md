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
```

`result`等于20。

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



# 内存

Rust 程序有 3 个存放数据的内存区域：

- **数据内存** - 对于固定大小和**静态**（即在整个程序声明周期中都存在）的数据。 例如 “Hello World”字面值常量，该文本的字节只能读取，因此它们位于该区域中。 编译器对这类数据做了很多优化，由于位置已知且固定，因此通常认为编译器使用起来非常快。
- **栈内存** - 对于在函数中声明为变量的数据。 在函数调用期间，内存的位置不会改变，因为编译器可以优化代码，所以栈数据使用起来非常快。
- **堆内存** - 对于在程序运行时创建的数据。 此区域中的数据可以添加、移动、删除、调整大小等。由于它的动态特性，通常认为它使用起来比较慢， 但是它允许更多创造性的内存使用。当数据添加到该区域时，我们称其为**分配**。 从本区域中删除 数据后，我们将其称为**释放**。

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

# 所有权

所有权是Rust的特性。所有权解决了堆栈分配与回收问题。

## 内存分配

| 语言     | 内存回收机制                                                 |
| -------- | ------------------------------------------------------------ |
| 其他语言 | GC会跟踪声明的变量，当它不再被使用时，自动清除。如果没有GC，程序员负责在恰当的时候释放这段申请的内存。 |
| Rust     | 使用RAII(“资源获取即初始化”)，在变量`invalid`时，调用`drop`回收。 |

## 移动

```rust
let s1 = String::from("hello");
let s2 = s1;
```

`s1`移动到了`s2`，不仅仅是`shallow copy`，`s1`还被置为`invalid`了。

Rust不会自动创建“深拷贝”，需要自己用`clone()`。但是，如果实现了`Copy`的`trait`，那么值会被复制入栈。在实现`Copy`时，必须先实现`Drop`。

将所有者作为参数传递给函数时，其所有权将移交至该函数的参数。 在一次**移动**后，原函数中的变量将无法再被使用。在**移动**期间，所有者的堆栈值将会被复制到函数调用的参数堆栈中。

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

# 生命周期

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

# 指针

原生指针：

- `*const T` - 指针常量。
- `*mut T` - 可变指针。

取得指针所指地址内的数据，需要在`unsafe{...}`中，因为不能保证该原生指针指向有效数据。



## 智能指针

### Box

`Box`将数据从栈上移动到堆。

```rust
struct Ocean {
    animals: Vec<Box<dyn NoiseMaker>>,
}
let ocean = Ocean {
        animals: vec![Box::new(ferris), Box::new(sarah)],
};
```

### Rc

引用计数指针，将数据从栈上移动到堆。允许其他`Rc`指针不可变引用同一个数据。

```rust
let heap_pie = Rc::new(Pie);
let heap_pie2 = heap_pie.clone();

heap_pie2.eat();
heap_pie.eat();
// all reference count smart pointers are dropped now
// the heap data Pie finally deallocates
```

### RefCell

一个智能指针容器。可变与不可变引用都可以，引用规则与之前一样。

```rust
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

### Mutex

智能指针容器，可变与不可变引用都可以。可以用来编排多核CPU线程任务。

## 内部可变性

组合智能指针：`Rc<Vec<Foo>>`，`Rc<RefCell<Foo>>`, `Arc<Mutex<Foo>>`。

