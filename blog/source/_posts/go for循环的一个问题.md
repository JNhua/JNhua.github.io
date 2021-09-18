---
title: go for循环的一个问题
date: '2020/11/23 19:48:24'
updated: '2020/11/23 19:48:51'
tags: []
category:
  - go
  - Go
mathjax: true
toc: false
abbrlink: 4d1a6fbb
---
在写go的for循环时，遇到了一个神奇的问题，记录一下。
<!--more-->
# 问题描述
有如下代码：
```go
package main

func main() {
	m := make(map[int] string)
	r := make(map[int]*string)
	m[1] = "1"
	m[2] = "2"

	for k, entry := range m {
		r[k] = &entry
	}

	for _, entry := range r{
		println(*entry)
	}
}

```

打印结果会是什么呢？多跑几次试试。

打印结果：

```bash
1
1
```

或者

```bash
2
2
```

如果是rust或者C++，java的代码，例如rust：

```rust
use std::collections::HashMap;

fn main() {
    let mut m: HashMap<u8, String> = HashMap::new();
    let mut r: HashMap<u8, Box<String>> = HashMap::new();
    m.insert(1, String::from("1"));
    m.insert(2, String::from("2"));

    for (key, value) in m {
        r.insert(key, Box::from(value));
    }

    for (_, v) in r{
        println!("{}", *v);
    }
}
```

打印的结果应该是：

```bash
1 2
// 或者 2 1
```

# 分析

对于所有的 range 循环，Go 语言都会在编译期将原切片或者数组赋值给一个新的变量，在赋值的过程中就发生了拷贝，所以我们遍历的切片已经不是原始的切片变量了。

而遇到这种同时遍历索引和元素的 range 循环时，Go 语言会额外创建一个新的变量存储切片中的元素，**循环中使用的这个变量 entry 会在每一次迭代被重新赋值而覆盖，在赋值时也发生了拷贝**。

因为在循环中获取返回变量的地址都完全相同，所以会发生神奇的现象。所以如果我们想要访问元素所在的地址，不应该直接获取 range 返回的变量地址，而应该使用 `&m[key]` 这种形式。