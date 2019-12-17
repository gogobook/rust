# Drop检查

> 源：[dropck.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/dropck.md) &nbsp; Commit: d870b6788ba078ba398f020305ef9210f7cbd740

我们已经知道生命周期给我们提供了一些很简单的规则，以保证我们永远不会读取悬垂引用。但是，到目前为止我们提到生命周期的长短时，指的都是非严格的关系。也就是说，当我们写`'a: 'b`的时候，`'a`其实也可以和`'b`一样长。乍一看，这一点没什么意义。本来也不会有两个东西被同时销毁的，不是吗？我们去掉下面的`let`表达式的语法糖看看：

``` Rust
let x;
let y;
```

``` Rust
{
    let x;
    {
        let y;
    }
}
```

每一个都创建了自己的作用域，可以很清楚地看出来一个在另一个之前被销毁。但是，如果是下面这样的呢？

``` Rust
let (x, y) = (vec![], vec![]);
```

有哪一个比另一个存活更长吗？答案是，没有，没有哪个严格地比另一个长。当然，x和y中肯定有一个比另一个先销毁，但是销毁的顺序是不确定的。并非只有元组是这样，复合结构体从Rust 1.0开始就不会保证它们的销毁顺序。

我们已经清楚了元组和结构体这种内置复合类型的行为了。那么Vec这样的类型又是什么样的呢？Vec必须通过标准库代码手动销毁它的元素。通常来说，所有实现了Drop的类型在临死前都有一次回光返照的机会。所以，对于实现了Drop的类型，编译器没有充分的理由判断它们的内容的实际销毁顺序。

可是我们为什么要关心这个？因为如果系统不够小心，就可能搞出来悬垂指针。考虑下面这个简单的程序：

``` Rust
struct Inspector<'a>(&'a u8);

fn main() {
    let (inspector, days);
    days = Box::new(1);
    inspector = Inspector(&days);
}
```

这段程序是正确且可以正常编译的。`days`并不严格地比`inspector`存活得更长，但这没什么关系。只要`inspector`还存活着，`days`就一定也活着。

可如果我们添加一个析构函数，程序就不能编译了！

``` Rust
struct Inspector<'a>(&'a u8);

impl<'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("再过{}天我就退休了！", self.0);
    }
}

fn main() {
    let (inspector, days);
    days = Box::new(1);
    inspector = Inspector(&days);
    // 如果days碰巧先被销毁了
    // 那么当销毁Inspector的时候，它会读取被释放的内存
}
```

```
error[E0597]: `days` does not live long enough
  --> src/main.rs:12:28
   |
12 |     inspector = Inspector(&days);
   |                             ^^^^ borrowed value does not live long enough
...
15 | }
   | - `days` dropped here while still borrowed
   |
   = note: values in a scope are dropped in the opposite order they are created

error: aborting due to previous error
```

实现`Drop`使得`Inspector`可以在销毁前执行任意的代码。一些通常认为和它生命周期一样长的类型可能实际上比它先销毁，而这会有潜在的问题。

有意思的是，只有泛型需要考虑这个问题。如果不是泛型的话，那么唯一可用的生命周期就是`'static`，而它确确实实会永远存在。这也就是这一问题被称之为“安全泛型销毁”的原因。安全泛型销毁是通过drop检查器执行的。我们还未涉及到drop检查器判断类型是否可用的细节，但其实我们之前已经讨论了这个问题的最主要规则：

**一个安全地实现Drop的类型，它的泛型参数生命周期必须严格地长于它本身**

遵守这一规则（大部分情况下）是满足借用检查器要求的必要条件，同时是满足安全要求的充分非必要条件。也就是说，如果类型遵守上述规则，它就一定可以安全地drop。

之所以并不总是满足借用检查器要求的必要条件，是因为有时类型借用了数据但是在Drop的实现里没有访问这些数据。

例如，上面的`Inspector`的这一变体就不会访问借用的数据：

``` Rust
struct Inspector<'a>(&'a u8, &'static str);

impl<'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("Inspector(_, {}) knows when *not* to inspect.", self.1);
    }
}

fn main() {
    let (inspector, days);
    days = Box::nex(1);
    inspector = Inspector(&days, "gadget);
    // 假设days碰巧先被销毁。
    // 可当Inspector被销毁时，它的析构函数也不会访问借用的days。
}
```

同样，这个变体也不会访问借用的数据：

