# 点运算符

> 原文跟踪[dot-operator.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/dot-operator.md) &emsp; Commit: 0e6c680ebd72f1860e46b2bd40e2a387ad8084ad

点运算符将执行很多魔术来转换类型。 它会表现自动引用，自动解除引用和强制，直到类型匹配。

TODO: 信息来源 http://stackoverflow.com/questions/28519997/what-are-rusts-exact-auto-dereferencing-rules/28552082#28552082
