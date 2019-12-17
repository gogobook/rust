# 并发和并行

> 原文跟踪[concurrency.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/concurrency.md) &emsp; Commit: 0e6c680ebd72f1860e46b2bd40e2a387ad8084ad

Rust作为一种语言并不真正对如何进行并发或并行性有任何意见。 标准库公开操作系统线程和阻塞系统调用，因为每个人都有这些，并且它们足够统一，您可以以相对无争议的方式对它们进行抽象。 消息传递，绿色线程和异步API等多种多样的，对它们的任何抽象都倾向于涉及我们不愿意为1.0承诺的权衡。

然而，Rust并发模型的方式使得将自己的并发范例设计为库并让其他人的代码与您合作变得相对容易。 只需要适当的生命周期，并在适当的时候`Send`和`Sync`。
