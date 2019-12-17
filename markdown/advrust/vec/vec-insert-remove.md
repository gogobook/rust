# 插入和删除

> 源-[vec-insert-remove.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/vec-insert-remove.md) &nbsp; Commit: 0e6c680ebd72f1860e46b2bd40e2a387ad8084ad

Slice并没有提供插入和删除功能，接下来我们就实现它们。

插入需要把目标位置后的所有元素都向右移动1。这里我们需要用到`ptr::copy`，它就是C中的`memmove`的Rust版。它把一块内存从一个地方拷贝到另一个地方，而且可以正确处理源和目标内存区域有重叠的情况（也正是我们这里遇到的情况）。

如果我们在`i`的位置插入，我们需要把[i .. len]移动到[i+1 .. len+1]，len指的是插入前的值。

``` Rust
pub fn insert(&mut self, index: usize, elem: T) {
    // 注意：<=是因为我们可以把值插到所有元素的后面
    // 这种情况等同于push
    assert!(index <= self.len, "index out of bounds");
    if self.cap == self.len { self.grow(); }

    unsafe {
        if index < self.len {
            // ptr::copy(src, dest, len): "从src拷贝len个元素到dest"
            ptr::copy(self.ptr.offset(index as isize),
                      self.ptr.offset(index as isize + 1),
                      self.len - index);
        }
        ptr::write(self.ptr.offset(index as isize), elem);
        self.len += 1;
    }
}
```

删除则是完全相反的行为。我们要把元素[i+1 .. len + 1]移动到[i .. len]，len是删除后的值。

``` Rust
pub fn remove(&mut self, index: usize) -> T {
    // 注意：<是因为我们不能删除所有元素之后的位置
    assert!(index < self.len, "index out of bounds");
    unsafe {
        self.len -= 1;
        let result = ptr::read(self.ptr.offset(index as isize));
        ptr::copy(self.ptr.offset(index as isize + 1),
                  self.ptr.offset(index as isize),
                  self.len - index);
        result
    }
}
```
