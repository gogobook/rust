# 类型转换

> 原文跟踪[conversions.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/conversions.md) &emsp; Commit: 0e6c680ebd72f1860e46b2bd40e2a387ad8084ad

在一天结束时，一切都只是一堆`bits`，类型系统只是帮助我们正确使用这些`bits`。类型`bits`有两个常见问题：需要将这些确切的`bits`重新解释为不同的类型，并且需要将`bits`更改为具有等效含义的不同类型。因为Rust鼓励在类型系统中编码重要属性，所以这些问题非常普遍。因此，Rust因此为您提供了几种解决方法。

首先，我们将介绍`Safe Rust`为您重新解释值的方法。实现这一目标的最简单方法是将一个值构造成其组成部分，然后从中构建一个新类型。例如

```rust
struct Foo {
    x: u32,
    y: u16,
}

struct Bar {
    a: u32,
    b: u16,
}

fn reinterpret(foo: Foo) -> Bar {
    let Foo { x, y } = foo;
    Bar { a: x, b: y }
}
```

但这令人讨厌。对于常见的转换，Rust提供了更符合人体工程学的替代品。

