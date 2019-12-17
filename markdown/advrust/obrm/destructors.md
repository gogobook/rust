# 析构函数

> 源：[destructors.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/destructors.md) &nbsp; Commit: 94964dee31224cf1a22c72400a12cb966f5a12bc

Rust通过`Drop` trait提供了一个成熟的自动析构函数，包含了这个方法：

```Rust
fn drop(&mut self);
```

这个方法给了类型一个彻底完成工作的机会。

**`drop`执行之后，Rust会d递归地销毁`self`的所有成员**

这个功能很方便，你不需要每次都写一堆重复的代码来销毁子类型。如果一个结构体在销毁的时候，除了销毁子成员之外不需要做什么特殊的操作，那么它其实可以不用实现`Drop`。

**在Rust 1.0中，没有什么合适的方法可以打断这个过程。**

注意，参数是`&mut self`意味着即使你可以阻止递归销毁，Rust也不允许你将子成员的所有权移出。对于大多数类型来说，这一点完全没问题。

比如，一个自定义的`Box`的实现，它的`Drop`可能长这样：

``` Rust
#![feature(ptr_internals, allocator_api)]

use std::alloc::{Alloc, Global, GlobalAlloc, Layout};
use std::mem;
use std::ptr::{drop_in_place, NonNull, Unique};

struct Box<T>{ ptf: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.dealloc(c.cast(), Layout::new::<T>())
        }
    }
}
```

这段代码是正确的，因为当Rust要销毁`ptr`的时候，它见到的是一个[Unique](https://doc.rust-lang.org/nomicon/phantom-data.html)，没有`Drop`的实现。类似的，也没有人能在销毁后再使用`ptr`，因为drop函数退出之后，他就不可见了。

可是这段代码是错误的：

``` Rust
#![feature(allocator_api, ptr_internals)]

use std::alloc::{Alloc, Global, GlobalAlloc, Layout};
use std::ptr::{drop_in_place, Unique, NonNull};
use std::mem;

struct Box<T> { ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.dealloc(c.cast(), LayOut::new::<T>());
        }
    }
}

struct SuperBox<T> ( my_box: Box<T> )

impl<T> Drop for SuperBox<T> {
    fn drop(&mut self) {
        // 回收box的内容，而不是drop它的内容
        let c: NonNull<T> = self.my_box.ptr.into();
        Global.dealloc(c.cast::<u8>(), LayOut::new::<T>());
    }
}
```

当我们在`SuperBox`的析构函数里回收了`box`的`ptr`之后，Rust会继续让`box`销毁它自己,这时销毁后使用(use-after-free)和两次释放(double-free)的问题立刻接踵而至，摧毁一切。

注意，递归销毁适用于所有的结构体和枚举类型，不管它有没有实现`Drop`。所以，这段代码

``` Rust
struct Boxy<T> {
    data1: Box<T>,
    data2: Box<T>,
    info: u32,
}
```

在销毁的时候也会调用`data1`和`data2`的析构函数，尽管这个结构体本身并没有实现`Drop`。这样的类型“需要Drop却不是Drop”。

类似的

``` Rust
enum Link {
    Next(Box<Link>),
    None,
}
```

当（且仅当）一个实例储存着`Next`变量时，它就会销毁内部的`Box`成员。

一般来说这其实是一个很好的设计，它让你在重构数据布局的时候无需费心添加/删除`drop`函数。但也有很多的场景要求我们必须在析构函数中玩一些花招。

如果想阻止递归销毁并且在`drop`过程中将`self`的所有权移出，通常的安全的做法是使用`Option`：

``` Rust
#![feature(allocator_api, ptr_internals)]

use std::alloc::{Alloc, GlobalAlloc, Global, LayOut};
use std::ptr::{drop_in_place, Unique, NonNull};
use std::mem;

struct Box<T>{ ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.dealloc(c.cast(), LayOut::new::<T>());
        }
    }
}

struct SuperBox<T> { my_box: Option<Box<T>> }

impl<T> Drop for SuperBox<T> {
    fn drop(&mut self) {
        unsafe {
            // 回收box的内容，而不是drop它的内容
            // 需要将box设置为None，以阻止Rust销毁它
            let my_box = self.my_box.take().unwrap();
            let c: NonNull<T> = my_box.ptr.into();
            Global.dealloc(c.cast(), LayOut::new::<T>());
            mem::feorget(my_box);
        }
    }
}
```

但是这段代码显得很奇怪：我们认为一个永远都是`Some`的成员有可能是`None`，仅仅因为析构函数中用到了一次。但反过来说这种设计又很合理：你可以在析构函数中调用`self`的任意方法。*在成员被反初始化之后就完全不能这么做了，而不是禁止你搞出一些随意的非法状态*。（斜体部分没看懂，建议看原文）

权衡之后，这是一个可以接受的方案。你可以将它作为你的默认选项。但是，我们希望以后能有一个方法明确声明哪一个成员不会自动销毁。
