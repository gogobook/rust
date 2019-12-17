# 外部函数接口

> 源：[ffi.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/ffi.md) &nbsp; Commit: b3d532f55bea88bf34aae8d3b6af4c5d1ceaf31e

## 介绍

这个教程会使用[snappy](https://github.com/google/snappy)压缩/解压缩库来介绍外部代码绑定的编写方法。Rust目前还不能直接调用C++的库，但是snappy有C的接口（文档在`snappy-c.h`中）。

### 关于libc的说明

接下来很多的例子会使用[`libc` crate](https://crates.io/crates/libc)，它为我们提供了很多C类型的定义。如果你要亲自尝试一下这些例子的话，你需要把`libc`添加到你的`Cargo.toml`:

``` Toml
[dependencies]
libc = "0.2.0"
```
然后在你的crate的根文件插入一句`extern crate libc;`

### 调用外部函数

下面是一个调用外部函数的小例子，安装了snappy才能编译成功。

``` Rust
extern crate libc;
use libc::size_t;

#[link(name = "snappy")]
extern {
    fn snappy_mx_compressed_length(source_length: size_t) -> size_t;
}

fn main() {
    let x = unsafe { snappy_max_compressed_length(100) };
    println!("max compressed length of a 100 byte buffer: {}", x);
}
```

`extern`代码块中是外部库的函数签名的列表，这个例子中使用的是平台相关的C的ABI。`#[link(...)]`属性用来构建一个链接snappy库的链接器，以便解析库中的符号(symbol)。

外部函数都被认为是不安全的，所以对它们的调用必须包装在`unsafe {}`中，也就是向编译器承诺块中的代码都是安全的。C的库经常暴露非线程安全的接口，而且几乎所有的接受指针参数的函数都是不合法的，因为指针可能是悬垂指针，而裸指针不符合Rust的内存安全模型。

在声明外部函数的参数类型时，Rust编译器不能检查声明的正确性，所以我们需要自己保证它是正确的，这也是运行期正确绑定的条件之一。

`extern`块还可以继续扩展，包含所有的snappy API：

``` Rust
extern crate libc;
use libc::{c_int, size_t};

#[link(name = "snappy")]
extern {
    fn snappy_compress(input: *const u8,
                       input_length: size_t,
                       compressed: *mut u8,
                       compressed_length: *mut size_t) -> c_int;
    fn snappy_uncompress(compressed: *const u8,
                         compressed_length: size_t,
                         uncompressed: *mut u8,
                         uncompressed_length: *mut size_t) -> c_int;
    fn snappy_max_compressed_length(source_length: size_t) -> size_t;
    fn snappy_uncompressed_length(compressed: *const u8,
                                  compressed_length: size_t,
                                  result: *mut size_t) -> c_int;
    fn snappy_validate_compressed_buffer(compressed: *const u8,
                                         compressed_length: size_t) -> c_int;
}
```

## 创建安全接口

原生的C API进行封装，以保证内存安全，还有使用vector等高级概念。库可以选择只暴露安全的、高级的接口，并隐藏非安全的内部细节。

我们使用`slice::raw`模块封装接受内存块的函数，这个模块会把Rust的vector转换为内存的指针。Rust的vector是一块连续的内存。它的长度是当前包含的元素的数量，容量是分配内存可存储的元素的总数。长度是小于等于容量的。

``` Rust
pub fn validate_compressed_buffer(src: &[u8]) -> bool {
    unsafe {
        snappy_validate_compressed_buffer(src.as_ptr(), src.len() as size_t) == 0
    }
}
```

上方的`validate_compressed_buffer`包装器用到了`unsafe`代码块，但是函数签名里没有`unsafe`关键字，这说明它保证函数调用对所有的输入都是安全的。

`snappy_compress`和`snappy_uncompress`函数更复杂一些，因为它们需要分配一块空间储存输出的结果。

`snappy_max_compressed_length`函数可以用来分配一段最大容积内的vector，以保存输出的结果。这个vector可以传递给`snappy_compress`函数作为输出参数。还会传递一个输出参数获取压缩后的真实长度，以便设置返回值的长度。

``` Rust
pub fn compress(src: &[u8]) -> Vec<u8> {
    unsafe {
        let srclen = src.len() as size_t;
        let psrc = src.as_ptr();

        let mut dstlen = snappy_max_compressed_length(srclen);
        let mut dst = Vec::with_capacity(dstlen as usize);
        let pdst = dst.as_mut_ptr();

        snappy_compress(psrc, srclen, pdst, &mut dstlen);
        dst.set_len(dstlen as usize);
        dst
    }
}
```

解压缩也是类似的，因为snappy的压缩格式中保存了未压缩时的大小，函数`snappy_uncompressed_length`可以获取需要的缓存区的尺寸。

``` Rust
pub fn uncompress(src: &[u8]) -> Option<Vec<u8>> {
    unsafe {
        let srclen = src.len() as size_t;
        let psrc = src.as_ptr();

        let mut dstlen: size_t = 0;
        snappy_uncompressed_length(psrc, srclen, &mut dstlen);

        let mut dst = Vec::with_capacity(dstlen as usize);
        let pdst = dst.as_mut_ptr();

        if snappy_uncompress(psrc, srclen, pdst, &mut dstlen) == 0 {
            dst.set_len(dstlen as usize);
            Some(dst)
        } else {
            None // SNAPPY_INVALID_INPUT
        }
    }
}
```

接下来，我们添加一些测试用例来展示如何使用它们。

``` Rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid() {
        let d = vec![0xde, 0xad, 0xd0, 0x0d];
        let c: &[u8] = &compress(&d);
        assert!(validate_compressed_buffer(c));
        assert!(uncompress(c) == Some(d));
    }

    #[test]
    fn invalid() {
        let d = vec![0, 0, 0, 0];
        assert!(!validate_compressed_buffer(&d));
        assert!(uncompress(&d).is_none());
    }

    #[test]
    fn empty() {
        let d = vec![];
        assert!(!validate_compressed_buffer(&d));
        assert!(uncompress(&d).is_none());
        let c = compress(&d);
        assert!(validate_compressed_buffer(&c));
        assert!(uncompress(&c) == Some(d));
    }
}
```

## 析构函数

外部库经常把资源的所有权返还给调用代码。如果是这样，我们必须用Rust的析构函数保证所有的资源都被释放了（特别是在panic的情况下）。

更多关于析构函数的内容，请见[Drop trait](https://doc.rust-lang.org/std/ops/trait.Drop.html)。

## C代码到Rust函数的回调

一些外部库需要用到回调向调用者报告当前状态或者中间数据。我们是可以把Rust写的函数传递给外部库的。要求是回调函数必须标为`extern`并遵守正确的调用规范，以保证C代码可以调用它。

然后回调函数会通过注册调用传递给C的库，并在外部库中被触发。

下面是一个简单的例子。

Rust代码：

``` Rust
extern fn callback(a: i32) {
    println!("I'm called from C with value {0}", a);
}

#[link(name = "extlib")]
extern {
   fn register_callback(cb: extern fn(i32)) -> i32;
   fn trigger_callback();
}

fn main() {
    unsafe {
        register_callback(callback);
        trigger_callback(); // 触发回调
    }
}
```

C代码：

``` C
typedef void (*rust_callback)(int32_t);
rust_callback cb;

int32_t register_callback(rust_callback callback) {
    cb = callback;
    return 1;
}

void trigger_callback() {
  cb(7); // Will call callback(7) in Rust.
}
```

这个例子中，Rust的`main()`要调用C的`trigger_callback()`，而这个函数会反过来调用Rust中的`callback()`。

### 将Rust对象作为回调

之前的例子演示了C代码如何调用全局函数。但是很多情况下回调也可能是一个Rust对象，比如说封装了某个C的结构体的Rust对象。

要实现这一点，我们可以传递一个指向这个对象的裸指针给C的库。C的库接下来可以将指针转换为Rust的对象。这样回调函数就可以非安全地访问相应的Rust对象了。

``` Rust
#[repr(C)]
struct RustObject {
    a: i32,
    // 其他成员……
}

extern "C" fn callback(target: *mut RustObject, a: i32) {
    println!("I'm called from C with value {0}", a);
    unsafe {
        // 用回调函数接收的值更新RustObject的值：
        (*target).a = a;
    }
}

#[link(name = "extlib")]
extern {
   fn register_callback(target: *mut RustObject,
                        cb: extern fn(*mut RustObject, i32)) -> i32;
   fn trigger_callback();
}

fn main() {
    // 创建回调用到的对象：
    let mut rust_object = Box::new(RustObject { a: 5 });

    unsafe {
        register_callback(&mut *rust_object, callback);
        trigger_callback();
    }
}
```

C代码：

``` C
typedef void (*rust_callback)(void*, int32_t);
void* cb_target;
rust_callback cb;

int32_t register_callback(void* callback_target, rust_callback callback) {
    cb_target = callback_target;
    cb = callback;
    return 1;
}

void trigger_callback() {
  cb(cb_target, 7); // 调用Rust的callback(&rustObject, 7)
}
```

### 异步回调

上面给出的例子里，回调都是外部C库的直接的函数调用。当前线程的控制权从Rust转移到C再转移回Rust，不过最终回调都是在调用触发回调的函数的线程里执行的。

如果外部库启动了自己的线程，并在那个线程里调用回调函数，情况就变得复杂了。这时再访问回调中的Rust数据结构是非常不安全的，必须使用正常地同步机制。除了Mutex等传统的同步机制，还有另一个选项就是使用channel（在`std::sync::mpsc`中）将数据从触发回调的C线程传送给一个Rust线程。

如果一个异步回调使用了一个Rust地址空间里的对象，一定要注意，在这个对象销毁之后C的库不能再调用任何的回调。我们可以在对象的析构函数里注销回调，并且重新设计库确保毁掉注销后就不会被调用了。

## 链接

`extern`代码块上的`link`属性用于指导rustc如何链接到一个本地的库。现在`link`属性有两种可用的形式：

- `#[link(name = "foo")]`
- `#[link(name = "foo", kind = "bar")]`

两种形式中，`foo`都是我们要链接的本地库的名字。而第二种形式中的`bar`是要链接的本地库的类型。目前有三种已知的本地库类型：

- 动态 - `#[link(name = "readline")]`
- 静态 - `#[link(name = "my_build_dependency", kind = "static")]`
- 框架 - `#[link(name = "CoreFundation", kind = "framework")]`

注意，框架只适用于MacOS平台。

不同的`kind`表明本地库以不同的方式参与链接。从链接器的角度看，Rust编译器产生两种输出结果：部分结果(rlib/staticlib)和最终结果(dylib/binary)。本地动态库和框架依赖可以被最终结果使用，而静态库则不会，因为静态库是直接集成在接下来的输出里的。

举几个这个模型用法的例子：

- 本地构建依赖。有时候编写Rust代码需要一些C/C++作为补充，但是把C/C++代码以一个库的形式发布却不容易。这种情况下，代码应该包装在`libfoo.a`中，然后Rust的crate会声明一个依赖`#[link(name = "foo", kind = "static")]`。
不管crate最终以哪种形式输出，本地静态库都会被包含在输出中，这表明发布静态库并不必要。

- 普通动态库。通用的系统库（比如`readline`）在许多系统中都支持，而我们经常遇到找不到库的本地备份的的情况。如果这样的依赖被包含在Rust的crate中，部分结果（比如rlib）不会链接到这个库中。但是如果rlib被最终结果包含了，本地库也会被链接。

在MacOS中，框架和动态库具有相同的语义。

## 非安全代码块

有一些操作，比如解引用裸指针、或者调用被标为unsafe的函数，它们只能存在于非安全代码块中。非安全代码块隔离了非安全性，并向编译器承诺非安全性不会影响到块以外的代码。

非安全函数则不同，它们声明非安全性一定会影响到函数之外。一个非安全函数写法如下：

``` Rust
unsafe fn kaboom(ptr: *const i32) -> i32 { *ptr }
```

这个函数只能在`unsafe`代码块或者另外一个`unsafe`函数里被调用。

## 访问外部全局变量

外部API经常暴露一些全局变量，用于记录全局状态等。为了访问这些变量，你需要在`extern`块中用`static`关键字声明它们：

``` Rust
extern crate libc;

#[link(name = "readline")]
extern {
    static rl_readline_version: libc::c_int;
}

fn main() {
    println!("You have readline version {} installed.",
             unsafe { rl_readline_version as i32 });
}
```

有时也可能需要通过外部的接口修改全局状态。如果要这么做，静态变量还要添加`mut`，让我们可以修改它们。

``` Rust
extern crate libc;

use std::ffi::CString;
use std::ptr;

#[link(name = "readline")]
extern {
    static mut rl_prompt: *const libc::c_char;
}

fn main() {
    let prompt = CString::new("[my-awesome-shell] $").unwrap();
    unsafe {
        rl_prompt = prompt.as_ptr();

        println!("{:?}", rl_prompt);

        rl_prompt = ptr::null();
    }
}
```

注意，所有和`static mut`的操作都是非安全的，不管是读还是写。处理全局可变状态的时候一定要格外的小心。

## 外部调用规范

大多数外部代码都暴露C的ABI，而Rust默认根据平台相关的C的调用规范调用外部函数。还有一些外部函数使用其他的规范，最典型的就是WindowsAPI。Rust也有方法告诉编译器使用哪种规范：

``` Rust
extern crate libc;

#[cfg(all(target_os = "win32", target_arch = "x86"))]
#[link(name = "kernel32")]
#[allow(non_snake_case)]
extern "stdcall" {
    fn SetEnvironmentVariableA(n: *const u8, v: *const u8) -> libc::c_int;
}
```

这段代码作用于整个`extern`代码块。支持的ABI包括：

- `stdcall`
- `appcs`
- `cdecl`
- `fastcall`
- `vectorcall` 这个目前被`abi_vectorcall`隐藏着，不允许修改。
- `Rust`
- `rust-intrinsic`
- `system`
- `C`
- `win64`
- `sysv64`

列表中所有的abi都是自解释的，但是`system`可能会显得有些奇怪。它的意思是选择一个合适的与目标库通信的ABI。比如，在win32的x86架构上，它实际使用的是`stdcall`。而在x86_64上，Windows使用`C`调用规范，所以它实际使用的是`C`。这意味着在我们之前的例子中，我们可以使用`extern "system" { ... }`为所有的Windows系统定义块，而不仅仅是x86的平台。

## 与外部代码互用性

只有给一个结构体指定了`#[repr(C)]`，Rust才保证结构体的布局与平台的C的表示方法相兼容。`#[repr(C, packed)]`可以让结构体成员之间无填充。`#[repr(C)]`也可以作用于枚举类型。

Rust的`Box<T>`用一个非空的指针指向它包含的对象。但是，这些指针不能手工创建，而是要由内部分配器去管理。引用可以安全地等同于非空指针。不过，违背借用检查和可变性规则就不能保证是安全的了，所以在需要使用指针的地方我们尽量使用裸指针，因为编译器不会对它做过多的限制。

Vector和String拥有相同的内存布局，而且`vec`和`str`模块里也有一些与C API相关的工具。但是，字符串不是以`\0`结尾的。如果你想要一个与C兼容的Null结尾的字符串，你应该使用`std::ffi`模块中的`CString`类型。

[crate.io的`libc` crate]`(https://crates.io/crates/libc)在`libc`模块中包含了C标准库的类型别名和函数定义，而Rust默认链接`libc`和`libm`。

## 可变函数

在C中，函数可以是“可变的”，也就是说可以接收可变数量的参数。在Rust中可以在外部函数声明的参数类表中插入`...`实现这一点：

``` Rust
extern {
    fn foo(x: i32, ...);
}

fn main() {
    unsafe {
        foo(10, 20, 30, 40, 50);
    }
}
```

普通的Rust函数不能是可变的：

``` Rust
// 这段不能通过编译
fn foo(x: i32, ...) { }
```

## 空指针优化

一些Rust类型被定义为永不为`null`，包括引用（`&T`、`&mut T`）、`Box<T>`、以及函数指针（`extern "abi" fn()`）。可是在使用C的接口时，指针是经常可能为`null`的。看起来似乎需要用到`transmute`或者非安全代码来处理各种混乱的类型转换。但是，Rust其实提供了另外的方法。

一些特殊情况中，`enum`很适合做空指针优化，只要它包含两个变量，其中一个不包含数据，而另外一个包含一个非空类型的成员。这样就不需要额外的空间做判断了：给那个包含非空成员的变量传递一个`null`，用它来表示另外那个空的变量。这种行为虽然被叫做“优化”，但是和其他的优化不同，它只适用于合适的类型。

最常见的受益于空指针优化的类型是`Option<T>`，其中`None`可以用`null`表示。所以`Option<extern "C" fn(c_int) - > c_int>`就很适合表示一个使用C ABI的可为空的函数指针（对应于C的`int (*)(int)`）。

下面是一个刻意造出来的例子。假设一些C的库提供了注册回调的方法，然后在特定的条件下调用回调。回调接受一个函数指针和一个整数，然后用这个整数作为参数调用指针指向的函数。所以我们会向FFI边界的两侧都传递函数指针。

``` Rust
extern crate libc;
use libc::c_int;

extern "C" {
    // 注册回调。
    fn register(cb: Option<extern "C" fn(Option<extern "C" fn(c_int) -> c_int>, c_int) -> c_int>);
}

// 这个函数其实没什么实际的用处。它从C代码接受一个函数指针和一个整数，
// 用整数做参数调用指针指向的函数，并返回函数的返回值。
// 如果没有指定函数，那默认就返回整数的平方。
extern "C" fn apply(process: Option<extern "C" fn(c_int) -> c_int>, int: c_int) -> c_int {
    match process {
        Some(f) => f(int),
        None    => int * int
    }
}

fn main() {
    unsafe {
        register(Some(apply));
    }
}
```

C的代码是像这样的：

``` C
void register(void (*f)(void (*)(int), int)) {
    ...
}
```

看，并不需要`transmute`！

## C调用Rust

你可能想要用某种方式编译Rust，让C可以直接调用它。这件事很简单，只需要做少数的处理：

``` Rust
#[no_mangle]
pub extern fn hello_rust() -> *const u8 {
    "Hello, world!\0".as_ptr()
}
```

`extern`让它对应的函数符合C的调用规范，在上面的[外部调用规范](https://doc.rust-lang.org/nomicon/ffi.html#foreign-calling-conventions)一节有详细讨论。`no_mangle`属性关闭Rust的name mangling，让它更方便被链接。

## FFI和panic

使用FFI的时候要格外注意`panic!`。跨越FFI边界的`panic!`属于未定义行为。如果你写的代码可能会panic，你应该使用`catch_unwind`在一个闭包里执行它：

``` Rust
use std::panic::catch_unwind;

#[no_mangle]
pub extern fn oh_no() -> i32 {
    let result = catch_unwind(|| {
        panic!("Oops!");
    });
    match result {
        Ok(_) => 0,
        Err(_) => 1,
    }
}

fn main() {}
```

请注意，`catch_unwind`只能捕获可展开的panic，不能捕获abort。更多的信息请参考`catch_unwind`的文档。

## 表示不透明结构体

有时候，C的库要提供一个指针指向某个东西，但又不想让你知道那个东西的内部细节。最简单的方式是使用`void *`：

``` C
void foo(void *arg);
void bar(void *arg);
```

在Rust中我们可以用`c_void`类型表示它：

``` Rust
extern crate libc;

extern "C" {
    pub fn foo(arg: *mut libc::c_void);
    pub fn bar(arg: *mut libc::c_void);
}
```

这是一个完全合法的方法。不过，我们其实还可以做得更好。要解决这个问题，一些C库可能会创建一个结构体，可结构体的细节和内存布局是私有的。这样提高了类型的安全性。这种结构体被称为”不透明“的。下面是一个C的例子：

``` C
struct Foo; /* Foo是一个接口，但它的内容不属于公共接口 */
struct Bar;
void foo(struct Foo *arg);
void bar(struct Bar *arg);
```

在Rust中，我们可以使用枚举来创建我们自己的不透明类型：

``` Rust
#[repr(C)] pub struct Foo { _private: [u8; 0] }
#[repr(C)] pub struct Bar { _private: [u8; 0] }

extern "C" {
    pub fn foo(arg: *mut Foo);
    pub fn bar(arg: *mut Bar);
}
# fn main() {}
```

给结构体一个私有成员而不给它构造函数，这样我们就创建了一个不透明的类型，而且我们不能在模块之外实例化它。（没有成员的结构体可以在任何地方实例化）因为我们希望在FFI中使用这个类型，我们必须加上`#[repr(C)]`。还为了避免在FFI中使用`()`的时候出现警告，我们用了一个空数组。空数组和空类型的行为一致，同时它还是FFI兼容的。

但因为`Foo`和`Bar`是不同的类型，我们需要保证两者之间的类型安全性，所以我们不能把`Foo`的指针传递给`bar()`。

注意，用空枚举作为FFI类型是一个很不好的设计。编译器将空枚举视为不可达的空类型，所以使用`&Empty`类型的值是很危险的，这可能导致很多程序中的问题（触发未定义行为）。
