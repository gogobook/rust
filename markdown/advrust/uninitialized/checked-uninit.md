# 检查未初始化的内存

> 原文跟踪[checked-uninit.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/checked-uninit.md) &emsp; Commit: d870b6788ba078ba398f020305ef9210f7cbd740

与C一样，Rust中的所有堆栈变量都是未初始化的，直到明确赋值给它们为止。 与C不同，Rust会在您执行以下操作之前静态地阻止您读它们：

```rust
fn main() {
    let x: i32;
    println!("{}", x);
}
```

```rust
  |
3 |     println!("{}", x);
  |                    ^ use of possibly uninitialized `x`
```

这基于一个基本的分支分析：每个分支必须在第一次使用之前为x赋值。 有趣的是，如果每个分支只分配一次，则Rust不要求变量是可变的来执行延迟初始化。 然而，分析没有利用常数分析或类似的东西。 所以这个编译：

```rust
fn main() {
    let x: i32;

    if true {
        x = 1;
    } else {
        x = 2;
    }

    println!("{}", x);
}
```

但这不行：

```rust
fn main() {
    let x: i32;
    if true {
        x = 1;
    }
    println!("{}", x);
}
```

```rust
  |
6 |     println!("{}", x);
  |                    ^ use of possibly uninitialized `x`
```

这样可以:

```rust
fn main() {
    let x: i32;
    if true {
        x = 1;
        println!("{}", x);
    }
    // Don't care that there are branches where it's not initialized
    // since we don't use the value in those branches
}
```

当然，虽然分析不考虑实际值，但它确实对依赖关系和控制流有了相对复杂的理解。 例如，这有效：

```rust
let x: i32;

loop {
    // Rust doesn't understand that this branch will be taken unconditionally,
    // because it relies on actual values.
    if true {
        // But it does understand that it will only be taken once because
        // we unconditionally break out of it. Therefore `x` doesn't
        // need to be marked as mutable.
        x = 0;
        break;
    }
}
// It also knows that it's impossible to get here without reaching the break.
// And therefore that `x` must be initialized here!
println!("{}", x);
```

如果某个值移出变量作用域，那么如果值的类型不是Copy，则该变量在逻辑上将变为未初始化。 那是：

```rust
fn main() {
    let x = 0;
    let y = Box::new(0);
    let z1 = x; // x is still valid because i32 is Copy
    let z2 = y; // y is now logically uninitialized because Box isn't Copy
}
```

但是，在此示例中重新分配`y`将需要将`y`标记为可变，因为`Safe Rust`程序可以观察到`y`的值已更改：

```rust
fn main() {
    let mut y = Box::new(0);
    let z = y; // y is now logically uninitialized because Box isn't Copy
    y = Box::new(1); // reinitialize y
}
```

否则就像`y`是一个全新的变量。