``` Rust
use std::fmt;

struct Inspector<T: fmt::Display>(T, &'static str);

impl<T: fmt::Display> Drop for Inspector<T> {
    fn drop(&mut self) {
        println!("Inspector(_, {}) knows when *not* to inspect.", self.1);
    }
}

fn main() {
    let (inspector, days): (Inspector<&u8>, Box<u8>);
    days = Box::new(1);
    inspector = Inspector(&days, "gadget");
    // 假设days碰巧先被销毁。
    // 可当Inspector被销毁时，它的析构函数也不会访问借用的days。
}
```

但是，借用检查器在分析`main`函数的时候会拒绝上面两段代码，并指出`days`存活得不够长。

这是因为，当借用检查分析`main`函数的时候，它并不知道每个`Inspector`的`Drop`实现的内部细节。它只知道inspector的析构函数有访问借用数据的可能。

因此，drop检查器强制要求一个值借用的所有数据的生命周期必须严格长于值本身。

## 留一个后门

上面的类型检查的规则在未来有可能会松动。

当前的分析方法是很保守甚至苛刻的，它强制要求一个值借用的数据必须比值本身长寿，以保证绝对的安全。

未来的版本中，分析过程会更加精细，以减少安全的代码被拒绝的情况。比如上面的两个`Inspector`，它们知道在销毁过程中不应该被检查。

同时，有一个还未稳定的属性可以用来（非安全地）声明类型的析构函数保证不会访问过期的数据，即使类型的签名显示有这种可能存在。

这个属性是`my_dangle`，在[RFC 1327](https://github.com/rust-lang/rfcs/blob/master/text/1327-dropck-param-eyepatch.md)中被引入。我们可以这样将其放在上面的`Inspector`例子里：

``` Rust
struct Inspector<'a>(&'a u8, &'static str);

unsafe impl<#[may_dangle] 'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("Inspector(_, {}) knows when *not* to inspect.", self.1);
    }
}
```

使用这个属性要求`Drop`的实现被标为`unsafe`，因为编译器将不会检查有没有过期的数据（比如`self.0`）被访问。

这个属性可以赋给任意数量的生命周期和类型参数。下面这个例子里，我们声明我们不会访问有生命周期`'b`的引用背后的数据，而类型`T`也只会被用来转移或销毁。但是我们没有为`'a`和`U`添加属性，因为我们确实会用到这个生命周期和类型：

``` Rust
use std::fmt::Display;

struct Inspector<'a, 'b, T, U: Display>(&'a u8, &'b u8, T, U);

unsafe impl<'a, #[may_dangle] 'b, #[may_dangle] T, U: Display> Drop for Inspector<'a, 'b, T, U> {
    fn drop(&mut self) {
        println!("Inspector({}, _, _, {})", self.0, self.3);
    }
}
```

上面的例子中，哪些数据不会被用到是一目了然的。但是，有时候这些泛型参数会被间接地访问。间接访问的形式包括：

- 使用回调函数
- 通过调用trait方法

（在日后的版本里可能增加其他间接访问的途径。）

以下是使用回调的例子：

``` Rust
struct Inspector<T>(T, &'static str, Box<for <'r> fn(&'r T) -> String>);

impl<T> Drop for Inspector<T> {
    fn drop(&mut self) {
        // 如果T的类型是&'a _，self.2的调用可能访问借用的数据
        println!("Inspector({}, {}) unwittingly inspects expired data.",
                 (self.2)(&self.0), self.1);
    }
}
```

这是trait方法调用的例子：

``` Rust
use std::fmt;

struct Inspector<T: fmt::Display>(T, &'static str);

impl<T: fmt::Display> Drop for Inspector<T> {
    fn drop(&mut drop) {
        // 下面有一个对<T as Display>::fmt的隐藏调用，
        // 当T的类型是&'a _时，可能访问借用数据
        println!("Inspector({}, {}) unwittingly inspects expired data.",
                 self.0, self.1);
    }
}
```

当然，这些访问可以进一步地被隐藏在其他的析构函数调用的方法里，而不仅是直接写在函数中。

上面的几个例子里，`&'a u8`都在析构函数里被访问了。如果给它添加`#[may_dangle]`属性，这些类型很可能会产生借用检查器无法捕捉的错误，引发不可预料的灾难。所以最好能避免使用这个属性。

## drop检查的故事讲完了吗？

我们发现，在写非安全代码时，其实并不用关心是否满足drop检查器的要求。不过有一个特殊的场景是例外的，我们将在下一章讲到它。
