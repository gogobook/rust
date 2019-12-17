# 使用未初始化的内存

> 原文跟踪[uninitialized.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/uninitialized.md) &emsp; Commit: 0e6c680ebd72f1860e46b2bd40e2a387ad8084ad

Rust程序中所有运行时分配的内存都以未初始化的方式开始。 在这种状态下，存储器的值是不确定的一堆比特，这些比特可能或甚至不能反映应该驻留在该存储器位置的类型的有效状态。 尝试将此内存解释为任何类型的值将导致未定义的行为。 不要这样做。

Rust提供了以已检查（安全）和未检查（不安全）方式处理未初始化内存的机制。
