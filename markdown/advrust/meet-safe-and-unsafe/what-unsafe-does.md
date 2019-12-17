# Unsafe Rust 能做什么

> 原文跟踪[what-unsafe-does.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/what-unsafe-does.md) &emsp; Commit: dd8054bef8c654aaa27ca7979e21aa984fd2abc6

Usafe Rust 唯一的区别在于，它允许你多做如下事情:

* 解引用原始指针
* 调用 `unsafe` 函数（包括 C 函数, 编译器内部预置的和原始内存分配器）
* 实现 `unsafe` trait
* 修改静态变量
* 访问 `union` 的字段

这就是全部了, 把这些操作归为 Unsafe 的原因是, 滥用这些会导致可怕的未定义的行为. 触发未定义的行为就相当于给予编译器在你程序里做一些武断且不好事情的权力. 你应该*完全避免*触发任何未定义的行为.

不像 C, Rust 中未定义的行为是限定在一定范围内的. 所有核心语法都会刻意的去预防如下这些事情：

* 解引用 null、悬空或未对齐的指针
* 读取[未初始化的内存][uninitialized memory]
* 打破[指针别名规则][pointer aliasing rules]
* 生成无效的原始值：
    * 悬空或 null 引用
    * 空 `fn` 函数指针
    * 非 0 或 1 的布尔值
    * 未定义的 `enum` 判别式
    * [0x0, 0xD7FF] 和 [0xE000, 0x10FFFF] 范围外的 `char` 值
    * 非 utf-8 的 `str`
* 栈解退（Unwinding）到另一种语言
* 引起[数据争用][race]

就这些而已, 这就是Rust中未定义行为的所有原因. 当然, unsafe 函数和 trait 可以自由随意的去声明一些其他约束, 用来约束程序程序坚持去避免未定义的行为. 举个例子, 内存分配器的 API 声明取消分配那些没分配过的内存, 就是未定义的行为.

但是, 违反这些约束通常只会导致上述问题的其中一个. 一些额外的约束也可以源自编译器内部, 并可以对如何优化代码做出特殊的假设. 举个例子, Vec 和 Box 使用编译器内部预置的函数, 要求它们的指针始终为非 null.

对另一些可以的操作, Rust 也可以相当宽容. Rust 认为以下都是"安全"的:

* 死锁
* 有[竞争条件][race]
* 内存泄漏
* 调用析构函数失败
* 整型溢出
* 中止程序
* 删除生产环境数据库

但是, 真这么做了可能就不太好了, 尤其最后一条. Rust 提供了很多工具去阻止或减少这些事情的发生. 但是如果要完全避免这些问题是不切实际的.


[pointer aliasing rules]: references.html
[uninitialized memory]: uninitialized.html
[race]: races.html
