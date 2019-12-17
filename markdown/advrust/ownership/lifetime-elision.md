# 生命周期省略

> 原文跟踪[lifetime-elision.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/lifetime-elision.md) &emsp; Commit: 9cc14cd6a46b2e4385f94b86fb9f1281327e8adc

为了使常见的模式更符合人体工程学，Rust允许在函数签名中省略生命周期 。

你可以一个类型写生命周期的任何地方都有一个生命周期的位置：

```rust
&'a T
&'a mut T
T<'a>
```

生命周期位置可以显示为“输入”或“输出”：

* 对于`fn`定义，`input`是指`fn`定义中形式参数的类型，而`output`是指返回结果类型。因此`fn foo(s: &str) -> (&str, &str)`在输入位置省略了一个生命周期，在输出位置省略了两个生命周期。请注意，`fn`方法定义的输入位置不包括方法`impl`声明中出现的生命周期（对于默认方法，也不包括特质声明中出现的生命周期）。

* 将来，应该可以以相同的方式在`impl`声明中实现隐式。

省略规则如下：

- 输入位置中的每个隐式生命周期都有不同的生命周期参数。

- 如果只有一个输入生命周期位置（省略或未省略），则将该生命周期分配给所有省略的输出生命周期。

- 如果存在多个输入生命周期位置，但其中一个是`&self`或 `&mut self`，则将生命周期`self`分配给所有省略的输出生命周期。

- 否则，忽略输出生命周期是错误的。

例子：

```rust
fn print(s: &str);                                      // elided
fn print<'a>(s: &'a str);                               // expanded

fn debug(lvl: usize, s: &str);                          // elided
fn debug<'a>(lvl: usize, s: &'a str);                   // expanded

fn substr(s: &str, until: usize) -> &str;               // elided
fn substr<'a>(s: &'a str, until: usize) -> &'a str;     // expanded

fn get_str() -> &str;                                   // ILLEGAL

fn frob(s: &str, t: &str) -> &str;                      // ILLEGAL

fn get_mut(&mut self) -> &mut T;                        // elided
fn get_mut<'a>(&'a mut self) -> &'a mut T;              // expanded

fn args<T: ToCStr>(&mut self, args: &[T]) -> &mut Command                  // elided
fn args<'a, 'b, T: ToCStr>(&'a mut self, args: &'b [T]) -> &'a mut Command // expanded

fn new(buf: &mut [u8]) -> BufWriter;                    // elided
fn new<'a>(buf: &'a mut [u8]) -> BufWriter<'a>          // expanded

```
