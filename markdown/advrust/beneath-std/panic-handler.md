# #[panic_handler]

> 原文跟踪[panic-handler.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/panic-handler.md) &emsp; Commit: 676e7d1aaa82fa9e1ba4de789ca904c6c0cda8d6

`＃[panic_handler]`用于定义`＃！[no_std]`应用程序中`panic！`的行为。`＃[panic_handler]`属性必须应用于具有签名`fn（＆PanicInfo）的函数
 - >！`这样的函数必须出现**once**在二进制/ dylib / cdylib的依赖图中。

鉴于`＃！[no_std]`应用程序没有标准输出在一些`＃！[no_std]`应用，例如嵌入式应用程序，需要不同的恐慌行为进行开发和使用释放它可能有助于恐慌`crates`，只包含一个`＃[panic_handler]`。这样，应用程序可以通过简单地链接到不同的恐慌`crate`来轻松交换恐慌行为。

下面显示了一个示例，其中应用程序具有不同的恐慌行为，具体取决于是使用dev配置文件（`cargo build`）编译还是使用发布配置文件（`cargo build --release`）

``` rust
// crate: panic-semihosting -- log panic messages to the host stderr using semihosting

#![no_std]

use core::fmt::{Write, self};
use core::panic::PanicInfo;

struct HStderr {
    // ..
    _0: (),
}

impl HStderr {
    fn new() -> HStderr { HStderr { _0: () } }
}

impl fmt::Write for HStderr {
    fn write_str(&mut self, _: &str) -> fmt::Result { Ok(()) }
}

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    let mut host_stderr = HStderr::new();

    // logs "panicked at '$reason', src/main.rs:27:4" to the host stderr
    writeln!(host_stderr, "{}", info).ok();

    loop {}
}
```

``` rust
// crate: panic-halt -- halt the thread on panic; messages are discarded

#![no_std]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

``` rust
// crate: app

#![no_std]

// dev profile
#[cfg(debug_assertions)]
extern crate panic_semihosting;

// release profile
#[cfg(not(debug_assertions))]
extern crate panic_halt;

// omitted: other `extern crate`s

fn main() {
    // ..
}
```
