# 引用

> 原文跟踪[references.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/references.md) &emsp; Commit: c4822cd9077591c73e3a80ad7386b7e554a90f61

有两种引用：

* 共享引用：`＆`
* 可变引用：`＆mut`

遵守以下规则：

* 引用不能超过它的引用者
* 可变引用不能别名

就这， 这是整个引用模型。

当然，我们应该定义*别名*的含义。

```rust
error[E0425]: cannot find value `aliased` in this scope
 --> <rust.rs>:2:20
  |
2 |     println!("{}", aliased);
  |                    ^^^^^^^ not found in this scope

error: aborting due to previous error
```

不幸的是，Rust实际上没有定义其别名模型。🙀

当我们等待Rust开发人员指定他们语言的语义时，让我们下一节来讨论一般的别名，以及它为何重要。

