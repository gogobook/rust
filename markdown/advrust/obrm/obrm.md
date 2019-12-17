# 基于所有权的资源管理

> 源：[obrm.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/obrm.md) &nbsp; Commit: 0e6c680ebd72f1860e46b2bd40e2a387ad8084ad

OBRM（AKA RAII：资源获取即初始化）是您在Rust中通常与之交互的东西。 特别是如果您使用标准库。

粗略地说，模式如下：要获取资源，您需要创建一个管理它的对象。 要释放资源，只需销毁对象，它就会为您清理资源。 这种模式管理的最常见的“资源”就是内存。 `Box`，`Rc`，以及`std :: collections`中的所有内容都可以方便地正确管理内存。 这在Rust中尤为重要，因为我们没有普遍的依赖GC来内存管理。 真正重点是：Rust是关于控制的。 但是，我们并不仅限于内存。 几乎所有其他系统资源（如线程，文件或套接字）都通过这种API公开。
