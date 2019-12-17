# 实现Arc和Mutex

> 原文跟踪[arc-and-mutex.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/arc-and-mutex.md) &emsp; Commit: 0e6c680ebd72f1860e46b2bd40e2a387ad8084ad

了解理论一切都很好，但理解某事的最好方法就是使用它。 为了更好地理解原子和内部可变性，我们将实现标准库的`Arc`和`Mutex`类型的版本。

