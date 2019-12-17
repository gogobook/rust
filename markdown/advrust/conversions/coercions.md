# 强制类型转换

> 源：[coercions.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/coercions.md) &nbsp; Commit: d870b6788ba078ba398f020305ef9210f7cbd740

在一些特定场景中，类型会被隐式地强制转换。这种转换通常导致类型被“弱化”，主要针对指针和生命周期。主要目的是让Rust适用于更多的场景，并且基本上是无害的。

强制转换包括下面几种：

如下几种类型之间允许进行强制转换：

- 传递性：当`T_1`可以强制转换为`T_2`且`T_2`可以强制转换为`T_3`时，`T_1`就可以强制转换为`T_3`
- 指针弱化：
  * `&mut T`转换为`&T`
  * `*mut T`转换为`*const T`
  * `&T`转换为`*const T`
  * `&mut T`转换为`*mut T`
- Unsize：如果`T`实现了`CoerceUnsized<U>`，那么`T`可以强制转换为`U`
- 强制解引用：如果`T`可以解引用为`U`（比如`T: Deref<Target=U>`），那么`&T`类型的表达式`&x`可以强制转换为`&U`类型的`&*x`

所有的指针类型（包括Box和Rc这些智能指针）都实现了`CoerceUnsized<Pointer<U>> for Pointer<T> where T: Unsize<U>`。Unsize只能被自动实现，并且实现如下转换方式：

- `[T; n]` => `[T]`
- `T` => `Trait`，其中`T: Trait`
- `Foo<..., T, ...>` => Foo<..., U, ...>`，其中
  * `T: Unsize<U>`
  * `Foo`是一个结构体
  * 只有`Foo`的最后一个成员是和`T`有关的类型
  * 其他成员的类型与`T`无关
  * 如果最后一个成员的类型是`Bar<T>`，那么必须有`Bar<T>: Unsize<Bar<U>>`

强制转换会在“强制转换位置”处发生。每一个显式声明了类型的位置都会引起到该类型的强制转换。但如果必须进行类型推断，则不会发生类型转换。表达式`e`到类型`U`的强制转换位置包括：

- let表达式，静态变量或者常量：`let x: U = e`
- 函数的参数：`takes_a_U(e)`
- 函数返回值：`fn foo() -> U {e}`
- 结构体初始化：`Foo { some_u: e }`
- 数组初始化：`let x: [U; 10] = [e, ...]`
- 元组初始化：`let x: (U, ..) = (e, ..)`
- 代码块中的最后一个表达式：`let x: U = { ..; e }`

注意，在匹配trait的时候不会发生强制类型转换（receiver除外，具体见下）。也就是说，如果为`U`实现了一个trait，`T`可以强制转换为`U`，并不能认为`T`也实现了这个trait。例如，下面的代码无法通过类型检查，虽然`t`可以强制转换为`&T`，而且有一个`&T`的trait实现。

``` Rust
trait Trait {}

fn foo<X: Trait>(t: X) {}

impl<'a> Trait for &'a i32 {}

fn main() {
    let t: &mut i32 = &mut 0;
    foo(t);
}
```

```text
error[E0277]: the trait bound `&mut i32: Trait` is not satisfied
 --> src/main.rs:9:5
  |
9 |     foo(t);
  |     ^^^ the trait `Trait` is not implemented for `&mut i32`
  |
  = help: the following implementations were found:
            <&'a i32 as Trait>
note: required by `foo`
 --> src/main.rs:3:1
  |
3 | fn foo<X: Trait>(t: X) {}
  | ^^^^^^^^^^^^^^^^^^^^^^
error: aborting due to previous error
```
