# Rust编程语言

> **🎉🎉欢迎参与🎉🎉**

这是[**中文版Rust标准库**](https://kayrx.xyz/rust/std/) 的主要源代码存储库。它包含编译器，标准库和中文文档。**但是它和官方存储库是一致的!!**

[Rust]: https://www.rust-lang.org

## 目标

- 一致性

  所有操作保持结构格式与官方一致

- 准确性

  直接进行Git Pull官方源码,直接翻译源码文档，共同参与监督

- 及时性

  按版本与官方保持同步, 同步更新
  

## 实施

**第 1 步**

```bash
git clone https://github.com/kriry/rust.git
```

**第 2 步**

中文翻译 **标准库** 源码文档部分(有能力可以翻译其他部分^_^)

```bash
cd src/libstd

cargo doc
```
生成文档查看校验中文翻译效果

**第 3 步**

提交PR，服务中文用户，并获取美誉！

## Trademark

The Rust programming language is an open source, community project governed
by a core team. It is also sponsored by the Mozilla Foundation (“Mozilla”),
which owns and protects the Rust and Cargo trademarks and logos
(the “Rust Trademarks”).

If you want to use these names or brands, please read the [media guide][media-guide].

Third-party logos may be subject to third-party copyrights and trademarks. See
[Licenses][policies-licenses] for details.

[media-guide]: https://www.rust-lang.org/policies/media-guide
[policies-licenses]: https://www.rust-lang.org/policies/licenses
