# 所有权与生命周期

> 原文跟踪[ownership.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/ownership.md) &emsp; Commit: 0e6c680ebd72f1860e46b2bd40e2a387ad8084ad

所有权是Rust的突破性特征。 它允许Rust完全内存安全且高效，同时避免垃圾回收。 在详细介绍所有权制度之前，我们将考虑这种设计的动机。

我们假设您接受垃圾收集（GC）并不总是最佳解决方案，并且希望在某些上下文中手动管理内存。 如果你不接受这个，你可能会对另一种语言对感兴趣？

无论您对GC的感受如何，显然对于使代码安全无疑是一个巨大的好处。 你永远不必担心过早的事情会消失（尽管你是否仍然想要指出那件事是另一个问题......）。 这是C和C ++程序需要处理的普遍问题。 考虑一下这个简单的错误，即我们所有使用非GC语言的人都曾在某个方面做过：

```rust
fn as_str(data: &u32) -> &str {
    // compute the string
    let s = format!("{}", data);

    // OH NO! We returned a reference to something that
    // exists only in this function!
    // Dangling pointer! Use after free! Alas!
    // (this does not compile in Rust)
    &s
}
```

这正是Rust的所有权系统要解决的问题。 Rust知道`＆s`的生存范围，因此可以防止它逃逸。 然而，这是一个简单的案例，即使是C编译器也可以合理地捕获。 随着代码变得越来越大，指针通过各种函数被提供，事情变得越来越复杂。 最终，C编译器将崩溃，并且无法执行足够的转义分析来证明您的代码不健全。 因此，在假设它是正确的情况下，它将被迫接受您的程序。

这将永远不会发生在Rust。 程序员可以向编译器证明一切都是正确的。

当然，Rust关于所有权的故事要比仅仅验证引用不会超出其引用范围复杂。 那是因为确保指针始终有效要比这复杂得多。 例如，在此代码中，

```rust
let mut data = vec![1, 2, 3];
// get an internal reference
let x = &data[0];

// OH NO! `push` causes the backing storage of `data` to be reallocated.
// Dangling pointer! Use after free! Alas!
// (this does not compile in Rust)
data.push(4);

println!("{}", x);
```

天真的范围分析不足以防止这个错误，因为数据确实存在，只要我们需要。 然而，当我们引用它时它被改变了。 这就是Rust要求任何引用来冻结引用对象及其所有者的原因。
