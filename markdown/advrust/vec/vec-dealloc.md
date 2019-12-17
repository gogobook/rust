# 重新分配

> 源-[vec-dealloc.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/vec-dealloc.md) &nbsp; Commit: e9335c82a2a73ad68f0516ff241c973dfa31ee16

接下来我们应该实现Drop，否则就要造成大量的资源泄露了。最简单的方法是循环调用`pop`直到产生None为止，然后再回收我们的缓存。注意，当`T: !Drop`的时候，调用`pop`不是必须的。理论上我们可以问一问Rust`T`是不是`need_drop`然后再省略一些`pop`调用。可实际上LLVM很擅长移除像这样的无副作用的代码，所以我们不需要再做多余的事，除非你发现LLVM不能成功移除（在这里它能）。

在`self.cap == 0`的时候，我们一定不要调用`heap::deallocate`，因为这时我们还没有实际分配过任何内存。

``` Rust
impl<T> Drop for Vec<T> {
    fn drop(&mut self) {
        if self.cap != 0 {
            while let Some(_) = self.pop() { }

            let align = mem::align_of::<T>();
            let elem_size = mem::size_of::<T>();
            let num_bytes = elem_size * self.cap;
            unsafe {
                heap::deallocate(self.ptr.as_ptr() as *mut _, num_bytes, align);
            }
        }
    }
}
```
