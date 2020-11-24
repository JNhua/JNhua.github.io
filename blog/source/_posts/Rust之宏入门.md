---
title: Rust之宏入门
date: '2020/10/10 23:51:57'
updated: '2020/11/23 20:01:37'
tags: []
category:
  - Rust
  - Rust基础
mathjax: true
---
# 编译过程
整体流程：[源代码]->分词->[Tokens词条流]->解析->[AST]->语法分析，宏扩展→[高级中间语言HIR]->类型检查->[中级中间语言MIR]->转换->[LLVM IR]->LLVM->[目标文件]->链接->[可执行程序]
<!--more-->

详细过程如下：
1. 解析输入：将`.rs`文件作为输入并进行解析生成`AST`(抽象语法树)；
2. 名称解析，宏扩展和属性配置：解析完毕后处理`AST`，处理`#[cfg]`节点解析路径，扩展宏；
3. 转为HIR：名称解析完毕后将AST转换为`HIR`(高级中间表示)，HIR比AST处理的更多，但是他不负责解析Rust的语法，例如`((1+2)+3)`和`1+2+3`在AST中会保留括号，虽然两者的含义相同但是会被解析成不同的树，但是在HIR中括号节点将会被删除，这两个表达式会以相同的方式表达；
4. 类型检查以及后续分析：处理HIR的重要步骤就是类型检查，例如使用`x.f`时如果我们不知道`x`的类型就无法判断访问的哪个`f`字段，类型检查会创建`TypeckTables`其中包括表达式的类型，方法的解析方式；
5. 转为MIR以及后置处理：完成类型检查后，将HIR转为MIR(中级中间表示)进行借用检查以及优化；
6. 转为LLVM IR和优化：LLVM进行优化，从而生成许多`.o`文件；
7. 链接： 最后将那些`.o`文件链接在一起。

Rust在预处理时，会加入以下代码：

```rust
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::v1::*;
#[macro_use]
extern crate std;
```

从而自动导入标准库。

# 声明宏（Declarative Macro）

宏解析器将宏扩展的时机在解析过程中。

声明宏中可以捕获的类型：

* item，语言项，比如模块、声明、函数定义、类型定义、结构体定义、impl实现等。
* block，代码块，由花括号限定的代码；
* stmt，语句，一般是指以分号结尾的代码；
* expr，表达式，会生成具体的值；
* pat，模式；
* ty，类型；
* ident，标识符；
* path，路径，比如foo、std::iter等；
* meta，元信息，包含在#[...]或者#![...]属性内的信息；
* tt，TokenTree的缩写，词条树；
* vis，可见性，比如pub；
* lifetime，生命周期参数。

词法树的范围比表达式的范围广，比如匹配一个语句块时，就必须用tt。

重复匹配的模式是“$(...) sep rep”，具体的说明如下：

* $(...)，代表要把重复匹配的模式置于其中；
* sep，代表分隔符，常用逗号、分号和=>。这个分隔符可以依据具体的情况忽略。
* rep，代表控制重复次数的标记，目前支持两种：* 和 +，代表“重复零次及以上”和“重复一次及以上”。

展开宏：

```bash
cargo rustc -- -Z unstable-options --pretty=expanded
```

或

```bash
rustc -Z unstable-options --pretty=expanded main.rs
```

```rust
#[macro_export]
macro_rules! my_vec { 
  // x 为重复匹配到的表达式，“，”可以根据情况忽略
    ($($x: expr), *) => {
        {
            let mut temp_vec = Vec::new();
          // 重复匹配到的值在这里访问
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```

## hashmap实现

### 递归调用

```rust
#![allow(unused)]
macro_rules! hashmap {
  // 利用递归调用消去最后键值对的结尾逗号
  // 如果是末尾仍有逗号的，会转换成($($key:expr => $value:expr),*)
    ($($key:expr => $value:expr,)*) =>
        {  hashmap!($($key => $value),*) };
    ($($key:expr => $value:expr),* ) => {
        {
            let mut _map = ::std::collections::HashMap::new();
            $(
                _map.insert($key, $value);
            )*
           _map
       }
   };
}
fn main(){
    let map = hashmap!{
        "a" => 1,
        "b" => 2,
        "c" => 3, 
    };
    assert_eq!(map["a"], 1);
}
```

第一层转换为`hashmap ! ("a" => 1, "b" => 2, "c" => 3)`。

### 重复匹配规则

