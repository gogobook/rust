# 分解借用

> 源：[borrow-splitting.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/borrow-splitting.md) &nbsp; Commit: d870b6788ba078ba398f020305ef9210f7cbd740

可变引用的Mutex属性在处理复合类型时能力非常有限。借用检查器只能理解一些简单的东西，而且极易失败。他对结构体还算是充分了解，知道结构体的成员可能被分别借用。所以这段代码现在可以正常工作：

``` Rust
struct Foo {
    a: i32,
    b: i32,
    c: i32,
}

let mut x = Foo {a: 0, b: 0, c: 0};
let a = &mut x.a;
let b = &mut x.b;
let c = &x.c;
*b += 1;
let c2 = &x.c;
*a += 10;
println!("{} {} {} {}", a, b, c, c2);
```

但是，借用检查器对于数组和slice的理解却是一团浆糊，所以这段代码无法通过检查：

``` Rust
let mut x = [1, 2, 3];
let a = &mut x[0];
let b = &mut x[1];
println!("{} {}", a, b);
```

```
error[E0499]: cannot borrow `x[..]` as mutable more than once at a time
 --> src/lib.rs:4:18
  |
3 |     let a = &mut x[0];
  |                  ---- first mutable borrow occurs here
4 |     let b = &mut x[1];
  |                  ^^^^ second mutable borrow occurs here
5 |     println!("{} {}", a, b);
6 | }
  | - first borrow ends here
error: aborting due to previous error
```

借用检查器连这个简单的场景都理解不了，那它更不可能理解一些通用容器类型了，比如说树，尤其是出现不同的键对应相同的值的时候。

为了能“教育”借用检查器我们的所作所为是正确的，我们还是要使用非安全代码。比如，可变slice暴露了一个`split_at_mut`的方法，它接收一个slice然后返回两个可变slice。一个包括索引值左边所有的值，另一个包含右边所有的值。我们知道这个方法是安全的，因为两个slice没有重叠部分，也就不会出现别名问题。但是它的实现还是要涉及到非安全的内容：

``` Rust
fn split_at_mut(&mut self, mid: usize) -> (&mut [T], &mut [T]) {
    let len = self.len();
    let ptr = self.as_mut_ptr();
    assert!(mid <= len);
    unsafe {
        (from_raw_parts_mut(ptr, mid)),
         from_raw_parts_mut(ptr.offset(mid as isize), len - mid))
    }
}
```

这有一点难懂。为了避免两个`&mut`指向相同的值，我们通过裸指针显式创建了两个全新的slice。

不过迭代器产生可变引用的方法更加难懂。迭代器trait的定义如下：

``` Rust
trait Iterator {
    typr Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

这份定义里，`Self::Item`与`slef`没有直接关系。也就是说我们可以连续调用`next`很多次，并且同时保存着所有的结果。对于值的迭代器这么做完全可以，完全符合语义。对于共享引用这么做也没什么问题，因为允许任意过个共享引用指向同一个值（当然迭代器本身需要是独立于被共享内容的对象）。

但是可变引用就麻烦了。乍一看，可变引用完全不适用这个API，因为那会产生多个指向相同对象的可变引用。

可实际上它能够正常工作，这是因为迭代器是一个一次性对象。`IterMut`生成的东西最多只会生成一次，所以实际上我们没有生成多个指向相同数据的可变指针。

更不可思议的是，可变迭代器对于许多类型的实现甚至不需要非安全代码！

例如，下面是单向列表的代码：

``` Rust
type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

pub struct LinkedList<T> {
    head: Link<T>,
}

pub struct IterMut<'a, T: 'a>(Option<&'a mut Node<T>>);

impl<T> LinkedList<T> {
    fn iter_mut(&mut self) -> IterMut<T> {
        IterMut(self.head.as_mut().map(|node| &mut **node))
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node| {
            self.0 = node.next.as_mut().map(|node| &mut **node);
            &mut node.elem
        })
    }
}
```

这是可变slice：

``` Rust
use std::mem;

pub struct IterMut<'a, T: 'a>(&'a mut[T]);

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        let slice = mem::replace(&mut self.0, &mut []);
        if slice.is_empty() { return None; }

        let (l, r) = slice.split_at_mut(1);
        self.0 = r;
        l.get_mut(0)
    }
}

impl<'a, T> DoubleEndedIterator for IterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        let slice = mem::replace(&mut self.0, &mut []);
        if slice.is_empty() { return None; }

        let new_len = slice.len() - 1;
        let (l, r) = slice.split_at_mut(new_len);
        self.0 = l;
        r.get_mut(0)
    }
}
```

还有二叉树：

``` Rust
use std::collections::VecDeque;

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    left: Link<T>,
    right: Link<T>,
}

pub struct Tree<T> {
    root: Link<T>,
}

struct NodeIterMut<'a, T: 'a> {
    elem: Option<&'a mut T>,
    left: Option<&'a mut Node<T>>,
    right: Option<&'a mut Node<T>>,
}

enum State<'a, T: 'a> {
    Elem(&'a mut T),
    Node(&'a mut Node<T>),
}

pub struct IterMut<'a, T: 'a>(VecDeque<NodeIterMut<'a, T>>);

impl<T> Tree<T> {
    pub fn iter_mut(&mut self) -> IterMut<T> {
        let mut deque = VecDeque::new();
        self.root.as_mut().map(|root| deque.push_front(root.iter_mut()));
        IterMut(deque)
    }
}

impl<T> Node<T> {
    pub fn iter_mut(&mut self) -> NodeIterMut<T> {
        NodeIterMut {
            elem: Some(&mut self.elem),
            left: self.left.as_mut().map(|node| &mut **node),
            right: self.right.as_mut().map(|node| &mut **node),
        }
    }
}


impl<'a, T> Iterator for NodeIterMut<'a, T> {
    type Item = State<'a, T>;

    fn next(&mut self) -> Option<Self::Item> {
        match self.left.take() {
            Some(node) => Some(State::Node(node)),
            None => match self.elem.take() {
                Some(elem) => Some(State::Elem(elem)),
                None => match self.right.take() {
                    Some(node) => Some(State::Node(node)),
                    None => None,
                }
            }
        }
    }
}

impl<'a, T> DoubleEndedIterator for NodeIterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        match self.right.take() {
            Some(node) => Some(State::Node(node)),
            None => match self.elem.take() {
                Some(elem) => Some(State::Elem(elem)),
                None => match self.left.take() {
                    Some(node) => Some(State::Node(node)),
                    None => None,
                }
            }
        }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        loop {
            match self.0.front_mut().and_then(|node_it| node_it.next()) {
                Some(State::Elem(elem)) => return Some(elem),
                Some(State::Node(node)) => self.0.push_front(node.iter_mut()),
                None => if let None = self.0.pop_front() { return None },
            }
        }
    }
}

impl<'a, T> DoubleEndedIterator for IterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        loop {
            match self.0.back_mut().and_then(|node_it| node_it.next_back()) {
                Some(State::Elem(elem)) => return Some(elem),
                Some(State::Node(node)) => self.0.push_back(node.iter_mut()),
                None => if let None = self.0.pop_back() { return None },
            }
        }
    }
}
```

所有这些都是完全安全而且能稳定运行的！这已经超出了我们之前看过的简单结构体的例子：Rust能够理解你把一个可变引用安全地分解为多个部分。接下来我们可以通过Option永久地访问这个引用（或者像对于slice那样，替换为一个空的slice）。
