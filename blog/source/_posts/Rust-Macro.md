---
title: Rust_Macro
copyright: true
date: 2019-09-06 12:32:36
categories:
- Rust
- base
tags:
---

# 编译过程

分词（词条流）→解析（抽象语法树）→简化（高级中间语言）→简化（中级中间语言）→转译（LLVM中间语言）→优化（机器码）

<!-- more -->

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

```rust
#![allow(unused)]
macro_rules! hashmap {
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

利用递归调用消去最后键值对的结尾逗号。第一层调用将其替换为`hashmap!(("a" ) => (1),("b" ) => (2),("c" ) => (3) )`。因为`c=>3`后面没有表达式，所以分隔符 “，”也不再加入。

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

将两个工具宏`unit`和`count`移到`hashmap!`内部。

`#![feature(trace_macros)]`在nightly版本下可以跟踪宏展开（编译过程输出信息，不是调试过程）。

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

在引号的括号要用{% raw %}`{{`{% endraw %}转义，最内部的`{}`是给`args`占位的。

## Bang宏

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

