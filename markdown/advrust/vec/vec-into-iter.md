# IntoIter

> 源-[vec-into-iter.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/vec-into-iter.md) &nbsp; Commit: e9335c82a2a73ad68f0516ff241c973dfa31ee16

我们继续编写迭代器。`iter`和`iter_mut`其实已经写过了，感谢神奇的DeRef。但是还有两个有意思的迭代器是Vec提供的而slice没有的：`into_iter`和`drain`。

IntoIter以值而不是引用的形式访问Vec，同时也是以值的形式返回元素。为了实现这一点，IntoIter需要获取Vec的分配空间的所有权。

IntoIter也需要DoubleEnd，即从两个方向读数据。从尾部读数据可以通过调用`pop`实现，但是从头读数据就困难了。我们可以调用`remove(0)`，但是它的开销太大了。我们选择直接使用`ptr::read`从Vec的两端拷贝数据，而完全不去改变缓存。

我们要用一个典型的C访问数组的方式来实现这一点。我们先创建两个指针，一个指向数组的开头，另一个指向结尾后面的那个元素。如果我们需要一端的元素，我们就从那一端指针指向的位置处读出值，然后把指针移动一位。当两个指针相等时，就说明迭代完成了。

注意，`next`和`next_back`中的读和offset的顺序是相反的。对于`next_back`，指针总是指向它下一次要读的元素的后面，而`next`的指针总是指向它下一次要读的元素。为什么要这样呢？考虑一下只剩一个元素还未被读取的情况。

这时的数组像这样：

```
          S  E
[X, X, X, O, X, X, X]
```

如果E直接指向它下一次要读的元素，我们就无法把上面的情况和所有元素都读过了的情况区分开了。

我们还需要保存Vec的分配空间的信息，虽然在迭代过程中我们并不关心它，但我们在IntoIter被drop的时候需要这些信息来释放空间。

所以我们要用下面这个结构体：

``` Rust
struct IntoIter<T> {
    buf: Unique<T>,
    cap: usize,
    start: *const T,
    end: *const T,
}
```

这是初始化的代码：

``` Rust
impl<T> Vec<T> {
    fn into_iter(self) -> IntoIter<T> {
        // 因为Vec是Drop，不能销毁它
        let ptr = self.ptr;
        let cap = self.cap;
        let len = self.len;

        // 确保Vec不会被drop，因为那样会释放内存空间
        mem::forget(self);

        unsafe {
            IntoIter {
                buf: ptr,
                cap: cap,
                start: *ptr,
                end: if cap == 0 {
                    // 没有分配空间，不能计算指针偏移量
                    *ptr
                } else {
                    ptr.offset(len as isize)
                }
            }
        }
    }
}
```

这是前向迭代的代码：

``` Rust
impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                let result = ptr::read(self.start);
                self.start = self.start.offset(1);
                Some(result)
            }
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        let len = (self.end as usize - self.start as usize)
                  / mem::size_of::<T>();
        (len, Some(len))
    }
}
```

这是逆向迭代的代码：

``` Rust
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                self.end = self.end.offset(-1);
                Some(ptr::read(self.end))
            }
        }
    }
}
```

因为IntoIter获得了分配空间的所有权，它需要实现Drop来释放空间。同时Drop也要销毁所有它拥有但是没有读取到的元素。

``` Rust
impl<T> Drop for IntoIter<T> {
    fn drop(&mut self) {
        if self.cap != 0 {
            // drop剩下的元素
            for _ in &mut *self {}

            let align = mem::align_of::<T>();
            let elem_size = mem::size_of::<T>();
            let num_bytes = elem_size * self.cap;
            unsafe {
                heap::deallocate(self.buf.as_ptr() as *mut _, num_bytes, align);
            }
        }
    }
}
```
