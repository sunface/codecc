# [2022-06-01]Effective Rust

# **[1. 介绍](https://www.lurklurk.org/effective-rust/intro.html#introduction)**



Scott Meyers 的原著 *Effective C++* 一书因为引入了一种新的编程风格而非常成功，其重点介绍了在使用 *C++* 开发软件的实际经验中，探索出来的一系列开发准则。重要的是，这些开发准则，明确解释了为什么它们是必要的，这允许读者根据自己的特殊场景来判断，是否要打破这些规则。

第一版 *Effective C++* 出版于 1992年，在那个时候，C++ 尽管还很年轻，但已经是一门拥有很多让开发者头晕的特性的一门语言了；有一个针对这些不同特性的交互指南是至关重要的。

Rust 同样是一门年轻的语言，但与 C++ 相比出乎意料的是， Rust 将这些难缠的特性中开发中解脱出来了。类型系统的强大性和一致性意味着，如果一个 Rust 程序能编译成功，其代码质量就已经相当不错了，这种现象以前只有在诸如 Haskell 这样的学术性较强、较难理解的语言中才能观察到。

同时涵盖类型安全和内存安全会造成一些开销。Rust 享有陡峭学习曲线的名声，这对新人来说，则意味着必须要经历与借用检查器斗争，然后重新设计自己的数据结构，被生命周期绕晕的成长仪式。Rust 编译器有着非常严格错误诊断，顺利通过编译的 Rust 程序有大概率能正常且合理地运行起来，但这也意味着会不可避免地要和编译器做斗争，被其折磨。

因此，这本书的目标与其他以 *Effective<编程语言>* 命名的书略有不同；本文会有更多的篇幅来讲解 Rust 中全新的概念，尽管官方文件已经包含了这些话题的良好介绍。这些话题有像 “*Understand*...” 和 “*Familiarize yourself with*...” 类似的格式。

Rust 的安全性也导致了在 Rust 编程世界中完全不会有以 “决不...” 开头的文章。如果您真的不应该做某事，编译器通常会阻止您做。

本文假设读者已经对 Rust 的基本知识有所了解。使用的是 2018 版本的 Rust， 工具链使用的稳定（stable）版本。

明确用于代码片段和错误消息的 `rustc` 版本是 1.49。Rust 现在已经足够稳定（并且有足够的后台兼容性保证)），以至于代码片段不太可能需要更改以后的版本，但是错误消息可能会随着特定的编译器版本而变化。

文本中还有一些对 C++ 的引用和比较，因为这可能是最接近的等价语言（特别是 C++ 11 的  move  语义），也是 Rust 的新用户最有可能遇到的以前的语言。

本书分为六个部分：

- 类型： 一些围绕着 Rust 核心类型系统的建议。
- 概念：关于 Rust 语言设计中的核心理念。
- 依赖: 一些与 Rust 包生态打交道的建议。
- 工具: 如何使用 Rust 编译器来改进代码库质量的建议。
- **异步 Rust**: 关于 与 Rust `async` 机制打交道的建议。
- **超标准 Rust**: 当你必须在 Rust 超标准、安全环境开发的建议





虽然 “概念” 章节可以说比 “类型” 章节更基础，但它被故意放在第二篇，以便从头到尾阅读的读者能够首先建立起一些信心。

下面的标记是从 *[Rust Book](https://doc.rust-lang.org/stable/book/ch00-00-introduction.html#ferris)* 中借来的，用于识别在某些方面不正确的代码





| Ferris | **Meaning** |
| :------------ |:---------------:|
| <img src="https://www.lurklurk.org/effective-rust/third_party/ferris/does_not_compile.svg" alt="img" style="zoom:10%;" /> |       这段代码不能通过编译！       |
| <img src="https://www.lurklurk.org/effective-rust/third_party/ferris/panics.svg" alt="img" style="zoom:10%;" /> | 这段代码会 panics! |
| <img src="https://www.lurklurk.org/effective-rust/third_party/ferris/unsafe.svg" alt="img" style="zoom:10%;" /> | 这段代码块包含不安全（unsafe）代码 |
| <img src="https://www.lurklurk.org/effective-rust/third_party/ferris/not_desired_behaviour.svg" alt="img" style="zoom:10%;" /> | 这段代码不会按照期望执行 |