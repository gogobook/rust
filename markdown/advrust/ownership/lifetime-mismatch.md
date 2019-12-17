# 生命周期限制

> 原文跟踪[lifetime-mismatch.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/lifetime-mismatch.md) &emsp; Commit: d870b6788ba078ba398f020305ef9210f7cbd740

给出以下代码：

```rust
struct Foo;

impl Foo {
    fn mutate_and_share(&mut self) -> &Self { &*self }
    fn share(&self) {}
}

fn main() {
    let mut foo = Foo;
    let loan = foo.mutate_and_share();
    foo.share();
}
```

人们可能期望它能够编译。我们称之为mutate_and_share临时借用 foo，但后来只返回共享引用。因此，我们期望foo.share()成功，因为foo不应该可变地借用。

但是当我们尝试编译它时：

```rust
error[E0502]: cannot borrow `foo` as immutable because it is also borrowed as mutable
  --> src/lib.rs:11:5
   |
10 |     let loan = foo.mutate_and_share();
   |                --- mutable borrow occurs here
11 |     foo.share();
   |     ^^^ immutable borrow occurs here
12 | }
   | - mutable borrow ends here
```

发生了什么？好吧，我们得到以下内容：

```rust
struct Foo;

impl Foo {
    fn mutate_and_share<'a>(&'a mut self) -> &'a Self { &'a *self }
    fn share<'a>(&'a self) {}
}

fn main() {
    'b: {
        let mut foo: Foo = Foo;
        'c: {
            let loan: &'c Foo = Foo::mutate_and_share::<'c>(&'c mut foo);
            'd: {
                Foo::share::<'d>(&'d foo);
            }
        }
    }
}
```

由于`loan`的生命周期和`mutate_and_share`的签名, 生命周期系统强制扩展`&mut foo`的生命周期为`'c`，然后当我们试着调用`share`的时候，它说我们正试图别名`&'c mut foo`！

根据我们实际关注的引用语义，这个程序显然是正确的，但是生命周期系统太粗糙而无法处理。