```rust
macro_rules! hashmap {
    ($($key:expr => $value:expr),* $(,)*) => {
        {
            let mut _map = ::std::collections::HashMap::new();
            $(
                _map.insert($key, $value);
            )*
            _map
        }
   };
}
fn main(){
    let map = hashmap!{
        "a" => 1,
        "b" => 2,
        "c" => 3, 
    };
    assert_eq!(map["a"], 1);
}
```

### 预分配空间

```rust
macro_rules! unit {
    ($($x:tt)*) => (());
}
macro_rules! count {
    ($($key:expr),*) => (<[()]>::len(&[$(unit!($key)),*]));
}
macro_rules! hashmap {
    ($($key:expr => $value:expr),* $(,)*) => {
        {
           let _cap = count!($($key),*);
           let mut _map 
               = ::std::collections::HashMap::with_capacity(_cap);
           $(
               _map.insert($key, $value);
           )*
           _map
       }
   };
}
```

### 消除外部宏

```rust
#![feature(trace_macros)]
macro_rules! hashmap {
    (@unit $($x:tt)*) => (());
    (@count $($rest:expr),*) => 
        (<[()]>::len(&[$(hashmap!(@unit $rest)),*]));
    ($($key:expr => $value:expr),* $(,)*) => {
        {
            let _cap = hashmap!(@count $($key),*);
            let mut _map = 
                ::std::collections::HashMap::with_capacity(_cap);
           $(
               _map.insert($key, $value);
           )*
           _map
       }
   };
}
fn main(){
   trace_macros!(true);
   let map = hashmap!{
       "a" => 1,
       "b" => 2,
       "c" => 3, 
   };
   assert_eq!(map["a"], 1);
}
```

`#![feature(trace_macros)]`在nightly版本下可以跟踪宏展开，在需要展开宏的地方使用`trace_macros!(true);`打开跟踪。

## 导入/导出

`#[macro_export]`表示下面的宏定义对其他包也是可见的。`#[macro_use]`可以导入宏。

在宏定义中使用`$crate`，可以在被导出时，让编译器根据上下文推断包名，避免依赖问题。

# 过程宏

过程宏的Cargo.toml中，设置lib类型：

```toml
[lib]
proc_macro = true
```

## 自定义派生属性

derive属性，自动为结构体或枚举类型进行语法扩展。可以使用TDD（测试驱动开发）的方式来开发。

包结构：

```
|-Cargo.toml
|-src
   	|- lib.rs
|-tests
	|- test.rs
```

先写test.rs

```rust
#[macro_use]
extern crate my_proc_macro;

#[derive(A)]
struct A;
#[test]
fn test_derive_a(){
    assert_eq!("hello from impl A".to_string(), A.a());
}
```

再写lib.rs

```rust
extern crate proc_macro;
use self::proc_macro::TokenStream;

// 自定义派生属性
#[proc_macro_derive(A)]
pub fn derive(input: TokenStream) -> TokenStream {
    let _input = input.to_string();
    assert!(_input.contains("struct A;"));
    r#"
        impl A {
            fn a(&self) -> String{
               format!("hello from impl A")
           }
       }
   "#.parse().unwrap()
}
```

## 自定义属性

可以说自定义派生属性是自定义属性的特例。例如条件编译属性`#[cfg()]`和测试属性`#[test]`都是自定义属性。

test.rs

```rust
use my_proc_macro::attr_with_args;
#[attr_with_args("Hello, Rust!")]
fn foo(){}
#[test]
fn test_foo(){
    assert_eq!(foo(), "Hello, Rust!");
}
```

lib.rs

```rust
extern crate proc_macro;
use self::proc_macro::TokenStream;

#[proc_macro_attribute]
pub fn attr_with_args(args: TokenStream, input: TokenStream)
                      -> TokenStream {
    let args = args.to_string();
    let _input = input.to_string();
    format!("fn foo() -> &'static str {% raw %}{{ {} }}{% endraw %}", args)
        .parse().unwrap()
}
```

在引号的括号要用`{% raw %}{{ {} }}{% endraw %}`转义，最内部的`{}`是给`args`占位的。

## Bang宏/类函数宏

test.rs

```rust
#![feature(proc_macro_hygiene)]
use my_proc_macro::hashmap;
#[test]
fn test_hashmap(){
    let hm = hashmap!{ "a":1,"b":2,};
    assert_eq!(hm["a"],1);
    let hm = hashmap!{"a"=>1,"b"=>2,"c"=>3};
    assert_eq!(hm["c"],3);
}
```

lib.rs

