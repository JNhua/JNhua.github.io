---
title: Rust_asynchronous
copyright: true
date: 2020-08-18 14:12:47
categories:
- Rust
- base
tags:
---

# 异步入门

## 为什么需要异步？

*异步*操作是在非阻塞方案中执行的操作，允许主程序流继续处理。

<!-- more -->

假设需求场景为客户端从多个服务器下载多个文件。

| 下载方式     | 缺点                                                 |
| :----------- | :--------------------------------------------------- |
| 依次按照顺序 | 必须等待前一个完成                                   |
| 多线程       | 为每一个下载任务创建线程，导致内存占满               |
| 线程池       | 发起下载请求后，需要等待服务端的响应，当前线程会阻塞 |

注意：多线程和线程池都可以是异步的一种实现方式。异步是和同步相对的概念。

## 异步的优缺点

因为异步操作无须额外的线程负担，并且使用回调的方式进行处理，在设计良好的情况下，处理函数可以不必使用共享变量（即使无法完全不用，最起码可以减少共享变量的数量），减少了死锁的可能。当然异步操作也并非完美无暇。编写异步操作的复杂程度较高，程序主要使用回调方式进行处理，与普通人的思维方式有些初入，而且难以调试。

## 简单使用

```rust
use async_std::task;
// use std::thread::sleep;
use std::time::Duration;
use futures::{ executor};

async fn learn_song() {
    //sleep(Duration::from_secs(5));
    task::sleep(Duration::from_secs(1)).await;
    println!("learn song");
}

async fn sing_song() {
    println!("sing song");
}

async fn dance() {
    println!("dance");
}

async fn learn_and_sing_song() {
    learn_song().await;
    sing_song().await;
}

async fn async_main() {
    let f1 = learn_and_sing_song();
    let f2 = dance();
    futures::join!(f1, f2);
}

fn main() {
    executor::block_on(async_main());
}
```

执行结果

```bash
dance
learn song
sing song
```

说明：

1. 如果使用`sleep(Duration::from_secs(5))`，结果会是按照顺序执行。因为外部的阻塞不能主动唤醒异步内部的线程，所以直接在外部进行阻塞。如果使用异步的`sleep`，`learn song`会让出资源；
2. 通过join，能等待多个Future完成，并发执行；
3. `.await`是在代码块中按顺序执行，会阻塞后面的代码，但是此时会让出线程；`block_on`会阻塞直到Future执行完成。

# Future

## 标准库定义

```rust
use crate::marker::Unpin;
use crate::ops;
use crate::pin::Pin;
use crate::task::{Context, Poll};

pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

impl<F: ?Sized + Future + Unpin> Future for &mut F { // F 的可变引用实现 Future
    type Output = F::Output;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        F::poll(Pin::new(&mut **self), cx)
    }
}

impl<P> Future for Pin<P> // 为 Pin<P=Unpin + ops::DerefMut<Target: Future>> 实现Future
where
    P: Unpin + ops::DerefMut<Target: Future>,
{
    type Output = <<P as ops::Deref>::Target as Future>::Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        Pin::get_mut(self).as_mut().poll(cx)
    }
}
```

`poll`函数传递一个 `&mut Context<'_>` 类型参数， 返回一个 `Poll` 类型参数：

- Context 主要包含一个 `Waker` 对象，由执行器提供，用于告诉执行器，重新执行当前 `poll` 函数

  ```rust
  #[stable(feature = "futures_api", since = "1.36.0")]
   // Context的生命周期不会比它包含的waker引用更久
  pub struct Context<'a> {
      waker: &'a Waker,
      // Ensure we future-proof against variance changes by forcing
      // the lifetime to be invariant (argument-position lifetimes
      // are contravariant while return-position lifetimes are
      // covariant).
      _marker: PhantomData<fn(&'a ()) -> &'a ()>,
  }
  ```

- Poll 是一个枚举类型包含两个枚举

  - `Ready<Output>` 当任务已经就绪，返回该对象
  - `Pending` 任务没有就绪时返回该对象，此Future将让出CPU，直到在其他线程或者任务执行调用`Waker`为止

- 实现者需要保证 poll 是非阻塞，如果是阻塞的话会导致循环进行不下去

