# Push与Pop

> 源-[vec-push-pop.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/vec-push-pop.md) &nbsp; Commit: 0e6c680ebd72f1860e46b2bd40e2a387ad8084ad

很好。我们可以初始化，我们也可以分配内存。现在我们开始实现一些真正的功能！我们就从`push`开始吧。它要做的事情就是检查空间是否已满，满了就扩容，然后写数据到下一个索引位置，最后增加长度。

写数据时，我们一定要小心，不要计算我们要写入的内存位置的值。最坏的情况，那块内存是一块未初始化的内存。最好的情况是那里存着我们已经pop出去的值。不管哪种情况，我们都不能直接索引这块内存然后解引用它，因为这样其实是把内存中的值当做了一个合法的T的实例。更糟糕的是，`foo[idx] = x`会调用`foo[idx]`处旧有值的`drop`方法！

正确的方法是使用`ptr::write`，它直接用值的二进制覆盖目标地址，不会计算任何的值。

对于`push`，如果原有的长度（调用push之前的长度）为0，那么我们就要写到第0个索引位置。所以我们应该用原有的长度做offset。

``` Rust
pub fn push(&mut self, elem: T) {
    if self.len == self.cap { self.grow(); }

    unsafe {
        ptr::write(self.ptr.offset(self.len as isize), elem);
    }

    // 这一句不会失败，而会首先OOM
    self.len += 1;
}
```

小菜一碟！那么`pop`是什么样的呢？尽管现在我们要访问的索引位置已经初始化了，Rust不允许我们用解引用的方式将值移出，因为那样的话整个内存都会回到未初始化状态！这时我们需要用`ptr:read`，它从目标位置拷贝出二进制值，然后解析成类型T的值。这时原有位置处的内存逻辑上是未初始化的，可实际上那里还是存在这一个正常的T的实例。

对于`pop`，如果原有长度是1，我们要读的是第0个索引位置。所以我们应该是按新的长度做offset。

``` Rust
pub fn pop(&mut self) -> Option<T> {
    if self.len == 0 {
        None
    } else {
        self.len -= 1;
        unsafe {
            Some(ptr::read(self.ptr.offset(self.len as isize)))
        }
    }
}
```
