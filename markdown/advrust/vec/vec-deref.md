# Deref

> 源-[vec-deref.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/vec-deref.md) &nbsp; Commit: e9335c82a2a73ad68f0516ff241c973dfa31ee16

不错！我们实现了一个成熟的小的栈。我们可以push、可以pop、也可以自动清理。但是还是有一堆的功能是我们需要的。特别是，我们已经有了一个很好的数组，但是还没有slice相关的功能。这非常容易解决：我们可以实现`Deref<Target=[T]>`。这样我们的Vec就神奇地变成了slice。

我们只需要使用`slice::from_raw_parts`。它能够为我们正确处理空slice。等到后面我们完成了零尺寸类型的支持，它们依然可以完美配合。

``` Rust
use std::ops::Deref;

impl<T> Deref for Vec<T> {
    type Target = [T];
    fn deref(&self) -> &[T] {
        unsafe {
            ::std::slice::from_raw_parts(self.ptr.as_ptr(), self.len)
        }
    }
}
```

我们把DefMut也实现了吧：

``` Rust
use std::ops::DerefMut;

impl<T> DerefMut for Vec<T> {
    fn deref_mut(&mut self) -> &mut [T] {
        unsafe {
            ::std::slice::from_raw_parts_mut(self.ptr.as_ptr(), self.len)
        }
    }
}
```

现在我们有了`len`、`first`、`last`、索引、分片、排序、`iter`、`iter_mut`，以及其他所有的slice提供的功能。完美！