实现一个 Future 类型的方式

- 方式1：使用 `async fn`，编译器会自动生成实现 Future Trait的类型
- 方式2：自定义结构体，并实现 Future Trait

# Pin

默认情况下，Rust中所有类型都是可以 `move` 的，Rust允许按值传递所有类型，并且像 `Box<T>` 、`&mut T` 之类的智能指针或者引用允许你通过 `mem::swap` 进行拷贝交换（移动），这样，如果存在结构体存在自引用，将导致引用失效。

而 async 编译后的结构可能就会出现一种自引用的结构，如下所示：

```rust
async {
    let mut x = [0; 128];
    let read_into_buf_fut = read_into_buf(&mut x);
    read_into_buf_fut.await;
    println!("{:?}", x);

}

// 编译后的伪代码如下
// 这是 最外层的 async {}
// struct AsyncFuture {
//    x: [u8; 128],
//    read_into_buf_fut: ReadIntoBuf<'what_lifetime?>,
// }

// 这是 read_into_buf_fut 的Future
// struct ReadIntoBuf<'a> {
//    buf: &'a mut [u8], // points to `x` below
// }
```

这样 AsyncFuture 构造出来后，就存在自引用（`AsyncFuture.read_into_buf_fut.buf` 指向 `AsyncFuture.x`）。但是如果AsyncFuture发生移动，x肯定也会发生移动，如果`read_into_buf_fut.buf`还是指向原来的值的话，则会变成无效。而Pin就是为了解决此问题的。

Pin 类型包着指针类型，保证指针背后的值将不被移动。例如 `Pin<&mut T>`，`Pin<&T>`， `Pin<Box<T>>` 都保证 `T` 不会移动（move）。

## 原理

```rust
pub struct Pin<P> {
    pointer: P,
}
```

- 首先 `Pin<T>` 和 `Box<T>` 类似都是一种智能指针。不同点在于`Pin<&mut T>`不能通过safe代码拿到`&mut T`，因此保证`mem::swap`无法调用，也就是`P`所指向的`T`在内存中固定住，不能移动。

  -  `Pin::as_mut` 返回的仍是 `Pin<T>`

  - 只有 `Pin<DerefMut<T: Unpin>>` 或者 `Pin<Deref<T: Unpin>>` 或者 `Pin<T: Unpin>` 可以通过 `get_mut` 或者 `get_ref` 拿到 `T` 的引用

  - `Pin::new` 只能是针对实现了 `Unpin` 的类型（重要）

    ```rust
    impl<P: Deref<Target: Unpin>> Pin<P> {
        /// Construct a new `Pin<P>` around a pointer to some data of a type that
        /// implements [`Unpin`].
        ///
        /// Unlike `Pin::new_unchecked`, this method is safe because the pointer
        /// `P` dereferences to an [`Unpin`] type, which cancels the pinning guarantees.
        #[stable(feature = "pin", since = "1.33.0")]
        #[inline(always)]
        pub fn new(pointer: P) -> Pin<P> {
            // Safety: the value pointed to is `Unpin`, and so has no requirements
            // around pinning.
            unsafe { Pin::new_unchecked(pointer) }
        }
    
        /// Unwraps this `Pin<P>` returning the underlying pointer.
        ///
        /// This requires that the data inside this `Pin` is [`Unpin`] so that we
        /// can ignore the pinning invariants when unwrapping it.
        #[stable(feature = "pin_into_inner", since = "1.39.0")]
        #[inline(always)]
        pub fn into_inner(pin: Pin<P>) -> P {
            pin.pointer
        }
    }
    ```

    只有`P<T>`的`T: Unpin`，才可以new出一个`Pin<P<T>>`。这里的T就是应该被pin的实例，可是由于`T: Unpin`实际上T的实例并不会被pin。

  - 如果要创建不实现`Unpin`的`Pin<P<T>>`，可以使用`unsafe{ Pin::new_unchecked(&mut t) }`。

- 本质上实现不移动就是加了一层指针，并未违反任意值都是可以移动的规则。

  - 比如 `Pin<T>` 发生移动时，仅仅是 `Pin` 这个结构发生了移动，但是 `T` 对象并没有移动

## Unpin

```rust
pub auto trait Unpin {}
```

