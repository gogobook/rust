# 处理零尺寸类型

> 源:[vec-zsts.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/vec-zsts.md) &nbsp; Commit: e9335c82a2a73ad68f0516ff241c973dfa31ee16

是时候和零尺寸类型开战了。安全Rust并不需要关心这个，但是Vec大量的依赖裸指针和内存分配，这些都需要零尺寸类型。我们要小心两件事情：

- 当给分配器API传递分配尺寸为0时，会导致未定义行为
- 对零尺寸类型的裸指针做offset是一个no-op，这会破坏我们的C-style指针迭代器。

幸好我们把指针迭代器和内存分配逻辑抽象出来放在RawValIter和RawVec中了。真是太方便了。

## 为零尺寸类型分配空间

如果分配器API不支持分配大小为0的空间，那么我们究竟储存了些什么呢？当然是`Unique::empty()`了！基本上所有关于ZST的操作都是no-op，因为ZST只有一个值，不需要储存或加载任何的状态。这也同样适用于`ptr::read`和`ptr::write`：它们根本不会看那个指针一眼。所以我们并不需要修改指针。

注意，我们之前的分配代码依赖于OOM会先于数值溢出出现的假设，对于零尺寸类型不再有效了。我们必须显式地保证cap的值在ZST的情况下不会溢出。

基于现在的架构，我们需要写3处保护代码，RawVec的三个方法每个都有一处。

``` Rust
impl<T> RawVec<T> {
    fn new() -> Self {
        // !0就是usize::MAX。这段分支代码在编译期就可以计算出结果。
        let cap = if mem::size_of::<T>() == 0 { !0 } else { 0 };

        // Unique::empty()有着“未分配”和“零尺寸分配”的双重含义
        RawVec { ptr: Unique::empty(), cap: cap }
    }

    fn grow(&mut self) {
        unsafe {
            let elem_size = mem::size_of::<T>();

            // 因为当elem_size为0时我们设置了cap为usize::MAX，
            // 这一步成立意味着Vec的容量溢出了
            assert!(elem_size != 0, "capacity overflow");

            let align = mem::align_of::<T>();

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
        let elem_size = mem::size_of::<T>();

        // 不要释放零尺寸空间，因为它根本就没有分配过
        if self.cap != 0 && elem_size != 0 {
            let align = mem::align_of::<T>();

            let num_bytes = elem_size * self.cap;
            unsafe {
                heap::deallocate(self.ptr.as_ptr() as *mut _, num_bytes, align);
            }
        }
    }
}
```

就是这样。我们现在已经支持push和pop零尺寸类型了。但是迭代器（slice未提供的）还不能工作。

## 迭代零尺寸类型

offset 0是一个no-op。这意味着我们的`start`和`end`总是会被初始化为相同的值，我们的迭代器也无法产生任何的东西。当前的解决方案是把指针转换为整数，增加他们的值，然后再转换回来：

``` Rust
impl<T> RawValIter<T> {
    unsafe fn new(slice: &[T]) -> Self {
        RawValIter {
            start: slice.as_ptr(),
            end: if mem::size_of::<T>() == 0 {
                ((slice.as_ptr() as usize) + slice.len()) as *const _
            } else if slice.len() == 0 {
                slice.as_ptr()
            } else {
                slice.as_ptr().offset(slice.len() as isize)
            }
        }
    }
}
```

现在我们有了一个新的bug。我们成功地让迭代器从完全不运行，变成了永远不停地运行。我们需要在迭代器的实现中玩同样的把戏。同时，`size_hint`在ZST的情况下会出现除数为0的问题。因为我们假设这两个指针都指向某个字节，我们在除数为0的情况下直接将除数变为1。

``` Rust
impl<T> Iterator for RawValIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                let result = ptr::read(self.start);
                self.start = if mem::size_of::<T>() == 0 {
                    (self.start as usize + 1) as *const _
                } else {
                    self.start.offset(1)
                };
                Some(result)
            }
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        let elem_size = mem::size_of::<T>();
        let len = (self.end as usize - self.start as usize)
                  / if elem_size == 0 { 1 } else { elem_size };
        (len, Some(len))
    }
}

impl<T> DoubleEndedIterator for RawValIter<T> {
    fn next_back(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                self.end = if mem::size_of::<T>() == 0 {
                    (self.end as usize - 1) as *const _
                } else {
                    self.end.offset(-1)
                };
                Some(ptr::read(self.end))
            }
        }
    }
}
```

很好，迭代器也可以工作了。
