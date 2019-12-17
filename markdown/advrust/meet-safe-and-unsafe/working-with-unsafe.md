# 与 Unsafe 玩耍

> 原文跟踪[working-with-unsafe.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/working-with-unsafe.md) &emsp; Commit: 79d7569b693ea5b0225d4b912e34cd039e61d291

`Rust` 通常只给我们相应工具, 在一定范围内用二进制的方式去和 `Unsafe Rust` 对话. 不幸的是, 现实情况要远远复杂的多. 举个例子, 请看如下示例函数:

```rust
fn index(idx: usize, arr: &[u8]) -> Option<u8> {
    if idx < arr.len() {
        unsafe {
            Some(*arr.get_unchecked(idx))
        }
    } else {
        None
    }
}
```

这个函数是安全且正确的. 我们检查了其索引是在范围内的, 如果在范围内, 用未检查的方式去索引这个数组里的元素. 但是就算再这么一个小的函数里, `unsafe` 作用域范围内其实是有问题的. 考虑下把`<` 改成 `<=`

```rust
fn index(idx: usize, arr: &[u8]) -> Option<u8> {
    if idx <= arr.len() {
        unsafe {
            Some(*arr.get_unchecked(idx))
        }
    } else {
        None
    }
}
```

安全是边界清晰的(译者注: 原文 `modular`, 直译模块化), 因为选择不安全并不要求您考虑任意其他类型的不良之处. 例如, 进行未检查边界的切片索引并不意味着你马上就要担心切片可能为空或者包含未初始化的内存. 没有什么根本上的变化. 但是安全又**不是**边界清晰的, 因为程序天然就是有状态的, 并且您的 `unsafe` 操作可能依赖其他任意的状态.

当我们包含实际的持久状态时, 这种非本地的状态会更糟糕.
考录有如下一个简单的 `Vec` 实现:

```rust
use std::ptr;

// 注意: 这个定义比较简单. 更多信息可查阅 Vec 实现章节.
pub struct Vec<T> {
    ptr: *mut T,
    len: usize,
    cap: usize,
}

// 注意这个实现没有正确处理 0长度时的情况.
// 更多信息可查阅 `Vec` 实现章节.
impl<T> Vec<T> {
    pub fn push(&mut self, elem: T) {
        if self.len == self.cap {
            // 作为实例, 无具体实现
            self.reallocate();
        }
        unsafe {
            ptr::write(self.ptr.offset(self.len as isize), elem);
            self.len += 1;
        }
    }
}
```

这段代码足够简单, 可以和进行审查和非正式的验证. 现在考虑添加如下方法:

```rust
fn make_room(&mut self) {
    // 增长容量(`capacity`)
    self.cap += 1;
}
```

这段代码 `100%` 是 `Safe Rust`, 但是他同时完全是错误的. 修改容量大小违反了 `Vec` 的不可变性(`cap` 反映了 `Vec` 里分配的空间). 这部分内容是无法通过 `Vec` 其余部分来来防御保证的. 他**必须**信任 `capacity` 这个字段因为这是没有办法验证的.

因为它依赖于结构体中字段的不变性, 所以这个 `unsafe` 代码不仅仅污染了整个方法: 还污染了整个**模块**. 通常, 限制非安全代码范围边界的唯一有效办法是利用模块的私有边界, 通过声明为本模块私有实现.

这样做简直**完美**. `make_room` 是否存在, 并不是 `Vec`程序健壮与否的问题所在, 因为我们并没有把它声明为 `public`. 只有定义这个方法的模块可以调用它. `make_room` 直接访问了`Vec` 的私有字段, 所以它也只能在同一个 `Vec` 模块中编写.

因此, 我们就有可能编写一个依赖复杂不变性的完全安全的抽象. 这就是 `Safe Rust` 和 `Unsafe Rust` 之间关系的**边界**.

我们已经看了 `Unsafe` 代码必须信任一些`Safe` 代码, 但是不应该信任 **一般的(generic)** `Safe` 代码. 同样的道理模块外部可见性是很重要的: 有了它, 我们就不必去信任全局所有的 `safe` 代码, 从而破坏其是否可信的状态.

安全万岁!