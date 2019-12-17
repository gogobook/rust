# 高阶特质界限(HRTBs)

> 原文跟踪[hrtb.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/hrtb.md) &emsp; Commit: 0e6c680ebd72f1860e46b2bd40e2a387ad8084ad

Rust的`Fn`特征有点神奇。 例如，我们可以写以下代码：

```rust
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F>
    where F: Fn(&(u8, u16)) -> &u8,
{
    fn call(&self) -> &u8 {
        (self.func)(&self.data)
    }
}

fn do_it(data: &(u8, u16)) -> &u8 { &data.0 }

fn main() {
    let clo = Closure { data: (0, 1), func: do_it };
    println!("{}", clo.call());
}
```

如果我们尝试以与生命周期部分相同的方式天真地去除这些代码，我们会遇到一些麻烦：

```rust
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F>
    // where F: Fn(&'??? (u8, u16)) -> &'??? u8,
{
    fn call<'a>(&'a self) -> &'a u8 {
        (self.func)(&self.data)
    }
}

fn do_it<'b>(data: &'b (u8, u16)) -> &'b u8 { &'b data.0 }

fn main() {
    'x: {
        let clo = Closure { data: (0, 1), func: do_it };
        println!("{}", clo.call());
    }
}
```

我们究竟应该用F的特征界限来表达生命周期？ 我们需要在那里提供一些生命周期，但是在我们进入调用体之前，我们关心的生命周期无法命名！ 而且，这不是一些固定的生命周期; 调用适用于任何生命周期和`&self`恰好在那一点上。

这项工作需要高等级特质界限（HRTBs）。 我们这种方式如下：

```rust
where for<'a> F: Fn(&'a (u8, u16)) -> &'a u8,
```

(其中`Fn（a，b，c） - > d`本身就是不稳定的真实`Fn`特征的糖）

`for <'a>`可以读作`对于'a`的所有选择，并且基本上产生的`F`必须满足无限特征边界列表。在`Fn`特征之外的地方我们遇到`HRTB`并不多，甚至对于那些普通案例我们都有一个很好的魔法糖。
