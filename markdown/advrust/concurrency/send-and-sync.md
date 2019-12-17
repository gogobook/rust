# `Send`和`Sync`

> 原文跟踪[send-and-sync.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/send-and-sync.md) &emsp; Commit: b0275ab6fd268e470114216f8b0b83481814c713

并非一切都遵循继承的可变性。 某些类型允许您在变异时，在内存中有多个别名的位置。 除非这些类型使用同步来管理这种访问，否则它们绝对不是线程安全的。 Rust通过`Send`和`Sync`特质做到这一点。

* 如果可以安全地将类型发送到另一个线程，则类型为`Send`。
* 如果在线程之间共享是安全的，则类型为`Sync`（`＆T`为`Send`）。

`Send`和`Sync`是Rust的并发故事的基础。 因此，存在大量特殊工具以使其正常工作。 首先和最重要的是，它们是**unsafe traits**。 这意味着它们是不安全实现，和其他不安全的代码可以假设它们是正确的实现。 因为它们是**标记特质**（它们没有相关的项，如方法），正确实施只是意味着它们具有内在的实现者应具有的属性。 错误地实现`Send`和`Sync`可以导致未定义的行为。

`Send`和`Sync`也是自动派生的特质。 这意味着，与其他所有特质不同，如果类型完全由`Send`和`Sync`类型组成，则它是`Send`或`Sync`。 几乎所有原语都是`Send`和`Sync`，因此几乎所有与之交互的类型都是`Send`和`Sync`。

主要例外包括：

* 原始指针既不是`Send`也不是`Sync`（因为它们没有安全防护）。
* `UnsafeCell`不是`Sync`（因此`Cell`和`RefCell`也不是）。
* `Rc`不是`Send`或`Sync`（因为`refcount`是共享和不同步的）。

`Rc`和`UnsafeCell`基本上不是线程安全的：它们启用了不同步的共享可变状态。 然而，严格来说，原始指针被标记为线程不安全，因为更多的是*lint*。 对原始指针执行任何有用的操作都需要取消引用它，这已经是不安全的。 从这个意义上讲，人们可以争辩说，将它们标记为线程安全是"很好的"。

但是，重要的是它们不是线程安全的，以防止包含它们的类型被自动标记为线程安全。 这些类型具有非普通的未经跟踪的所有权，并且他们的作者不太可能思考线程安全性。 在`Rc`的情况下，我们有一个很好的例子，其中包含一个绝对不是线程安全的`* mut`。

如果需要，非自动派生的类型可以简单地实现它们：

```rust
struct MyBox(*mut u8);

unsafe impl Send for MyBox {}
unsafe impl Sync for MyBox {}
```

在非常罕见的情况下，类型被不适当地自动派生为`Send`或`Sync`，那么人们也可以实现`Send`和`Sync`：

```rust
#![feature(optin_builtin_traits)]

// I have some magic semantics for some synchronization primitive!
struct SpecialThreadToken(u8);

impl !Send for SpecialThreadToken {}
impl !Sync for SpecialThreadToken {}
```

请注意，它本身不可能错误地导出`Send`和`Sync`。 只有被其他不安全代码赋予特殊含义的类型才可能因错误`Send`或`Sync`而导致问题。

原始指针的大多数用法应该封装在足够的抽象之后，可以导出`Send`和`Sync`。 例如，Rust的所有标准集合都是`Send`和`Sync`（当它们包含`Send`和`Sync`类型时），尽管它们普遍使用原始指针来管理分配和复杂的所有权。 类似地，进入这些集合的大多数迭代器都是`Send`和`Sync`，因为它们在很大程度上表现为集合中的`＆`或`＆mut`。
