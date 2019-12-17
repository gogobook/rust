# 原子性

> 原文跟踪[atomics.md](https://github.com/rust-lang-nursery/nomicon/blob/master/src/atomics.md) &emsp; Commit: 6dd445b8e72bcbc502cf28240830be52d2a8240d

在内存原子性(atomics)模型上, rust 就公然的直接继承了 C11 的模型, 这不是因为这个模型多好或者多容易懂, 相反这个模型挺复杂的, 还有若干已知[缺陷](http://plv.mpi-sws.org/c11comp/popl15.pdf). 但事实上不是每个人都善于去给内存原子性建模, 所以似乎这样也不错的. 至少我们可以从 C 现有的工具和研究中学到点什么.

本文不会尝试完全充分的解释这个模型. 它是用令人发狂的因果关系图定义的, 需要用一整本书去尽可能的解释的清楚. 想知道更多细节可以看下 [C 语言的规范(第 7.17 章节部分)](http://www.open-std.org/jtc1/sc22/wg14/www/standards.html#9899). 不过本文会尝试把基本的概念和一些 Rust 开发者碰到的一些问题都覆盖到.

C11 内存模型本质上来说, 是为了在桥接对人来说的语义性, 对编译器来说优化性, 对硬件来说的非一致混乱性. 简单说就是希望我们写的程序, 按照我们的想法跑的又快又好.




# Compiler Reordering 编译器重新排序

编译器本质上希望能够通过进行一些复杂的转换和减少数据依赖, 来消除死代码(dead code). 特别是, 它可以从根本上改变代码的运行的顺序, 或者让一些代码的逻辑永远不再出现. 举个例子, 如果有如下代码:

```rust
x = 1;
y = 3;
x = 2;
```

编译器可能推断出, 改成如下可能更好:

```rust
x = 2;
y = 3;
```

这样的顺序倒转, 就消除了第一条赋值事件. 从单线程的角度, 这样的变化是完全无法察觉的: 所有语句执行完之后的状态完全相同. 但是如果程序是多线程的, 第一句赋值在 y 分配之前的逻辑, 就完全可能真的是有逻辑依赖的. 我们希望编译器有能力进行这样类型的优化, 毕竟这样做可以大幅提升性能. 另一方面, 我们也希望程序能够完全按照*我们的意愿*执行.




# Hardware Reordering 硬件重新排序

话说就算编译器完全按照我们的意愿去执行理解我们的代码, 还有硬件也会出来给你使绊子. 主要原因是多级高速缓存的结构对 CPU 的影响. 全局共享内存对 cpu 来说*又远又慢*. 每个 cpu 核心宁可使用本地数据缓存, 只有当本地缓存没数据时, 才会去找全局共享内存找.

所以这个问题主要是在 cpu 的多级缓存上, 如果每次读取缓存都要去共享内存立检查下数据有没有改变, 那缓存就没存在意义了. 结果就是硬件层面上, 不同线程不能保证程序逻辑相同. 为此, 我们必须通过发出特殊的 cpu 指令, 让他不要这么聪明.

举个例子, 我们保证编译器编译结果逻辑如下:

```text
initial state: x = 0, y = 1

THREAD 1        THREAD2
y = 3;          if x == 1 {
x = 1;              y *= 2;
                }
```

理想情况下这段程序有这样 2 个分支:

* `y = 3`: (线程 2 先于线程 1 执行)
* `y = 6`: (线程 2 后于线程 1 执行)

但是还有第 3 个潜在的分支:

* `y = 2`: (线程 2 取到 x 时 `x = 1`, 但此时 y 还没有被赋值为 3)

注意不同类型的 CPU 提供不同的保证, 通常将硬件分为两类: 强排序(strongly-ordered)和弱排序(weakly-ordered).
X86/64 架构提供强排序保证, ARM 架构提供弱排序排序保证.
这对并发编程产生了两个影响:

* 在强排序保证机器上要求强排序保证, 代价极低, 低到可以忽略不计, 因为硬件提供了无条件的强排序保证. 弱排序保证可能在弱排序机器上有性能优势.

* 即使程序确实有问题, 多数情况下, 是在强排序保证机器上, 要求极弱顺序的保证是有可能正常工作的. 如果可能, 应该在弱排序保证的机器上测试并发算法的性能



# Data Accesses 数据访问

C11 内存模型试图通过让我们讨论程序*因果关系*, 来拉近我们和程序底层的逻辑. 通常, 这是通过在程序的各个部分与运行它们的线程之间建立*之前的*关系. 这样就给了硬件和编译器优化的空间, 可以更加激进的优化"发生前"关系建立之前, 但是强制它们在关系建立后更加谨慎. 我们通过 *数据访问(data acesses)* 和 *原子性访问(atomic accesses)* 来和这些关系联系起来.
> The C11 memory model attempts to bridge the gap by allowing us to talk about the
*causality* of our program. Generally, this is by establishing a *happens before* relationship between parts of the program and the threads that are running them. This gives the hardware and compiler room to optimize the program
more aggressively where a strict happens-before relationship isn't established,
but forces them to be more careful where one is established. The way we
communicate these relationships are through *data accesses* and *atomicaccesses*.


数据访问是编程世界的面包和黄油. 它们本质上是非同步的, 编译器可以自由激进的优化它们. 尤其是当程序是单线程的, 编译器可以自由的重新排序所有数据访问. 硬件层面上也可以将数据的改动惰性的非一致的随意传播. 关键是, 数据访问会产生数据竞争. 数据访问对硬件和编译非常友好, 但是在代码语义上, 数据访问提供了极为*弱鸡*的语法, 事实上简直弱爆了.

**仅用数据访问来写正确同步代码是不可能的**

使用原子性的数据访问(atomic accesses)正是我们在告诉编译器和硬件, 我们现在写的程序是多线程的. 每个原子访问可以用*顺序*标记, 以指定它与其他访问建立的关系类型. 实际上, 这其实就是在告诉编译器和硬件, 哪些优化你*别做*, 哪些优化你*不能做*. 对编译器来说, 主要是内容就是围绕指令的重新排序(re-ordering). 对硬件来说, 主要内容就是围绕数据传播到其他线程的写入方式. Rust 暴露这几种:

* Sequentially Consistent (SeqCst)
* Release
* Acquire
* Relaxed

(Note: Rust 明确不暴露 C11 的 *consume* 顺序)

TODO: negative reasoning vs positive reasoning? TODO: "can't forget to
synchronize"


# Sequentially Consistent 顺序一致性
顺序一致性是最强大的, 意味着限制也是最大的. 直观的说, 顺序一致的操作是不能被重新排序的: 一个线程上的所有 SeqCst 访问必须按照访问的先后顺序进行访问. 如果一个无数据竞争的程序来只用顺序一致的原子性访问和数据访问, 那其实是有好处的, 所有线程有了一个全局唯一的执行空间. 本质上来说这个全局唯一的执行空间是是所有线程交错独立的去进入并执行的. 但前提是没有使用更弱的原子访问顺序

这个对开发人员友好的顺序一致的特性, 并不是没有代价的. 即使在强顺序一致(strongly-ordered)的平台上, 就涉及到了通过使用内存栅栏(memory fences)而实现顺序一致的访问特性.

实际实践里, 在一个正确的程序中, 顺序一致的访问只在很少的情况是必须的. 但是如果你不确定要怎么选内存读写顺序, 顺序一致的数据访问肯定是最正确的选择. 让程序逻辑正确但是跑的慢一点, 总比程序有 bug 要好的多. 就算只后需要对原子访问顺序降级, 也不是很麻烦的. 只要把 `SeqCst` 改成 `Relaxed` 就完成了! 当然, 证明这个改动是*正确*的改动, 那就是另外一回事儿了.


# Acquire-Release 获取-释放顺序
获取和释放的内存访问顺序, 大部分情况下是设计成成对使用的. 就和它的名字一样, 它非常适合获取和释放锁, 并确保关键的中间那部分不重叠.

直观的说, 获取访问(acquire access)保证每次访问都在获取(acquire)之后. 但是在获取之前的操作可以在其之后自由的重新排序.
同样的, 释放访问(release access)保证都在释放(release)之前, 但是在释放之后的操作可以在其之前自由的重新排序.

当线程 A 释放了某个位置的内存访问控制, 随后线程 B 获取了内存中*相同*位置的访问控制, 因果关系就建立起来了. 在 A 每次释放前的写操作, 都可以在 B 获取访问控制后观察到. 但是其他的线程访问不会建立这样的因果关系. 同样, 其他*不同*内存位置的访问也不会建立这样的因果关系.

获取-释放的访问顺序控制的基础使用很简单: 获取某个内存位置后, 开始中间关键代码逻辑(避免数据争用)部分, 然后释放这个位置的访问控制就结束了.
举个例子, 一个简单的自旋锁(spinlock)看起来就像这样:

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, Ordering};
use std::thread;

fn main() {
    let lock = Arc::new(AtomicBool::new(false)); // lock 值代表访问是否被锁定

    // ... 进行一些给不同线程分配锁的操作 ...

    // 尝试获取锁并赋值为 true
    while lock.compare_and_swap(false, true, Ordering::Acquire) { }
    // 结束循环我们就成功的获取了锁

    // ... 进行谨慎的数据访问操作 ...

    // 到这里操作结束了就释放锁
    lock.store(false, Ordering::Release);
}
```

在强顺序的平台上, 多数访问都具有释放和获取的指令, 意味着释放获取常常几乎没有性能损耗. 但是在弱顺序平台上就不是这样得了.


# Relaxed

Relaxed 访问绝对是最弱的访问顺序控制. 它们可以自由的重新排序, 不提供线性事件的发生关系. 尽管这样 Relaxed 操作仍是原子性的. 就是说, 它不能算是数据访问控制, 但是在读写操作时, 是以原子性的方式进行的. Relaxed 方式的操作适合在你明确希望某个操作一定会"发生", 但是其他就不管的场景下. 举个例子, 在只有异步的情形下, 多个线程对一个计数器使用 Relaxed 的 `fetch_add` 操作是安全的.

在强顺序一致性的机器上用 Relaxed 操作很少会有太大的收益, 因为强顺序一致的机器本身就提供了获取-释放的语义支持. 但是 Relaxed 操作在弱顺序一致性的机器上会有较高的性能收益.



[C11-busted]: http://plv.mpi-sws.org/c11comp/popl15.pdf
[C11-model]: http://www.open-std.org/jtc1/sc22/wg14/www/standards.html#9899
