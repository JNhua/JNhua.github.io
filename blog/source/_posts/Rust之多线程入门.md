---
title: Rust之多线程入门
date: '2020/10/11 13:53:33'
updated: '2020/11/23 20:07:29'
tags: []
category:
  - Rust
  - Rust基础
mathjax: true
---
# 线程基本使用
## 创建线程
`thread::spawn`，`thread::Builder::spawn`
<!--more-->
```rust
let handle = thread::spawn(|| {
       for i in 1..10 {
           println!("number {} in spawn thread!", i);
           thread::sleep(Duration::from_millis(1));
       }
});

let builder = Builder::new().name(thread_name).stack_size(size);
let child = builder.spawn(move || {
    println!("in child:{}", current().name().unwrap());
}).unwrap();
```
## 等待线程

`JoinHandler<()>.join().unwrap()`

```rust
 handle.join().unwrap();
```

## move闭包

以下情况需要使用`move`：不能判断子线程执行时间，而在子线程中引用了主线程变量。

# Send,Sync

- Send：实现Send的类型可以安全的在线程间传递所有权。
- Sync：实现Sync的类型可以安全的在线程间传递不可变借用。

`spawn`函数的源码

```rust
#[stable(feature = "rust1", since = "1.0.0")]
pub fn spawn<F, T>(f: F) -> JoinHandle<T> where
    F: FnOnce() -> T, F: Send + 'static, T: Send + 'static
{
    Builder::new().spawn(f).expect("failed to spawn thread")
}
```

其参数F和返回值类型T都加上了`Send + 'static`限定，Send表示闭包必须实现Send，这样才可以在线程间传递。而`'static`表示T只能是非引用类型，因为使用引用类型则无法保证生命周期。

# 消息传递

## 通道

### 基本使用

Rust中一个实现消息传递并发的主要工具是通道。通道由两部分组成，一个是发送端，一个是接收端，发送端用来发送消息，接收端用来接收消息。发送者或者接收者任一被丢弃时就可以认为通道被关闭了。

通道介绍
（1）通过`mpsc::channel`，创建通道，`mpsc`是多个生产者，单个消费者；mpsc：Multi-producer, single-consumer FIFO queue communication primitives.
（2）通过`spmc::channel`，创建通道，spmc是一个生产者，多个消费者；
（3）创建通道后返回的是发送者和消费者，示例：
`let (tx, rx) = mpsc::channel(); 	let (tx, rx) = spmc::channel();`

```rust
fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

1、发送者的send方法返回的是一个`Result<T,E>`，如果接收端已经被丢弃了，将没有发送值的目标，此时发送会返回错误。`send`会发生`move`。
2、接受者的recv返回值也是一个Result类型，当通道发送端关闭时，返回一个错误值。
3、接收端这里使用的recv方法，会阻塞到有一个消息到来。我们也可以使用try_recv()，不会阻塞，会立即返回。

### 发送多个消息

```rust
fn main() {
    let (tx, rx) = mpsc::channel();
    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
        ];
        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });
    for recv in rx {
        println!("Got: {}", recv);
    }
}
```

### 多个生产者

```rust
let (tx, rx) = mpsc::channel();
let tx1 = mpsc::Sender::clone(&tx);
let tx2 = mpsc::Sender::clone(&tx);
// ... 分别在各自线程中进行发送
// 接受
for rec in rx {
    println!("Got: {}", rec);
}
```

## 共享内存

### 互斥锁`Mutex<T>`

```rust
fn main() {
    let m = Mutex::new(5);
    {
        let mut num = m.lock().unwrap();
        *num = 6;
    }//离开作用域时，自动释放
    println!("m = {:?}", m);
}
```

（1）`Mutex<T>`是一个智能指针，更准确的说，`lock`调用返回一个叫做`MutexGuard`的智能指针；

```rust
#[stable(feature = "rust1", since = "1.0.0")]
    pub fn lock(&self) -> LockResult<MutexGuard<'_, T>> {
        unsafe {
            self.inner.raw_lock();
            MutexGuard::new(self)
        }
    }
```

（2）内部提供了`Drop`方法，实现当`MutexGuard`离开作用域时自动释放锁。

### `Arc<T>`

`Arc<T>`是一个类似于`Rc<T>`，但是可以安全的用于并发环境的类型。

```rust
// let counter = Mutex::new(0);  // 所有权被移动到线程闭包中，只可用一次

// Arc<Mutex<T>>可以组合成内部可变的线程安全指针
let counter = Arc::new(Mutex::new(0));	// 多线程共享
```

# 简单的WebServer

## 线程池

```rust
use std::sync::mpsc;
use std::sync::Arc;
use std::sync::Mutex;
use std::thread;

struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Message>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let message = receiver.lock().unwrap().recv().unwrap();
            match message {
                Message::NewJob(job) => {
                    println!("worker {} receive a job", id);
                    job();
                },
                Message::Terminate => {
                    println!("Worker {} receive terminate", id);
                    break;
                },
            }
        });

        Worker { 
            id, 
            thread: Some(thread), 
        }
    }
}

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Message>,
}

type Job = Box<dyn FnOnce() + Send + 'static>;
enum Message {
    NewJob(Job),
    Terminate,
}

impl ThreadPool {
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);
        let mut workers = Vec::with_capacity(size);
        let (sender, receiver) = mpsc::channel();
        let receiver = Arc::new(Mutex::new(receiver)); //线程安全

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        ThreadPool { workers, sender }
    }

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);
        self.sender.send(Message::NewJob(job)).unwrap();
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        //send terminate message to all workers
        for _ in &mut self.workers {
            self.sender.send(Message::Terminate).unwrap();
        }

        //wait all workers terminate
        for worker in &mut self.workers {
            //wait for worker thread terminate 
            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}
```

## 服务端

```rust
use mylib::ThreadPool;
use std::fs;
use std::io::{Read, Write};
use std::net::{TcpListener, TcpStream};
use std::{thread, time};

fn handle_client(mut stream: TcpStream) {
    let mut buffer = [0; 512];
    stream.read(&mut buffer).unwrap();
    let get = b"GET / HTTP/1.1\r\n";
    let (status_line, filename) = if buffer.starts_with(get) {
        ("HTTP/1.1 200 OK\r\n\r\n", "main.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND\r\n\r\n", "404.html")
    };

    let contents = fs::read_to_string(filename).unwrap();
    let response = format!("{}{}", status_line, contents);

    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();

    let te = time::Duration::from_millis(10000);
    thread::sleep(te); //睡眠一段时间，模拟处理时间很长
}

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080")?;
    let pool = ThreadPool::new(4);

    for stream in listener.incoming().take(2) {
        let stream = stream.unwrap();
        //thread pool
        pool.execute(|| handle_client(stream));
    }

    Ok(())
}
```

