# Drain

> 源-[vec-drain.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/vec-drain.md) &nbsp; Commit: e45316fbe872ed879c0d0be4cd90492b6c3afa2d

我们接着看看Drain。Drain和IntoIter基本相同，只不过它并不获取Vec的值，而是借用Vec并且不改变它的分配空间。现在我们只是先最“基本”的全范围(full-range)的版本。

``` Rust
use std::marker::PhantomData;

struct Drain<'a, T: 'a> {
    // 这里需要限制生命周期。我们使用&'a mut Vec<T>，因为这就是语义上我们包含的东西。
    // 我们只调用pop()和remove(0)
    vec: PhantomData<&'a mut Vec<T>>,
    start: *const T,
    end: *const T,
}

impl<'a, T> Iterator for Drain<'a, T> {
    type Item = T;
    fn next(&mut self) -> Option<T> {
        if self.start == self.end {
            None
```

——等一下，这个看着有点眼熟。我们需要做进一步的压缩。IntoIter和Drain有着完全一样的结构，我们把它提取出来。

``` Rust
struct RawValIter<T> {
    start: *const T,
    end: *const T,
}

impl<T> RawValIter<T> {
    // 构建它是非安全的，因为它没有关联的生命周期。
    unsafe fn new(slice: &[T]) -> Self {
        RawValIter {
            start: slice.as_ptr(),
            end: if slice.len() == 0 {
                // 如果len == 0，说明没有真的分配内存。这时需要避免offset，
                // 因为那会给LLVM的GEP提供错误的信息
                slice.as_ptr()
            } else {
                slice.as_ptr().offset(slice.len() as isize)
            }
        }
    }
}

// Iterator和DoubleEndedIterator的实现与IntoIter完全一样。
```

IntoIter变成了这样：

``` Rust
pub struct IntoIter<T> {
    _buf: RawVec<T>, // 我们并不关心这个，只是需要它们保持分配空间不被销毁
    iter: RawValIter<T>,
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<T> { self.iter.next() }
    fn size_hint(&self) -> (usize, Option<usize>) { self.iter.size_hint() }
}

impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> { self.iter.next_back() }
}

impl<T> Drop for IntoIter<T> {
    fn drop(&mut self) {
        for _ in &mut self.iter {}
    }
}

impl<T> Vec<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        unsafe {
            let iter = RawValIter::new(&self);

            let buf = ptr::read(&self.buf);
            mem::forget(self);

            IntoIter {
                iter: iter,
                _buf: buf,
            }
        }
    }
}
```

注意，我在设计中留下了一些小后门，以便更简单地将Drain升级为可访问任意子范围的版本。特别是，我们可以在drop中让RawValIter遍历它自己。但是这种设计不适用于更复杂的Drain。我们还使用一个slice简化Drain的初始化。

好了，现在Drain变得很简单：

``` Rust
use std::marker::PhantomData;

pub struct Drain<'a, T: 'a> {
    vec: PhantomData<&'a mut Vec<T>>,
    iter: RawValIter<T>,
}

impl<'a, T> Iterator for Drain<'a, T> {
    type Item = T;
    fn next(&mut self) -> Option<T> { self.iter.next() }
    fn size_hint(&self) -> (usize, Option<usize>) { self.iter.size_hint() }
}

impl<'a, T> DoubleEndedIterator for Drain<'a, T> {
    fn next_back(&mut self) -> Option<T> { self.iter.next_back() }
}

impl<'a, T> Drop for Drain<'a, T> {
    fn drop(&mut self) {
        for _ in &mut self.iter {}
    }
}

impl<T> Vec<T> {
    pub fn drain(&mut self) -> Drain<T> {
        unsafe {
            let iter = RawValIter::new(&self);

            // 这一步是为了mem::forget的安全。如果Drain被forget，我们会泄露整个Vec的内容
            // 同时，既然我们无论如何都会做这一步，为什么不现在做呢？
            self.len = 0;

            Drain {
                iter: iter,
                vec: PhantomData,
            }
        }
    }
}
```

关于更多的`mem::forget`的问题，请见[关于泄露的章节](https://rustlang-cn.org/office/rust/advrust/obrm/leaking.html)。
