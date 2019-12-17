# 标准库之下

> 原文跟踪[beneath-std.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/beneath-std.md) &emsp; Commit: a0c1de174abda5bc5655ed27b0c5741a1a48a92e

本节介绍（或将记录）标准库提供的功能那个`＃！[no_std]`开发人员必须处理（即提供）构建`＃！[no_std]`二进制包。 一个
（可能不完整）此类功能列表如下所示：

- #[lang = "eh_personality"]
- #[lang = "start"]
- #[lang = "termination"]
- #[panic_implementation]
