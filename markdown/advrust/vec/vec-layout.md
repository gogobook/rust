# 布局

> 源：[vec-layout.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/vec-layout.md) &nbsp; Commit: 94964dee31224cf1a22c72400a12cb966f5a12bc

我们先来看看结构体的布局。Vec由三部分组成：一个指向分配空间的指针、空间的大小、以及已经初始化的元素的数量。

简单来说，我们的设计只要这样：

``` Rust
pub struct Vec<T> {
    ptr: *mut T,
    cap: usize,
    len: usize,
}
```

这段代码可以通过编译。可不幸的是，它是不正确的。首先，编译器产生的变性过于严格。所以`&Vev<&'static str>`不能当做`&Vev<&'a str>`使用。更主要的是，它会给drop检查器传递错误的所有权信息,因为编译器会保守地假设我们不拥有任何的值。关于变性和drop检查的细节，请见[所有权和生命周期](https://rustlang-cn.org/office/rust/advrust/ownership/ownership.html)。

.
正如我们在所有权一章见到的，当裸指针指向一块我们拥有所有权的位置，我们应该使用`Unique<T>`代替`*mut T`。尽管Unique是不稳定的，我们尽可能不去使用它。

复习一下，Unique封装了一个裸指针，并且声明它自己：

- 对`T`可变
- 拥有类型T的值（用于drop检查）
- 如果`T`是Send/Sync，那就也是Send/Sync
- 指针永远不为null（所以`Option<Vec<T>>可以做空指针优化）

除了最后一点，其余的我们都可以用稳定的Rust实现：

``` Rust
use std::marker::PhantomData;
use std::ops::Deref;
use std::mem;

struct Unique<T> {
    ptr: *const T,            // 使用*const保证变性  
    _marker: PhantomData<T>,  // 用于drop检查
}

// 设置Send和Sync是安全地，因为我们是Unique中的数据的所有者
// Unique<t>好像就是T一样
unsafe impl<T: Send> Send for Unique<T> {}
unsafe impl<T: Sync> Sync for Unique<T> {}

impl<T> Unique<T> {
    pub fn new(ptr: *mut T) -> Self {
        Unique { ptr: ptr, _marker: PhantomData }
    }

    pub fn as_ptr(&self) -> *mut T {
        self.ptr as *mut T
    }
}
```

可是，声明数据不为0的方法是不稳定的，而且短期内都不太可能会稳定下来。so我们还是接受现实，使用比标准库的Unique：

``` Rust
#![feature(ptr_internals)]

use std::ptr::{Unique, self};

pub struct Vec<T> {
    ptr: Unique<T>,
    cap: usize,
    len: usize,
}
```

如果你不太在意空指针优化，那么你可以使用稳定代码。但是我们之后的代码会依赖于这个优化去设计。还要注意，调用`Unique::new`是非安全的，因为给它传递null属于未定义行为。我们的稳定Unique就不需要让`new`是非安全的，因为它没有对于它的内容做其他的保证。

