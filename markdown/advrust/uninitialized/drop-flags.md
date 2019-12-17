# 析构(Drop)标志

> 原文跟踪[drop-flags.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/drop-flags.md) &emsp; Commit: a73391dd35c32061bec678257d4c3ddac268c51b

上一节中的示例为Rust引入了一个有趣的问题。 我们已经看到，可以完全安全地有条件地初始化，取消初始化和重新初始化内存位置。 对于`Copy`类型，这并不是特别值得注意，因为它们只是一堆随机的`bits`。 但是，带有析构函数的类型是一个不同的故事：Rust需要知道是否在分配变量时调用析构函数，或者变量超出范围。 如何通过条件初始化来做到这一点？

请注意，这不是所有分配都需要担心的问题。 特别是，通过解引用进行分配将无条件地析构(drop)，在`let`中分配无条件不会析构(drop)：

```rust
let mut x = Box::new(0); // let makes a fresh variable, so never need to drop
let y = &mut x;
*y = Box::new(1); // Deref assumes the referent is initialized, so always drops
```

当覆盖先前初始化的变量或其子字段之一时，这是一个问题。

事实证明，Rust实际上跟踪是否应该在运行时删除类型。 当变量初始化并且未初始化时，将切换该变量的drop标志。 当可能需要删除变量时，将评估此标志以确定是否应删除该变量。

当然，通常情况下，可以在程序的每个点静态地知道值的初始化状态。 如果是这种情况，那么编译器理论上可以生成更高效的代码！ 例如，`straight-
line`代码具有这样的静态析构语义(static drop semantics)：

```rust
let mut x = Box::new(0); // x was uninit; just overwrite.
let mut y = x;           // y was uninit; just overwrite and make x uninit.
x = Box::new(0);         // x was uninit; just overwrite.
y = x;                   // y was init; Drop y, overwrite it, and make x uninit!
                         // y goes out of scope; y was init; Drop y!
                         // x goes out of scope; x was uninit; do nothing.
```

类似地，所有分支在初始化方面具有相同行为的分支代码具有静态析构语义(static drop semantics)：

```rust
# let condition = true;
let mut x = Box::new(0);    // x was uninit; just overwrite.
if condition {
    drop(x)                 // x gets moved out; make x uninit.
} else {
    println!("{}", x);
    drop(x)                 // x gets moved out; make x uninit.
}
x = Box::new(0);            // x was uninit; just overwrite.
                            // x goes out of scope; x was init; Drop x!
```

但是这样的代码需要运行时信息才能正确析构：

```rust
# let condition = true;
let x;
if condition {
    x = Box::new(0);        // x was uninit; just overwrite.
    println!("{}", x);
}
                            // x goes out of scope; x might be uninit;
                            // check the flag!
```

当然，在这种情况下，检索静态析构语义是微不足道的：

```rust
# let condition = true;
if condition {
    let x = Box::new(0);
    println!("{}", x);
}
```

`drop`标记在堆栈上被跟踪，不再存储在实现`drop`的类型中。
