# RawVec

> 源-[vec-raw.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/vec-raw.md) &nbsp; Commit: e9335c82a2a73ad68f0516ff241c973dfa31ee16

我们遇到了一个很有意思的情况：我们把初始化缓存和释放内存的逻辑在Vec和IntoIter里面一模一样地写了两次。现在我们已经实现了功能，而且发现了逻辑的重复，是时候对代码做一些压缩了。

我们要抽象出`(ptr, cap)`，并赋予它们分配、扩容和释放的逻辑：

``` Rust
struct RawVec<T> {
    ptr: Unique<T>,
    cap: usize,
}

impl<T> RawVec<T> {
    fn new() -> Self {
        assert!(mem::size_of::<T>() != 0, "TODO:实现零尺寸类型的支持");
        RawVec { ptr: Unique::empty(), cap: 0 }
    }

    // 与Vec一样
    fn grow(&mut self) {
        unsafe {
            let align = mem::align_of::<T>();
            let elem_size = mem::size_of::<T>();

            let (new_cap, ptr) = if self.cap == 0 {
                let ptr = heap::allocate(elem_size, align);
                (1, ptr)
            } else {
                let new_cap = 2 * self.cap;
                let ptr = heap::reallocate(self.ptr.as_ptr() as *mut _,
                                            self.cap * elem_size,
                                            new_cap * elem_size,
                                            align);
                (new_cap, ptr)
            };

            // 如果分配或再分配失败，我们会得到null
            if ptr.is_null() { oom() }

            self.ptr = Unique::new(ptr as *mut _);
            self.cap = new_cap;
        }
    }
}


impl<T> Drop for RawVec<T> {
    fn drop(&mut self) {
        if self.cap != 0 {
            let align = mem::align_of::<T>();
            let elem_size = mem::size_of::<T>();
            let num_bytes = elem_size * self.cap;
            unsafe {
                heap::deallocate(self.ptr.as_mut() as *mut _, num_bytes, align);
            }
        }
    }
}
```

然后像下面这样改写Vec：

``` Rust
pub struct Vec<T> {
    buf: RawVec<T>,
    len: usize,
}

impl<T> Vec<T> {
    fn ptr(&self) -> *mut T { self.buf.ptr.as_ptr() }

    fn cap(&self) -> usize { self.buf.cap }

    pub fn new() -> Self {
        Vec { buf: RawVec::new(), len: 0 }
    }

    // push/pop/insert/remove基本没变，只改变了:
    // self.ptr -> self.ptr()
    // self.cap -> self.cap()
    // self.grow -> self.buf.grow()
}

impl<T> Drop for Vec<T> {
    fn drop(&mut self) {
        while let Some(_) = self.pop() {}
        // 释放空间由RawVec负责
    }
}
```

最后我们可以简化IntoIter：

``` Rust
struct IntoIter<T> {
    _buf: RawVec<T>, // 我们并不关心这个，只是需要它们保持分配空间不被销毁
    start: *const T,
    end: *const T,
}

// next和next_back保持不变，因为它们并没有用到buf

impl<T> Drop for IntoIter<T> {
    fn drop(&mut self) {
        // 只需要保证所有的元素都被读到了
        // 缓存会在随后自己清理自己
        for _ in &mut *self {}
    }
}

impl<T> Vec<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        unsafe {
            // 需要使用ptr::read非安全地把buf移出，因为它不是Copy，
            // 而且Vec实现了Drop（所以我们不能销毁它）
            let buf = ptr::read(&self.buf);
            let len = self.len;
            mem::forget(self);

            IntoIter {
                start: *buf.ptr,
                end: buf.ptr.offset(len as isize),
                _buf: buf,
            }
        }
    }
}
```

现在看起来好多了。
