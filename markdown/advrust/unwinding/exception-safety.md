# 异常安全性

> 源：[exception-safety.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/exception-safety.md) &nbsp; Commit: c4ef161ed0cf6438966d4a44ee53948b540789e8

虽然前面说过我们应该慎用展开，但是还是有许多的地方会Panic。如果你对`None`调用`unwrap`、使用超出范围的索引值、或者用0做除数，你的程序就要panic。在debug模式下，所有的计算操作在溢出的时候也都会panic。除非你十分小心并且严格控制着每一条代码的行为，否则所有的东西都有展开的可能，你需要时刻准备迎接它。

在更广大的程序设计世界里，应对展开这件事通常被称之为“异常安全“。在Rust中，我们需要考虑两个层次的异常安全性：

- 在非安全代码中，异常安全的下限是要保证不能违背内存安全性。我们称之为最小异常安全性。
- 在安全代码中，异常安全性要保证程序时刻在做正确的事情。我们称之为最大异常安全性。

在许多情况下，非安全代码在处理展开的时候需要考虑到那些写得很糟糕的安全代码。一些只是暂时导致不稳定状态的程序需要小心，一旦触发了Panic会导致这种状态无法使用。这表示在不稳定状态依然存在的情况下，我们需要保证值运行不触发Panic的代码；或者在触发Panic的时候即使处理，清除这种状态。这也表明Panic看到的状态并不一定非得是连续的状态，我们只需要保证它是安全地状态就可以。

大多数非安全代码都比较容易实现异常安全。因为它控制着程序运行的每个细节，而且大部分代码不会Panic。但是非安全代码也经常要做诸如在未初始化数据的数组上反复运行外部代码这样的操作。这种代码就需要小心考虑异常安全性了。

## Vec::push_all

`Vec::push_all`使用一个`slice`扩充`Vec`，由于它没有具体化类型，所以能获得较高的效率。下面是一个简单的实现：

``` Rust
impl<T: Clone> Vec<T> {
    fn push_all(&mut self, to_push: &[T]) {
        self.reserve(to_push.len());
        unsafe {
            // 因为我们调用了reserve，所以不会出现溢出
            self.set_len(self.len() + to_push.len());

            for (i, x) in to_push.iter().enumerate() {
                self.ptr().offset(i as isize).write(x.clone());
            }
        }
    }
}
```

我们不去使用`push`，因为它会对Vec的容量和`len`做额外的检查，而有些情况下我们能够明确知道容量是充足的。这段代码的逻辑是完全正确的，但是却有一个问题：它不是异常安全的！`set_len`、`offset`和`write`都没问题，但是`clone`是一颗引发Panic的炸弹。

`Clone`的实现是我们无法控制的，它很可能会panic。如果它真的panic了，这个方法会提前退出，但我们之前给Vec设置的更大的长度会一致保持下去。当Vec被访问或者销毁的时候，它会读取未初始化内存！

解决方法很简单。如果我们要保证我们clone的值都被销毁了，我们可以在每一次循环里设置`len`。如果我们只是想保证不会出现读取未初始化内存的情况，我们可以在循环之后设置`len`。

## BinaryHeap::sift_up

对二叉堆做冒泡比扩充一个Vec要更复杂一点。伪代码是这样的：

```
bubble_up(heap, index):
    while index != 0 && heap[index] < heap[parent(index)]:
        heap.swap(index, parent(index))
        index = parent(index)
```

将它翻译成Rust很容易，但是性能不会让人满意：`self`元素要一遍一遍做无意义的交换。我们更喜欢下面的版本：

```
bubble_up(heap, index):
    let elem = heap[index]
    while index != 0 && elem < heap[parent(index)]:
        heap[index] = heap[parent(index)]
        index = parent(index)
    heap[index] = elem
```

这段代码保证各个元素被尽量少的复制(通常每个元素需要被复制两次)。但是这样它会引发异常安全问题！任何时刻都存在着一个值的两份拷贝。如果这个方法中出现panic，有一些东西可能会被二次释放。不幸的是，我们同样不能完全掌控这段代码，因为比较操作是用户定义的。

这个解决方案比Vec的要困难。一个选项是把用户定义代码和非安全代码拆分成两个阶段：

```
bubble_up(heap, index):
    let end_index = index;
    while end_index != 0 && heap[end_index] < heap[parent(end_index)]:
        end_index = parent(end_index)

    let elem = heap[index]
    while index != end_index:
        heap[index] = heap[parent(index)]
        index = parent(index)
    heap[index] = elem
```

如果用户定义的代码爆炸了，也不会伤及无辜，因为我们还没有实际改变堆的状态。等我们开始在堆上搞事情的时候，我们只会使用我们信任的数据和函数，不用担心panic。

你可能对这个设计感到很不爽。这个属于作弊！而且我们必须对堆完整遍历两次！好吧，让我们直面困难，把不信任代码和不安全代码混合在一起。

如果Rust像Java一样有`try`和`finally`，我们可以这么做：

```
bubble_up(heap, index):
    let elem = heap[index]
    try:
        while index != 0 && elem < heap[parent(index)]:
            heap[index] = heap[parent(index)]
            index = parent(index)
    finally:
        heap[index] = elem
```

基本思想很简单：如果比较操作panic了，我们就把取出的元素塞回到逻辑上未初始化的位置然后退出。访问这个堆的人可能会发现堆的状态是不连续的，但是至少这个方案不会引发二次释放！如果算法正常结束的话，这个设计就和我们最开始不做任何处理的方案一模一样了。

可惜，Rust并没有这些东西，所以我们只能自己早轮子了！我们把算法的状态储存在一个独立的结构体中，结构体的析构函数起到了”finally“的功能。不管有没有panic，析构函数都会被调用并且清除我们留下状态。

``` Rust
struct Hole<'a, T: 'a> {
    data: &'a mut [T],
    // elt从始至终都会是Some
    elt: Option<T>,
    pos: usize,
}

impl<'a, T> Hole<'a, T> {
    fn new(data: &'a mut [T], pos: usize) -> Self {
        unsafe {
            let elt = ptr::read(&data[pos]);
            Hole {
                data: data,
                elt: Some(elt),
                pos: pos,
            }
        }
    }

    fn pos(&self) -> usize { self.pos }

    fn removed(&self) -> &T { self.elt.as_ref().unwrap() }

    unsafe fn get(&self, index: usize) -> &T { &self.data[index] }

    unsafe fn move_to(&mut self, index: usize) {
        let index_ptr: *const _ = &self.data[index];
        let hole_ptr = &mut self.data[self.pos];
        ptr::copy_nonoverlapping(index_ptr, hole_ptr, 1);
        self.pos = index;
    }
}

impl<'a, T> Drop for Hole<'a, T> {
    fn drop(&mut self) {
        // 再次填充hole
        unsafe {
            let pos = self.pos;
            ptr::write(&mut self.data[pos], self.elt.take().unwrap());
        }
    }
}

impl<T: Ord> BinaryHeap<T> {
    fn sift_up(&mut self, pos: usize) {
        unsafe {
            // 取出pos处的值，然后创建一个hole
            let mut hole = Hole::new(&mut self.data, pos);

            while hole.pos() != 0 {
                let parent = parent(hole.pos());
                if hole.removed() <= hole.get(parent) { break }
                hole.move_to(parent);
            }
            // 无论有没有panic，hold在此处都会无条件地被填充
        }
    }
}
```
