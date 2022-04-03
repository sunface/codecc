# 用 Rust 写 DevOps 工具

> 原文链接: https://www.fpcomplete.com/blog/rust-for-devops-tooling/
>
> **翻译：[Xiaobin.Liu](https://github.com/lxbwolf)**
>
> 选题：[Xiaobin.Liu](https://github.com/lxbwolf)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

用 Rust 编写 DevOps 工具的初学者教程。

## 介绍

本篇博文我们会介绍几个 Rust 用在 DevOps 领域的案例，以及为什么使用 Rust。其中我们还会介绍一些在 AWS 上基于 Rust 的 DevOps 工具常用的库。

可能你已经熟悉用其他语言如何编写 DevOps 工具，本文介绍的内容是如何用 Rust 编写。

我们会介绍为什么 Rust 语言是编写 DevOps 工具以及构建软件所需的云基础设施的相当不错的选择。我们也会介绍一个用 Rust 编写的 DevOps 工具小 demo。这个工程旨在帮助不熟悉 Rust 语言生态的开发者了解 Rust 工程结构。

如果你是 Rust 新手且对这门语言感兴趣，那么请阅读我们的 [Rust 速成教程](https://www.fpcomplete.com/rust/crash-course/)。

## Rust 为什么是独一无二的

> Rust 是一种系统编程语言，关注三个方面：安全、速度和并发。它没有垃圾回收机制，这使得它在某些场景下比其他语言更优秀，如嵌入到其他语言中，有特殊空间和时间需求时的编程，编写低级代码如设备驱动和操作系统等。

*Rust 教程（出版）*

Rust 最初是 Mozilla 创造的，目前已经受到了广泛的应用和支持。像 Rust 教程中提到的那样，它的目标是取代 C++或 C 的地位（Rust 也没有垃圾回收和运行时）。Rust 的设计中也包含了零成本抽象和你在其他的高级语言（像 Go 和 Haskell）能见到的一些概念。基于以上原因，外加一些其他因素，Rust 的应用范围已经远远超过普通的低级安全系统语言。

Rust 的所有权系统对于编写正确的和资源高效的代码非常有用。所有权是 Rust 语言的杀手锏，可以帮助程序员在编译时捕获其他语言不能捕获或忽略的各种资源错误。

Rust 是一种高性能和高效率的语言，速度可以媲美 C/C++ 语言。由于 Rust 没有垃圾回收机制，因此可以很容易计算出它的性能。

## Rust 与 DevOps

能使 Rust 变得独特的因素，同时也对从机器人到火箭技术等领域都非常有用，但这些因素适用于 DevOps 吗？在 DevOps 中，我们关心可执行程序的效率吗？我们在意资源的控制权吗？使用 Rust 是不是杀鸡用牛刀？

*是也不是*

当我们对性能要求极高，或者要求结果的确定性和一致性时，Rust 显然很有用。上面的场景意味着需要操作低级空间，而这些操作以前只有 C 和 C++ 才能完成。在 Rust 出现以前，开发者遇到以上场景时不得不面临所用语言的内在风险以及在大型代码库上开发的额外开销。Rust 可以在这些领域表现良好，且不会有 C 和 C++ 语言的风险。

但在 DevOps 和基础设施编程中，我们并不受这些需求的约束。对于 DevOps 而言，我们可以从 Go、Python 或 Haskell 语言中选择，因为我们不需要局限于没有垃圾回收的语言。由于我们可以选择其他的语言，因此可能有人会质疑是否真的需要用 Rust，我们再回顾几个要点来反驳这种观点。

### 你为什么想用 Rust 来编写 DevOps 工具？

- 与其他语言如 Go、Java 相比，更小的可执行文件
- 方便在不同的操作系统间移植
- 资源利用率（缩减 AWS 成本）
- 最快的语言之一（包括与 C 对比）
- 零成本抽象 —— Rust 是一种高性能的低级语言，却拥有高级语言的泛型和抽象

针对上面这几条，细化一下：

#### 不同架构与操作系统间 Rust 的交叉编译

对于 DevOps 而言，我们仍要讨论在不同架构和不同操作系统间移植 Rust 代码的便捷性。

官方的 Rust 工具链安装器 `rustup` 可以很简单地安装与你的系统匹配的标准库。Rust [支持多种平台](https://doc.rust-lang.org/nightly/rustc/platform-support.html)。`rustup` 工具的文档中有[一个章节](https://rust-lang.github.io/rustup/cross-compilation.html)介绍如何获取适合你的架构的预先编译好的工具。你可以运行 `rustup target add` 来安装某个架构的目标平台（而不是默认安装的宿主平台）：

```bash
$ rustup target add x86_64-pc-windows-msvc 
info: downloading component 'rust-std' for 'x86_64-pc-windows-msvc'
info: installing component 'rust-std' for 'x86_64-pc-windows-msvc'
```

交叉编译已经默认被构建进了 Rust 的编译器。当安装完 `x86_64-pc-windows-msvc` 目标后，你可以用 `cargo` 构建工具的 `--target` 参数来构建适合 Windows 的程序。

（默认目标是宿主的架构）

如果你的依赖涉及到原生的库（如非 Rust 的），那么你需要确保交叉编译没有问题。`rustup target add` 只能安装当前目标的 Rust 标准库。当交叉编译时需要获取其他工具时，你可以使用 https://github.com/cross-rs/cross 工具。这个工具本质上是对 cargo 的封装，可以对安装了所有依赖的东西的 docker 镜像进行交叉编译。

#### 体积小的可执行文件

不需要运行时和垃圾回收是 Rust 区别于其他语言的一个关键因素。与 Python 或 Haskell 语言对比：Rust 不依赖任何运行时（对比 Python）、不依赖系统库（对于 Haskell）极大地增强了可移植性。

在生产上 —— 既然我们已经提到了 DevOps —— 移植性即 Rust 构建出的可执行文件比其他脚本更容易部署。相对于 Python 和 Bash，在使用 Rust 时我们不用提前配置代码的环境。这样我们就无需关心是否已经配置了该语言的运行时的依赖。

除此之外，使用 MUSL libc（Rust 默认静态链接所有的 Rust 代码）你可以从 Rust 源码构建出 100% 的静态可执行文件。这意味着，把 Rust DevOps 工具二进制部署到不同的 Linux 服务器上时，你无需担心正确的 `libc` 或其他库是否已经提前安装好了。

创建静态的 Rust 可执行文件很简单。之前介绍过，你可以用 Rust 构建适配不同操作系统的目标。你可以使用下面的命令来构建适配 Linux MUSL 目标的静态可执行文件：

```bash
$ rustup target add x86_64-unknown-linux-musl
```

然后你可以用这个新添加的目标把你的 Rust 项目构建成完全的静态可执行文件：

```bash
$ cargo build --target x86_64-unknown-linux-musl
```

由于没有运行时和垃圾回收器，Rust 的可执行文件非常小。例如，有一个常见的 DevOps 工具，名为 CredStash，最初是用 Python 写的，后来用 Go 编写（GCredStash），现在用 Rust 编写（RuCredStash）。

对比下 Rust 的可执行文件与用 Go 实现的 CredStash，Rust 的可执行文件大小只有 Go 版本的四分之一。

| **Implementation**                            | **Executable Size** |
| --------------------------------------------- | ------------------- |
| Rust CredStash: (RuCredStash Linux amd64)     | 3.3 MB              |
| Go CredStash: (GCredStash Linux amd64 v0.3.5) | 11.7 MB             |

项目链接：

- [github.com/psibi/rucredstash](https://github.com/psibi/rucredstash)
- [github.com/winebarrel/gcredstash](https://github.com/winebarrel/gcredstash)

这不是一个完美的比较，8 MB 看起来不是很多，但是如果考虑到构建后体积会自动缩小到四分之一你可能就会很期待了。

这可以减小你的 docker 镜像、AWS AMI、Azure VM 镜像的体积，也可以加速新的部署过程。

这种大小的工具，可执行文件的体积减少了 75%，如果减少的比例小于 75% 可能看起来并没有那么明显。在这种刻度标准下 8 MB 的效果仍然不是很好。但随着工具（或基于 Rust 的软件与工具的集合）体积的增大，差异也会累积，最后会变成生产级的差异，这个差异可能就会值得考虑了。

Rust 实现的版本也没有完全达到理想的体积。如果很注重体积大小的话，可能需要做一些其他的修改 —— 但那不是本文讨论的范围。

#### Rust 速度很快

即便是每天常用的 Rust 代码，速度也很快。不仅如此，有证据表明 Rust 的错误捕获比 C 和 C++更简单。

在财富基准测试（测试范围包括：ORM、数据库连通、动态大小的集合、排序、服务侧模板、XSS 防攻击对策、字符编码）中，Rust 位于第二和第三位，以 4% 的差值落后于首位的基于 C++ 的框架。

![01.png (2313×329) (raw.githubusercontent.com)](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402 Using Rust for DevOps tooling/01.png)

数据库单次查询的基准测试中，Rust 是第一和第二：

![01.png (2313×329) (raw.githubusercontent.com)](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402 Using Rust for DevOps tooling/02.png)

包含所有内容的基准测试中，基于 Rust 的框架位于第二和第三。

![01.png (2313×329) (raw.githubusercontent.com)](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402 Using Rust for DevOps tooling/03.png)

当然生产环境中不只有语言和框架，但当涉及到其他因素时这个语言间的对比仍然是公平的（带有上下文且关注基准测试本身）。

来源：https://www.techempower.com/benchmarks/

### 你为什么不想用 Rust 写 DevOps 工具呢？

对于大中型项目，拥有类似 Rust 中的类型系统和编译时检查是很重要的，而你在 Python 或 Bash 中也能找到这些。你用 Python 或 Bash 可以很容易地实现功能。这在一定程度上使开发“更快”。

某些场景下，特别是在小型项目，更适合用解释型的语言。在这些项目中，相比于 Rust 带来的好处（安全、执行速度、可移植性），不需要重新编译、重新部署就可以快速地修改代码即时生效这些好处更重要。

### 项目结构

我们使用 [Rusoto](https://www.rusoto.org/) 库来进行 AWS 集成。对于我们的中型 Rust DevOps 工具 demo，我们拉取 [rusoto_core](https://docs.rs/rusoto_core/0.45.0/rusoto_core/) 代码和 [rusoto_s3](https://docs.rs/rusoto_s3/0.45.0/rusoto_s3/) crate（Rust 的 crate 类似库或包）。

对于 CLI 选项，我们使用 [structopt](https://docs.rs/structopt/0.3.16/structopt/)。它包含多个 CLI 库，可以很容易地为 Rust 结构体实现 CLI 接口。

这个工具通过[正则表达式](https://github.com/fpco/rust-aws-devops/blob/54d6cfa4bb7a9a15c2db52976f2b7057431e0c5e/src/main.rs#L211)匹配用户传入的 CLI 选项和参数来进行处理。

我们可以匹配定义的 CLI 选项，调用该选项的方法。

```rust
match opt {
    Opt::Create { bucket: bucket_name } => {
        println!("Attempting to create a bucket called: {}", bucket_name);
        let demo = S3Demo::new(bucket_name);
        create_demo_bucket(&demo);
    },
```

上面代码匹配了 `Opt` 枚举的 [Create](https://github.com/fpco/rust-aws-devops/blob/54d6cfa4bb7a9a15c2db52976f2b7057431e0c5e/src/main.rs#L182) 变量。

然后我们用 `S3Demo::new(bucket_name)` 创建一个 `S3Client`，我们定义的独立的 `create_demo_bucket` 可以创建一个 S3 bucket，然后使用 `S3Client`。

这个工具的代码在 [src/main.rs](https://github.com/fpco/rust-aws-devops/blob/54d6cfa4bb7a9a15c2db52976f2b7057431e0c5e/src/main.rs)，使用起来很简单。

### 构建 Rust 项目

在这个项目中构建代码之前，你需要先安装好 Rust。安装过程请遵照[官方的安装教程](https://www.rust-lang.org/tools/install)。

Rust 的默认构建工具是 Cargo。你可以参照 [Cargo 文档](https://doc.rust-lang.org/cargo/guide/)，我们这里也提供了一个构建项目的快速指南。

在 [git 仓库](https://github.com/fpco/rust-aws-devops)的根目录执行下面的命令来构建项目：

```bash
cargo build
```

你可以用 `cargo run` 来运行代码，也可以用 `./target/debug/rust-aws-devops` 来执行直接执行：

```bash
$ ./target/debug/rust-aws-devops 

Running tool
RustAWSDevops 0.1.0
Mike McGirr <mike@fpcomplete.com>

USAGE:
    rust-aws-devops <SUBCOMMAND>

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

SUBCOMMANDS:
    add-object       Add the specified file to the bucket
    create           Create a new bucket with the given name
    delete           Try to delete the bucket with the given name
    delete-object    Remove the specified object from the bucket
    help             Prints this message or the help of the given subcommand(s)
    list             Try to find the bucket with the given name and list its objects``
```

该命令可以输出由我们的 `structopt` 自动创建的 CLI 帮助信息。

执行下面的命令来构建 release 版本（需要开启优化，编译过程可能会有点慢）：

```bash
cargo build --release
```

## 总结

从这个小 demo 可以看出，用 Rust 编写 DevOps 工具并不难。我们并不需要在部署的难易程度和代码的性能之间进行权衡。

希望在下次编写 DevOps 软件、用于特定 DevOps 操作的简单 CLI 工具或 Kubernetes 时，你能考虑下 Rust。如果你有关于 Rust 的其他问题，或在实现 Rust 项目时需要帮助，请查阅 FP Complete 的 Rust 工程和教程！

想学习更多关于 Rust 的知识？请查阅 [Rust 速成教程](https://www.fpcomplete.com/rust/crash-course/)，也可以浏览我们的 [Rust 首页](https://www.fpcomplete.com/rust/)。