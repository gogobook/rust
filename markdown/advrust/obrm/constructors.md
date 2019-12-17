# 构造函数

> 原文跟踪[constructors.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/constructors.md) &emsp; Commit: 0e6c680ebd72f1860e46b2bd40e2a387ad8084ad

只有一种方法可以创建用户定义类型的实例：命名它，并立即初始化其所有字段：

```rust
struct Foo {
    a: u8,
    b: u32,
    c: bool,
}

enum Bar {
    X(u32),
    Y(bool),
}

struct Unit;

let foo = Foo { a: 0, b: 1, c: false };
let bar = Bar::X(0);
let empty = Unit;
```

你创建一个类型实例的每一种方式都只是调用一个完全是`vanilla`函数来做一些事情并最终触及`The One True Constructor`。

与C ++不同，Rust没有一大堆内置的构造函数。没有`Copy`，`Default`, `Assignment`, `Move`,或任何构造函数。造成这种情况的原因是多种多样的，但它很大程度上归结为Rust的明确的哲学。

`Move`构造函数在Rust中没有意义，因为我们不允许类型"关心"它们在内存中的位置。每种类型都必须准备好将它盲目地存储到内存中的其他地方。这意味着纯粹在堆栈但仍然可移动的`intrusive linked`根本不会发生在 Rust（safely）中。

类似的赋值和复制构造函数也不存在，因为移动语义是Rust中唯一的语义。最多`x = y`只是将`y`的位移动到`x`变量中。 Rust确实提供了两种用于提供`C ++`面向`copy-oriented`的工具：`Copy`与`Clone`。`Clone`是我们在复制构造函数中的等价物，但它永远不会被隐式调用。您必须在要克隆的元素上显式调用`clone`。复制是`Clone`的一个特例，其实现只是"复制位"。复制类型在移动时会被隐式克隆，但由于`Copy`的定义，这意味着不将旧`copy`视为未初始化 。

虽然Rust提供了一个`Default`特质来指定默认构造函数的等价物，但使用这个特性却极为罕见。这是因为变量未被隐式初始化。默认基本上只对泛型编程有用。在具体的上下文中，类型将为任何类型的"默认"构造函数提供静态`new`方法。这与其他语言中的`new`无关，没有特殊含义。这只是一个命名惯例。