```rust
#[proc_macro]
pub fn hashmap(input: TokenStream) -> TokenStream {
    // 转换input为字符串
    let _input = input.to_string();
    // 将input字符串结尾的逗号去掉，否则在下面迭代中将报错
    let input = _input.trim_end_matches(',');
    // 用split将字符串分割为slice，然后用map去处理
    // 为了支持「"a" : 1」或 「"a" => 1」这样的语法
    let input: Vec<String> = input.split(",").map(|n| {
        let mut data = if n.contains(":") {  n.split(":") }
                       else { n.split(" => ") };
        let (key, value) =
           (data.next().unwrap(), data.next().unwrap());
       format!("hm.insert({}, {})", key, value)
    }).collect();
    let count: usize = input.len();
    let tokens = format!("
        {% raw %}{{
        let mut hm =
            ::std::collections::HashMap::with_capacity({});
            {}
            hm
        }}{% endraw %}", count,
        input.iter().map(|n| format!("{};", n)).collect::<String>()
    );
    // parse函数会将字符串转为Result<TokenStream>
    tokens.parse().unwrap()
}
```

## 第三方包

使用`syn`,`quote`和`proc_macro`可以实现自定义派生属性功能。

以下代码实现了派生属性`New`。

```rust
extern crate proc_macro;
use {
    syn::{Token, DeriveInput, parse_macro_input},
    quote::*,
    proc_macro2,
    self::proc_macro::TokenStream,
};

#[proc_macro_derive(New)]
pub fn derive(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);
    let result = match ast.data {
        syn::Data::Struct(ref s) => new_for_struct(&ast, &s.fields),
        _ => panic!("doesn't work with unions yet"),
    };
    result.into()
}

fn new_for_struct(ast: &syn::DeriveInput,fields: &syn::Fields) -> proc_macro2::TokenStream
{
    match *fields {
        syn::Fields::Named(ref fields) => {
            new_impl(&ast, Some(&fields.named), true)
        },
        syn::Fields::Unit => {
            new_impl(&ast, None, false)
        },
        syn::Fields::Unnamed(ref fields) => {
            new_impl(&ast, Some(&fields.unnamed), false)
        },
    }
}

fn new_impl(ast: &syn::DeriveInput,
            fields: Option<&syn::punctuated::Punctuated<syn::Field, Token![,]>>,
            named: bool) -> proc_macro2::TokenStream
{
    let struct_name = &ast.ident;

    let unit = fields.is_none();
    let empty = Default::default();

    let fields: Vec<_> = fields.unwrap_or(&empty)
        .iter()
        .enumerate()
        .map(|(i, f)| FieldExt::new(f, i, named)).collect();

    let args = fields.iter().map(|f| f.as_arg());
    let inits = fields.iter().map(|f| f.as_init());

    let inits = if unit {
        quote!()
    } else if named {
        quote![ { #(#inits),* } ]
    } else {
        quote![ ( #(#inits),* ) ]
    };


    let (impl_generics, ty_generics, where_clause) = ast.generics.split_for_impl();
    let (new, doc) = (
        syn::Ident::new("new", proc_macro2::Span::call_site()),
        format!("Constructs a new `{}`.", struct_name)
    );
    quote! {
        impl #impl_generics #struct_name #ty_generics #where_clause {
            #[doc = #doc]
            pub fn #new(#(#args),*) -> Self {
                #struct_name #inits
            }
        }
    }
}

struct FieldExt<'a> {
    ty: &'a syn::Type,
    ident: syn::Ident,
    named: bool,
}
impl<'a> FieldExt<'a> {
    pub fn new(field: &'a syn::Field, idx: usize, named: bool) -> FieldExt<'a> {
        FieldExt {
            ty: &field.ty,
            ident: if named {
                field.ident.clone().unwrap()
            } else {
                syn::Ident::new(&format!("f{}", idx), proc_macro2::Span::call_site())
            },
            named: named,
        }
    }
    pub fn as_arg(&self) -> proc_macro2::TokenStream {
        let f_name = &self.ident;
        let ty = &self.ty;
        quote!(#f_name: #ty)
    }

    pub fn as_init(&self) -> proc_macro2::TokenStream {
        let f_name = &self.ident;
        let init =  quote!(#f_name);
        if self.named {
            quote!(#f_name: #init)
        } else {
            quote!(#init)
        }
    }
}
```

# 参考

[使用Rust开发编译系统(C以及Rust编译的过程)](https://blog.csdn.net/qq_41698827/article/details/104055493)