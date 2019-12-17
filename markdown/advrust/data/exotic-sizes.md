# 特殊尺寸类型

> 原文跟踪[exotic-sizes.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/exotic-sizes.md) &emsp; Commit: 7f019ec5c87da39fe0b9b5149e413d914528e945

大多数情况下，我们希望类型具有可静态确定的、非负的大小。 但在Rust中，并非总是如此。

## 动态大小类型（DST）

Rust支持动态大小类型（DST）：无法静态地确定大小或对齐的类型。 从表面上看，这有点荒谬：Rust必须知道某些东西的大小和对齐单位才能正确使用它！ 在这方面，DST不是正常的类型。 因为它们缺少静态可知的大小，所以这些类型只能存在于指针背后。 因此，任何指向DST的指针都会变成一个宽指针：由指针和"使其成为完整类型"的信息组成（下面将详细介绍）。

Rust语言暴露了两个主要的DST：

* trait objects: `dyn MyTrait`
* slices: `[T]`, `str`, and others

trait对象表示实现了其指定的特质的某种类型。 原始类型通过一个包含了该类型所有信息的vtable被擦除，以支持运行时反射。 vtable指针让trait对象指针成为了一个完整类型：在运行时，可以从vtable中动态地获悉指针所指向的对象的大小。

切片只是一些连续内存的一个视图——通常是数组或Vec。 切片指针包含的额外信息只不过是它指向的元素的数量。 指针所指向对象的运行时大小只是单个元素的大小乘以元素的数量。

`Struct`实际上可以直接存储单个DST作为它们的最后一个字段，但这同时也让它们成为了DST：

```rust
// Can't be stored on the stack directly
struct MySuperSlice {
    info: u32,
    data: [u8],
}
```

尽管因为缺少构造手段，这种类型在很大程度上是没用的。 目前，唯一能够构造出自定义DST的方法是，在类型上增加泛型参数，并加以`?Sized`修饰：

```rust
struct MySuperSliceable<T: ?Sized> {
    info: u32,
    data: T
}

fn main() {
    let sized: MySuperSliceable<[u8; 8]> = MySuperSliceable {
        info: 17,
        data: [0; 8],
    };

    let dynamic: &MySuperSliceable<[u8]> = &sized;

    // prints: "17 [0, 0, 0, 0, 0, 0, 0, 0]"
    println!("{} {:?}", dynamic.info, &dynamic.data);
}
```

(是的，现在，自定义DST是一个很大程度上不够成熟的特性.)

## 零大小类型（ZST）

Rust还允许声明不占用空间的类型：

```rust
struct Nothing; // No fields = no size

// All fields have no size = no size
struct LotsOfNothing {
    foo: Nothing,
    qux: (),      // empty tuple has no size
    baz: [u8; 0], // empty array has no size
}
```

就其自身而言，零大小类型（ZST）由于显而易见的原因非常没用。然而，正如Rust中许多奇特的内存布局决策一样，它们的潜力是发挥在泛型上下文中的：Rust大多数情况下能够理解到，构造或存储ZST的任何操作都可以简化为no-op。首先，存储ZST甚至是没有意义的——它不占用任何空间。此外，ZST类型的值只有一个，所以任何需要用到它的时候，都可以凭空构造一个——这也是一种no-op，毕竟它不占用任何空间。

其中一个最极端的例子是`Set`和`Map`。给定`Map <Key，Value>`，通常`Set <Key>`会被实现为`Map <Key，UselessJunk>`的一层薄薄的封装。在许多语言中，这需要为`UselessJunk`分配空间，并且需要储存和读取`UselessJunk`后再丢弃它。对编译器来说，证明这种不必要是很困难的。

但是在Rust中，我们可以说`Set <Key> = Map <Key，（）>`。现在Rust编译器能够静态地得知每个读取和存储都是无用的，并且没有分配任何空间。结果就是，通过简单地单态化`HashMap`实现的`HashSet`，在`HashMap`的值上不需要花费任何额外开销。

safe的代码不必担心ZST，但在unsafe的代码中，必须小心ZST类型带来的后果。特别是，对指针偏移是no-op，并且当申请零大小的内存分配时，标准分配器可以返回`null`，因而无法区分这是否意味着出现了“内存不足”错误。

## 空类型

Rust甚至还允许声明无法实例化的类型。 这些类型只能在类型层面进行讨论，而不能在值的层面进行讨论。 可以通过指定不带变量的枚举来声明一个空类型：

```rust
enum Void {} // No variants = EMPTY
```

空类型比ZST更加边缘化。 空类型的主要积极示例是类型层面的不可达性。 例如，假设一个API通常需要返回一个结果，但实际上，这个API在特定情况是绝对可靠的。 这一点，通过返回一个`Result <T，Void>`，就可以在类型层面表达出来。 API的使用者可以自信地对这样的结果进行拆箱，因为已经知道这个值在静态上不可能是`Err`——否则就意味着需要提供一个`Void`类型的值，这是不可能的。

原则上，Rust可以基于这个事实做一些有趣的分析和优化。 例如，`Result <T，Void>`仅表示为`T`，因为`Err`情况实际上并不存在（严格来说，这只是一个无法保证的优化，因此，例如将一个`Result <T，Void>`转换为另一个`Result <T，Void>`仍然是未定义行为）。

以下代码也可以通过编译：

```rust
enum Void {}

let res: Result<u32, Void> = Ok(0);

// Err doesn't exist anymore, so Ok is actually irrefutable.
let Ok(num) = res;
```

但是这个trick现在还不能用。

关于空类型的最后一个细微的细节是，构造一个空类型的裸指针是合法的，但是解引用它们是未定义的行为，因为这没有意义。

我们建议不要使用`* const Void`对C的`void *`类型进行模拟。 很多人一开始会这样做，但很快就遇到了麻烦，因为Rust没有任何的安全措施来防止尝试用不安全的代码去实例化空类型，如果你这样做，那就是未定义行为。 这尤其成问题，因为开发人员习惯将原始指针转换为引用，而`&Void`也是未定义行为。

`* const（）`（或等效的）适用于`void *`，并且可以在没有任何安全问题的情况下成为引用。 它仍然不会阻止您尝试读取或写入值，但至少它会编译为`no-op`而不是UB。

## 外部类型

有一个已经被接受的[RFC](https://github.com/rust-lang/rfcs/blob/master/text/1861-extern-types.md)为未知大小的类型增加了一种合适的类型，称为`extern`类型，这将使Rust开发人员更准确地模拟C的`void *`和其他"声明了但未定义"类型的内容。 但是，从Rust 2018开始，这个特性在`size_of :: <MyExternType>（）`应该如何表现上陷入了困境。