定义在`std::marker`中，如果`T: Unpin`，那么`T`在`pin`后可以安全地移动，可以拿到`&mut T`。`Unpin`只对`Pin<P<T>>`的`T`起作用，不对`P`本身起效，例如对`Pin<Box<T>>`的`Box<T>`是无效的。

默认为以下类型实现了`Unpin`：

```rust
impl<'a, T: ?Sized + 'a> Unpin for &'a T {}
impl<'a, T: ?Sized + 'a> Unpin for &'a mut T {}
impl<T: ?Sized> Unpin for *const T {}
impl<T: ?Sized> Unpin for *mut T {}
```

`async`生成的匿名结构体（`impl Future<Output=()>`）没有实现`Unpin`。

### !Unpin

对Unpin取反，!Unpin的双重否定就是pin。如果一个类型中包含了PhantomPinned，那么这个类型就是!Unpin。

```rust
#[derive(Debug, Copy, Clone, Eq, PartialEq, Ord, PartialOrd, Hash)]
pub struct PhantomPinned;
impl !Unpin for PhantomPinned {}
```

一般在结构体中使用`_marker: PhantomPinned`来实现`!Unpin`。

## 完整示例

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;
use std::mem;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned
}

impl Test {
    fn new(txt: &str) -> Self {
        //此指针指向栈对象， 必须慎重考虑其生命长短，避免出现`悬指针`。
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned
        }
    }
    fn init<'a>(self: Pin<&'a mut Self>) {
        let self_ptr: *const String = &self.a;
        let this = unsafe { self.get_unchecked_mut() };
        this.b = self_ptr;
    }

    fn a<'a>(self: Pin<&'a Self>) -> &'a str {
        &self.get_ref().a
    }
    fn b<'a>(self: Pin<&'a Self>) -> &'a String {
        unsafe { &*(self.b) }
    }
}

fn main() {
// test1 is safe to move before we initialize it
    let mut test1 = Test::new("test1");
// Notice how we shadow `test1` to prevent it from being accessed again
//同名的新指针变量屏蔽了原来的test1, 以此确保只能通过Pin来访问到Test.
//这样确保不可能再访问到旧test1指针！
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());

    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));

//swap导致编译错误， 因为Pin实质上就是禁止获得&mut T引用(指针) ，
//无法获得&mut T指针，则无法Move , 比如：swap等。
//之所以用Pin 包裹原来的裸指针，目的就是禁止获取到：&mut T.
//     std::mem::swap(test1.get_mut(), test2.get_mut());

    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}
// 结果
// a: test1, b: test1
// a: test2, b: test2

fn main() {
    let mut test1 = Test::new("test1");
    let mut test1_pin = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1_pin.as_mut());
    drop(test1_pin); //Pin指针被提前drop , 因为test1未被遮蔽， 后面代码仍然可以访问到， 但是test1已被析构

    let mut test2 = Test::new("test2");
    mem::swap(&mut test1, &mut test2);
    println!("Not self referential anymore: {:?}", test1.b); //test1.b == 0x00 ， Pin析构时析构了test1 所指的Test Struct, 其内部指针归0，
