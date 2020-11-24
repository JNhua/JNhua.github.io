---
title: Rust之trait
date: '2020/10/10 23:49:43'
updated: '2020/10/10 23:50:07'
tags: []
category:
  - Rust
  - Rust基础
mathjax: true
abbrlink: 35f77f38
---
# 行为上对类型的约束

<!--more-->
trait是Rust对Ad-hoc（点对点/特别的/临时的）多态的支持。

## 接口抽象

* 接口中可以定义方法，并支持默认实现；

```rust
trait NoiseMaker {
    fn make_noise(&self);
    
    fn make_alot_of_noise(&self){
        self.make_noise();
        self.make_noise();
        self.make_noise();
    }
}
```

* 接口中不能实现另一个接口，但是接口之间可以继承；

```rust
trait NoiseMaker {
    fn make_noise(&self);
}

trait LoudNoiseMaker: NoiseMaker {
    fn make_alot_of_noise(&self) {
        self.make_noise();
        self.make_noise();
        self.make_noise();
    }
}

impl NoiseMaker for SeaCreature {
    fn make_noise(&self) {
        println!("{}", &self.get_sound());
    }
}

impl LoudNoiseMaker for SeaCreature {}
```

* 同一个接口可以同时被多个类型实现，但不能被同一个类型实现多次；

为不同的类型实现trait，属于一种函数重载，也是`Ad-hoc`多态。

### 关联类型

```rust
pub trait Add<RHS = Self> {
	type Output;
	fn add(self, rhs: RHS) -> Self::Output;
}
```

`Self`是每个`trait`都带有的隐式类型参数，代表实现当前`trait`的具体类型。实现时，未指明泛型，默认为`Self`类型。

`Output`为关联类型。

#### 为u32类型实现Add trait

```rust
impl Add for u32 {
    type Output = u32;
    fn add (self, other: u32) -> u32 { self + other }
}
```

#### 关联类型的作用

关联类型在trait定义中指定占位符类型。trait 的实现者会针对特定的实现在这个类型的位置指定相应的具体类型。如此可以定义一个使用多种类型的 trait。

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
// 为什么不像下面这样写，使用泛型？
// pub trait Iterator<T> {
//    fn next(&mut self) -> Option<T>;
// }
```

使用泛型的方式，则如例子中在实现trait的时候必须带上具体的类型，调用时也必须带上具体的类型。

### 孤儿规则

如果要实现某个trait，那么该trait和实现该trait的那个类型至少有一个要在当前crate中定义。

绕开这个限制的方法是使用 newtype 模式（newtype pattern）。

```rust
use std::fmt;
struct Wrapper(Vec<String>);
impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({})", self.0.join(","))
    }
}
```

在上述例子中，我们在 Vec 上实现 Display，而孤儿规则阻止我们直接这么做，因为 Display trait 和 Vec 都定义于我们的 crate 之外。我们可以创建一个包含 Vec 实例的 Wrapper 结构体，然后再实现。

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
// where
  // T: Add<T, Output=T>
    a + b
}
assert_eq!(sum(1u32, 2u32), 3);
assert_eq!(sum(1u64, 2u64), 3);
```

约束`sum`函数，只有实现了`Add`这个trait的类型才可以当做参数。

`where`关键字，可以把泛型中的trait限定移到语句最后。

## 抽象类型

### trait对象

trait的类型大小在编译期间无法确定，所以trait对象必须使用指针。可以利用引用操作符`&`或`Box<T>`来制造一个trait对象。trait对象的结构体包含一个可变data指针和一个可变虚表指针。

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

`impl Trait`只能用于单个参数指定抽象类型。

`dyn Trait`是与`impl Trait`相对应的动态分发。

## 标签trait

5个重要的标签：Sized（编译器可确定大小），Unsize（动态大小），Copy（可以按位复制），Send（跨线程安全通信），Sync（线程间安全共享引用）。

# trait对象的生命周期

* trait对象的生命周期默认是'static；
* 如果实现trait的类型包含&'a X 或者 &'a mut X，则默认生命周期是'a；
* 如果实现trait的类型只有T: 'a，则默认生命周期是'a；
* 如果实现trait的类型包含多个类似T:'a 的从句，则生命周期需要明确指定。

# 动态分发

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
//fn fly_dyn(s: &impl Fly) -> bool {
//    s.fly()
//}

fn main() {
    let pig = Pig;
    assert_eq!(fly_static::<Pig>(pig), false);
    let duck = Duck;
    assert_eq!(fly_static::<Duck>(duck), true);
  // 需要推断具体类型
    assert_eq!(fly_dyn(&Pig), false);
    assert_eq!(fly_dyn(&Duck), true);
}
```

fly_static是静态分发，fly_dyn是动态分发。Trait对象拥有实例对象方法的指针（定义在该Trait中的），类似于C++的虚函数表。

内存细节：动态分发会稍慢，因为要推断类型，去查找真正的函数调用。

# 完全限定语法

## 同名方法

```rust
trait Trait {
    fn foo(&mut self, x: i32);
}

struct Foo;

impl Foo {
    fn foo(&self) {
        println!("Foo::foo");
    }
}

impl Trait for Foo {
    fn foo(&mut self, x: i32) {
        //self.foo(); 　　　//１、出错点１ (&*self).foo();　按照此方式或者Self::foo(self)调用ok
        println!("Trait::foo {}", x);
    }
}

fn main() {
    let mut a: Foo = Foo {};
    a.foo();
    //a.foo(3); //２、出错点２，此方式调用出错 Trait::foo(&mut a, 3);
}
```

Rust在进行方法解析的时候试用的规则比较简单，编译器查看方法“ receiver”（`.` 之前的东西，在本例中为`self`，其类型为`＆mut Foo`），并检查它是否具有称为`foo`的方法。如果没有`foo`方法，则尝试借用或取消引用接收方后，再次检查是否有此方法。编译器会一直重复此过程，直到找到匹配的方法为止。 在此例中，编译器就会匹配到`fn foo(&mut self, x: i32)`方法，但是没有足够的参数，所以按照**出错点１的写法会出错**，正确的方式是显示地调用。
默认会调用`Foo`类型的`foo`方法，那么如果要调用`trait`中的方法怎么办呢？用trait名显示调用即可`Trait::foo(&mut a, 3);`。

## 关联函数的完全限定语法

当不能判断由哪个类型实现的方法来调用，需要使用完全限定语法：

`<Type as Trait>::function(receiver_if_method, next_arg, ...);`