//所以说不再是自引用。
}
```

如果使用`Box::pin(t)`创建`Pin<Box<T>>`，则会将数据固定到堆上。

```rust
impl Test {
    fn new(txt: &str) -> Pin<Box<Self>> {
     let t = Test {
         a: String::from(txt),
         b: std::ptr::null(),
         _marker: PhantomPinned,
     };
     //Constructs a new Pin<Box<T>>. If T does not implement Unpin, then x will be pinned in memory and unable to be moved.
     let mut boxed = Box::pin(t);
     let self_ptr: *const String = &boxed.as_ref().a;
      // boxed.as_mut() -> Pin<&mut Test>
      // boxed.as_mut().get_unchecked_mut() -> &mut Test
     unsafe { boxed.as_mut().get_unchecked_mut().b = self_ptr };
     boxed
 }
}
fn main() {
    let mut test1 = Test::new("test1");
    let mut test2 = Test::new("test2");

    println!("a: {}, b: {}", test1.as_ref().a(), test1.as_ref().b());
    std::mem::swap(&mut test1, &mut test2);
    println!("a: {}, b: {}", test2.as_ref().a(), test2.as_ref().b()); 
  // 结果
  // a: test1, b: test1
	// a: test1, b: test1
}
```

为什么堆上的`pin`对象可以进行`swap`？

`boxed`只是一个栈变量，所指的对象在堆上。通过`swap`仅仅`bitcopy and swap`两个栈变量`test1`和`test2`，相当于两者交换了所有权，交换了指向，而堆上的数据不受影响。`T`类型对象内存位置固定，所有没有违反`Pin`的语义要求。

# async/await

async转化的Future对象和其它Future一样是具有惰性的，即在运行之前什么也不做。运行Future最常见的方式是`.await`。

## async

有如下代码：

```rust
async fn async_main() {
    let f1 = async_function1();
    let f2 = async_function2();

    let f = async move {
        f1.await;
        f2.await;
    };
    f.await;
}
```

那么实际上会生成一个匿名的`Future trait object`，包裹一个 `Generator`。也就是一个实现了 `Future` 的 `Generator`。`Generator`实际上是一个状态机，配合`.await`当每次`async` 代码块中任何返回 `Poll::Pending`则即调用`generator yeild`，让出执行权，一旦恢复执行，`generator resume` 继续执行剩余流程。

```rust
pub const fn from_generator<T>(gen: T) -> impl Future<Output = T::Return>
where
    T: Generator<ResumeTy, Yield = ()>,
{
    #[rustc_diagnostic_item = "gen_future"]
    struct GenFuture<T: Generator<ResumeTy, Yield = ()>>(T);

    // We rely on the fact that async/await futures are immovable in order to create
    // self-referential borrows in the underlying generator.
    impl<T: Generator<ResumeTy, Yield = ()>> !Unpin for GenFuture<T> {}

    impl<T: Generator<ResumeTy, Yield = ()>> Future for GenFuture<T> {
        type Output = T::Return;
        fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
            // Safety: Safe because we're !Unpin + !Drop, and this is just a field projection.
            let gen = unsafe { Pin::map_unchecked_mut(self, |s| &mut s.0) };

            // Resume the generator, turning the `&mut Context` into a `NonNull` raw pointer. The
            // `.await` lowering will safely cast that back to a `&mut Context`.
            match gen.resume(ResumeTy(NonNull::from(cx).cast::<Context<'static>>())) {
                GeneratorState::Yielded(()) => Poll::Pending,
                GeneratorState::Complete(x) => Poll::Ready(x),
            }
        }
    }

    GenFuture(gen)
}
```

每一次`gen.resume(）`会顺序执行`async block`中代码直到遇到`yield`。`async block`中的`.await`语句在无法立即完成时会调用`yield`交出控制权等待下一次`resume`。而当所有代码执行完，也就是状态机入`Complete`，`async block`返回`Poll::Ready`，代表`Future`执行完毕。

生成的匿名对象类似如下：

```rust
struct AsyncFuture {
    fut_one: FutFunction1,
    fut_two: FutFunction2,
    state: State,
}

//state的定义可能如下
enum State {
    AwaitingFutFunction1,
    AwaitingFutFunction2,
    Done,
}
```

`poll`进行轮询每个`Future`的状态，如果`Poll::Ready()`则进入下一个`State`，直到`Done`。

## 生命周期

```rust
async fn foo(x: &u8) -> u8 { *x }
fn good() -> impl Future<Output = ()> {
    async {
        let x = 5;
        foo(&x).await;
    }
}
```

通过将`x`移动到`async`中，延长`x`的生命周期和`foo`返回的Future生命周期一致。

## move

async 块和闭包允许 move 关键字，就像普通的闭包一样。一个 async move 块将获取它引用变量的所有权，允许它活得比目前的范围长，但放弃了与其它代码分享那些变量的能力。

## 线程间移动

在使用多线程Future的excutor时，Future可能在线程之间移动，因此在async主体中使用的任何变量都必须能够在线程之间传输，因为任何.await变量都可能导致切换到新线程。

async fn Future是否为Send的取决于是否在.await点上保留非Send类型。编译器尽其所能地估计值在.await点上的保存时间。

```rust
async fn foo() {
    {
        let x = Rc<()>; // Rc<T>不可Send
    }
    bar().await;
}
```

如果`x`不在`await`前drop，那么该Future是非Send的。而Future的异步特性要求需要有`Send`约束。

# Stream

`Stream`是由一系列的`Future`组成，我们可以从`Stream`读取各个`Future`的结果，直到`Stream`结束。

## 定义

```rust
trait Stream {
    type Item;

    fn poll_next(self: Pin<&mut Self>, lw: &LocalWaker)
        -> Poll<Option<Self::Item>>;
}
```

`poll_next`函数有三种可能的返回值，分别如下：

- `Poll::Pending` 说明下一个值还没有就绪，仍然需要等待。
- `Poll::Ready(Some(val))` 已经就绪，成功返回一个值，程序可以通过调用`poll_next`再获取下一个值。
- `Poll::Ready(None)` 表示`Stream`已经结束，不应该在调用`poll_next`。

## 迭代

`Stream`不支持使用`for`，而`while let`和 `next/try_next`则是允许的。

```rust
async fn sum_with_next(mut stream: Pin<&mut dyn Stream<Item = i32>>) -> i32 {
    use futures::stream::StreamExt; // for `next`
    let mut sum = 0;
    while let Some(item) = stream.next().await {
        sum += item;
    }
    sum
}

async fn sum_with_try_next(mut stream: Pin<&mut dyn Stream<Item = Result<i32, io::Error>>>
) -> Result<i32, io::Error> {
    use futures::stream::TryStreamExt; // for `try_next`
    let mut sum = 0;
    while let Some(item) = stream.try_next().await? {
        sum += item;
    }
    Ok(sum)
}
```

## 并发

```rust
async fn jump_around(
    mut stream: Pin<&mut dyn Stream<Item = Result<u8, io::Error>>>,
) -> Result<(), io::Error> {
    use futures::stream::TryStreamExt; // for `try_for_each_concurrent`
    const MAX_CONCURRENT_JUMPERS: usize = 100;

    stream.try_for_each_concurrent(MAX_CONCURRENT_JUMPERS, |num| async move {
        jump_n_times(num).await?;
        report_n_jumps(num).await?;
        Ok(())
    }).await?;

    Ok(())
}
```

# Select

select宏也允许并发的执行Future，但是和join、try_join不同的是，select宏只要有一个Future返回，就会返回。

```rust
use futures::{select, future::FutureExt, pin_mut};
use tokio::runtime::Runtime;
use std::io::Result;

async fn function1() -> Result<()> {
    tokio::time::delay_for(tokio::time::Duration::from_secs(10)).await;
    println!("function1 ++++ ");
    Ok(())
}

async fn function2() -> Result<()> {
    println!("function2 ++++ ");
    Ok(())
}

async fn async_main() {
  // Fuse：基础迭代器一次返回None后，就一直返回None
    let f1 = function1().fuse();
    let f2 = function2().fuse();

  // 在栈上pin
    pin_mut!(f1, f2);

    select! {
        _ = f1 => println!("task one completed first"),
        _ = f2 => println!("task two completed first"),
    }
}

fn main() {
    let mut runtime = Runtime::new().unwrap();
    runtime.block_on(async_main());
    println!("Hello, world!");
}
```

# BoxFuture

一个拥有的动态类型[' Future ']，在不是静态输入或需要添加一些间接类型的情况下使用。

比如在递归使用Future时：

```rust
use futures::future::{BoxFuture, FutureExt};

fn re() -> BoxFuture<'static, ()> {
	async move{
		re().await;
		re().await;
	}.boxed()
}

fn main() {
	re();
}

fn boxed<'a>(self) -> BoxFuture<'a, Self::Output>
    where Self: Sized + Send + 'a,
    {
        Box::pin(self)
    }
```



# 参考

1. [Rust异步编程](https://www.rectcircle.cn/posts/rust%E5%BC%82%E6%AD%A5%E7%BC%96%E7%A8%8B)
2. [Rust 异步编程，Pin 介绍](https://zhuanlan.zhihu.com/p/157348723)
3. [Pin Unpin学习笔记](https://rustcc.cn/article?id=1d0a46fa-da56-40ae-bb4e-fe1b85f68751)

