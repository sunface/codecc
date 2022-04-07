> 原文链接: https://gankra.github.io/blah/fix-rust-pointers/
>
> **翻译：[BK0717](https://github.com/hyuuko)**
>
> 选题：[BK0717](https://github.com/hyuuko)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

# Rust 的 unsafe 指针类型需要大修

我经常思考 Rust 中的 unsafe 指针问题。

我逐字写完了 [The Rustonomicon][nomicon] 这本书，还有这本 [Learn Rust With Entirely Too Many Linked Lists][too-many-lists]，并且 [重新设计了 Rust 的指针 APIs][1966-unsafe-pointer-reform]，还设计了标准库的 [对 unsafe 堆分配的缓冲区的抽象](https://github.com/rust-lang/rust/pull/26955)，以及维护了一个 [可替代的 Vec 实现](https://github.com/Gankra/thin-vec/)。

[nomicon]: https://doc.rust-lang.org/nightly/nomicon/
[too-many-lists]: https://rust-unofficial.github.io/too-many-lists/
[1966-unsafe-pointer-reform]: https://github.com/rust-lang/rfcs/blob/master/text/1966-unsafe-pointer-reform.md

我经常思考 Rust 中的 unsafe 指针，而且我十分讨厌它们。

不要误会我的意思，我认为以上所有的工作都让它们变得更好，但它们仍然有很大的缺陷。事实上，它们已经变得更糟糕了。不是因为 API 发生了变化，而是因为当我在做这些事情的时候，我们对指针应该如何工作的理解太天真了。其他人已经做了很多杰出的工作来扩展这种理解，而现在这些缺陷更加明显了。

这篇文章分为 3 个部分：概念背景、当前的设计存在的问题，以及推荐的解决方案。

## 1 背景

如果你了解计算机的一切，可以跳过此部分。

### 1.1 别名

别名是编译器和语言语义学中一个非常重要的概念。在高层次上，它是对内存修改的*可观测性*的研究。我们之所以称它为*别名*，是因为你能以多种方式引用一块内存。指针只是某段内存的昵称。

别名的主要功能是作为一个模型，让编译器可以在语义上对内存访问进行缓存。这可能意味着假设内存中的一个值没有被修改，或者假设不需要写到内存中。这一点特别重要，因为*本质上所有的程序状态都在语义上处于内存中*。对于一个代表程序员做任何事情的通用编程语言，我们不可能允许它对内存进行任意的读和写。

这里有一个我们都能同意的极其明显的例子，一个编译器应该能够假定以下程序会把 `1` 传给 `println!`：

```rust
let mut x = 0;
let mut y = 1;

// 如果这里会使 y 被修改，岂不是很操蛋？
x = 2;

println!("{}", y);
```

当我们谈到别名时，我们通常会立即想到指针，因为这是最难的一部分。变量在语义上是没有别名的，直到你真正对其进行引用为止！

这实际上是把东西放在寄存器里的一个基本假设，因为把东西放在寄存器里就是在缓存它。如果一个编译器不能决定把值放在通用寄存器里或者是放在栈上，那么它充其量只是一个汇编器。我们希望造出比汇编器层级更高的语言！

在阐述完“无论如何你都需要这个”后，让我们来谈谈指针是如何使这个问题变得困难的。在下面的函数中，我们可以假设 `input` 和 `output` 是指内存的不同区域吗？

```rust
fn compute(input: &u32, output: &mut u32) {
    if *input > 10 {
        *output = 1;
    }
    if *input > 5 {
        *output *= 2;
    }
}
```

如果我们可以这么假设，那么编译器就可以自由地将其重写如下：

```rust
fn compute(input: &u32, output: &mut u32) {
    // 把 `*input` 放进一个寄存器
    let cached_input = *input;
    if cached_input > 10 {
        // 如果 `*input` 的值不会发生改变，> 10 就意味着：
        //
        // *output = 1
        // *output *= 2
        //
        // 我们可以将其简化为：
        *output = 2;
    } else if cached_input > 5 {
        *output *= 2;
    }
}
```

如果它们*确实*指向重叠的内存，那么写操作 `*output = 1` 会影响读操作 `*input > 5` 的结果，我们把这些称为*别名*访问。当我们执行（潜在的）别名访问时，编译器必须保守地从内存中加载和存储，正如源码所暗示的那样。

现在，谈论*访问*别名通常是很笨拙的，所以我们通常简记为*指针*别名。所以我们可以合理地说，`input` 和 `output` 彼此是别名。这个*实际的*模型之所以是用术语*访问*而不是*指针*，是因为*访问*就是我们关心的事情。

实际上在某些情况下我们并不关心你是否传入了两个“别名”的指针：

- 只有其中一个被使用（没有第二个观测者）
- 两个都只读（两个读显然不会互相影响）（这个假设就是为什么内存映射的硬件必须使用 `volatile`）

这也是为什么 Rust 在“唯一可变”引用（`&mut`）和“共享不可变”引用（`&`）之间有如此明显的区分。对于只读指针，你想复制多少就复制多少，但如果你想真正写到内存，那么知道它是如何别名的就非常重要了！

（你可能会注意到这是一个充满谎言的简化的模型，如果你不想要谎言，请阅读我[关于 Stacked Borrows 的极其详细的讨论](https://rust-unofficial.github.io/too-many-lists/fifth-stacked-borrows.html)。）

下面是一些其他的有用的总结：

- 如果程序员不能用名字或指针来引用某块内存，那么这块内存就是**匿名的**。
- 如果某块内存目前只存在一个引用，那么这块内存就是**无别名的**。

匿名的内存在某种意义上是“完全在编译器的控制之下”，因此可以自由地假定它是无别名的，可以被编译器信任/修改。无别名的内存不会被看似“不相关”的东西“随机”修改（我们将在下一节中讨论这意味着什么）。

编程语言可以有*更严格*或*更弱*的别名模型。更严格的模型允许编译器做更多的优化，但对程序员在语言范围内允许做的事情施加了更多的限制。下面是一些常见的规则，严格程度依次递增：

- 已入栈的由调用者保存的值是匿名的（函数的返回地址、栈帧指针）。
- 由编译器分配到栈的“Scratch”值是匿名的。
- 一个新声明的变量是无别名的，直到存在对它的引用。
- 由 `malloc` 返回的内存是无别名的。
- 一个结构体内的字段不会相互别名（位域除外）。
- 用于填充的字节（Padding bytes）是模糊的匿名的（由于 memcpy/memet/unions/punning 的原因，很混乱）。
- 不可变的变量是无别名的，因为它们永远不会改变值。
- 在 Rust 中，`&mut` 是无别名的（[Stacked Borrows](https://plv.mpi-sws.org/rustbelt/stacked-borrows/)）。
- 在 C/C++ 中，如果 `T!=U` 或者不是 `char`，那么 `T*` 和 `U*` 是无别名的（[Strict Aliasing](https://blog.regehr.org/archives/1307)）。

（我必须强调这一切是多么的简明扼要，魔鬼在细节中，正式规定这些东西是无数博士论文中的主题。我现在不是要写一篇博士论文。除非你真的在 C/C++ 标准委员会工作或你是 Ralf Jung，否则我是不会接受你的关于这些定义和术语的见解的。）

### 1.2 别名分析和指针来源（Pointer Provenance）

现在你有了一些关于内存如何被认为是“无别名”的定义，但是只要你拿一个指针到某个东西上，或者复制一个指针，或者偏移一个指针......这一切都会消失，对吗？就像你必须假设任何东西都可以被其他东西别名？

不！别名规则是语言语义学和优化的一些最基础的部分。如果你违反了语言的别名规则，你就会有未定义的行为，而且错误的编译可能是非常残酷的。

但是，是的，一旦你开始玩起了指针，对于编译器、内存模型和程序员来说，事情就会变得*更加困难*。一旦指针开始被扔来扔去，为了使别名成为一个有用的概念，内存模型很快发现需要定义两个概念：

- 分配（Allocations）
- 指针来源（Pointer Provenance）

*分配*抽象地描述了像单个变量和堆分配这样的东西。一个新创建的分配（变量声明，malloc）被带入这个世界时总是无别名的，因此就像一个有唯一的真名的*沙盒*——*除了*通过唯一的真名，没有办法访问沙盒中的内存（这不是未定义行为）。

访问分配的沙盒的权限可以从唯一真名*授予*，授予方式是从唯一真名派生出一个新的指针（或递归地从它派生出的任何东西）。从唯一真名到所有派生指针的“监管链”的跟踪过程就是*指针来源*。

从形式化内存模型的角度来看，对一个分配的所有访问必须有追溯到该分配的*来源*。如果不知道指针的来源，那就意味着程序员闯出了沙盒，或者从太虚中取得了一个指针，恰好指向了某个随机的沙盒。无论哪种方式，如果允许的话，一切都会变得混乱，变得没有任何意义。

从编译器优化的角度来看，跟踪来源允许编译器证明两个访问不存在别名。如果两个指针已知有不同的出处，那么它们就不可能别名，你可以得到 Good Codegen。如果它曾经失去了对某个内存的指针的跟踪（即如果指针被传递给一个不透明的函数），那么它就把这个内存/指针放在了一个“可能是别名”的桶里。对两个“可能是别名”的访问必须保守地被假定是别名，并可能得到 Bad Codegen。

这是编译器应用于所有不可能的问题的基本技巧：通过一个简单的分析，可以用“是”、“不是”或“也许是”来回答你的询问，然后根据哪一个更安全，将“也许是”转换成“是”或“不是”。这两个访问“也许是”别名吗？那么“是”的，它们是别名。问题解决了。

在内存安全的语言中，这一切“只是”一种优化方案，因为程序员不能“破坏”编译器的分析。但是一旦你做了不安全的事情（比如用 C 或 Unsafe Rust），编译器就需要你来帮助它，并真正地遵守一些该死的规则。具体来说，每个人都认为你*真的*不应该被允许突破分配沙盒。

这就是为什么 LLVM 的 [GetElementPointer (GEP)](https://llvm.org/docs/GetElementPtr.html) 指令（用来计算指针偏移量的）几乎总是被编译器用 `inbounds` 关键字发出。`inbounds` 关键字基本上是“我保证这个偏移量不会使指针脱离它的分配沙盒，并完全破坏别名和来源”。这就像，你所有的指针偏移量都应该遵循这个规则！

让我们再上一层楼，看看 rustc：任何时候你做 `(*ptr).my_field`，编译器都会发出 `GEP inbounds`。你有没有想过，为什么 [ptr::offset](https://doc.rust-lang.org/std/primitive.pointer.html#method.offset) 之类的文档是如此的奇怪和复杂？因为它们和 `GEP inbounds` 相关，需要遵循它的规则！[ptr::wrapping_offset](https://doc.rust-lang.org/std/primitive.pointer.html#method.wrapping_offset) 只是 `GEP`，没有 `inbounds`。而且 `wrapping_offset` 实际上也不允许破坏来源：

> 与 `offset` 方法相比，这个方法基本上推迟了停留在同一分配对象内的要求：`offset` 方法在指针跨越对象边界时，直接就是未定义行为；`wrapping_offset` 方法产生一个指针，但如果指针在它所连接的对象的界外，它被解引用时仍然会导致未定义行为。

### 1.3 CHERI

我的错，我好几年都将 [CHERI](https://www.cl.cam.ac.uk/research/security/ctsrd/cheri/) 称为不会交付的雾件（译者注：Vaporware，指在开发完成前就开始宣传的、雷声大雨点小、也许根本就不会问世的产品）！我要吃掉自己的帽子，因为 [ARM Morello 确实建造并交付了一个完整的基于 CHERI 的 CPU](https://www.arm.com/architecture/cpu/morello)。祝贺所有为之工作的人！

那么什么是 CHERI？我不打算讨论这些细枝末节，但大致上来说，它是一个 128 位架构。嗯，实际上是 129 位。嗯，实际上是 64 位。_对不起，什么？_

好吧，CHERI 的整个想法是，它实际上重新定义并实现了上一节中的“沙盒”模型。每个指针都被标记了额外的元数据，由硬件维护和验证。如果你突破了你的沙盒，硬件会抓住它，操作系统可能会杀死你的进程。

我不知道编码或元数据的全部细节，但我们关心的部分是，每个指针都包含一个允许它指向的压缩的*切片*（内存范围），以及它所指向的*实际*地址。这个切片是这个指针的沙盒，所有从它派生出来的指针都继承了这个沙盒（或这个沙盒的一部分）。每当你访问一些内存时，硬件就会检查该指针是否仍在其沙盒内。

这个元数据并不便宜：CHERI 中的指针是 128 位宽的，但有效的地址空间仍然最多是 64 位（我不知道确切的上限，重要的是地址适合 64 位）。现在 128 位*真的*很臃肿，所以在 C/C++ 中 CHERI 实际上得到了我们的老克星 [The Wobbly C Interger Hierarchy](https://gankra.github.io/blah/rust-layouts-and-abis/#the-c-integer-hierarchy) 的帮助。

C 对 `intptr_t`（“指针大小的整数”）和 `ptrdiff_t`/`size_t`（“偏移大小的整数”）进行了区分。在 CHERI 下，`intptr_t` 是 128 位，`ptrdiff_t`/`size_t` 是 64 位。它可以这样做，因为地址空间仍然只有 64 位，所以任何指代偏移量或大小的东西仍然可以是 64 位。

好了，你现在可能有两个迫切的问题：如果我可以在一个指针上涂鸦并破坏它的元数据，这到底怎么可能工作，以及为什么你说它实际上是 129 位。事实证明，这些都是同一个问题！

我发现将其概念化的最好方法是把它想成 [ECC（错误校正代码）RAM](https://en.wikipedia.org/wiki/ECC_memory)。在 ECC RAM 中，每根内存条实际上都有比它声称的更多的物理内存，因为它透明地使用这些额外的内存来纠正或检测随机比特翻转。因此，在*某个地方*有所有这些额外的内存，但就编译器或程序员而言，内存看起来非常正常，没有任何奇怪的额外位。

CHERI 做了同样的事情，但是硬件向你隐藏的额外的第 129 位是一个“元数据有效”位。你看，为了在 CHERI 中正确地操作指针，你需要用特定的指令访问包含指针的内存/寄存器，以完成该任务。如果你试图以其他方式操作它们（例如通过在它上面 memcpying 随机字节），硬件将禁用“元数据有效”位。然后，如果你试图把它*作为*一个指针使用，硬件就会发现你的元数据不可信，并且故障/杀死你的进程。

这真是太神奇了！

（我们在将 Rust 与 CHERI 集成时看到的很多问题实际上很像*分段式架构*的问题，但我从来没有使用过这些，所以我只是模糊地比划了一下，然后摆摆手。请记住，每当我提到 CHERI 的时候，类似的论点*也*可能适用于分段式。所以，如果你关心分段式架构，你也可能关心 CHERI！）

## 2 问题

现在让我们看看 Rust 当前的 unsafe 指针 API 是如何因我们上面看到的所有背景而造成问题的。

### 2.1 整数到指针的转换是魔鬼

Rust 目前说下面这段代码是完全很酷、很好的。

```rust
// 将包装进指针的标签屏蔽掉
let mut addr = my_ptr as usize;
addr = addr & !0x1;
let new_ptr = addr as *mut T;
*new_ptr += 10;
```

这是一些非常普通的代码，用于处理带标签的指针，这有什么不对吗？

想想我们刚才讨论的背景。想想指针来源（Pointer Provenance）。想想 CHERI。

要想让它能够与指针来源和别名分析一起正常工作，那些东西必须无孔不入地感染所有整数，假设它们*可能*是指针。这对那些试图[正式定义 Rust 内存模型](https://plv.mpi-sws.org/rustbelt/stacked-borrows/)的人来说是个巨大的麻烦，对那些[试图为 Rust 建立能够捕捉未定义行为的 sanitizers（用于检测程序缺陷的工具集）](https://github.com/rust-lang/miri)的人来说也是如此。我向你保证，这[对所有 LLVM 和 C/C++ 的人来说也是一个头痛的问题](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2676.pdf)。

为了使其能够在 CHERI 中正常工作，我们需要将 usize 设为 128 位（尽管地址空间是 64 位），并且总是用“指针指令”来操作它，假设它*可能*是一个 `intptr_t` 的指针。是的，人们已经尝试在 CHERI 下运行 Rust，[这正是他们必须做的](https://nw0.github.io/cheri-rust.pdf)。结果，并不怎么好。

对于 CHERI 来说，不幸的是，Rust 实际上将 `usize` *定义*为与指针相同的大小，尽管它的主要作用是作为一个偏移量/大小。对于主流平台来说，这是一个*非常*合理的假设，但它却触犯了 CHERI（以及分段式架构）！

如果你*不把* usize 变成 128 位，而只是试图让它成为 64 位的“地址”部分，那么 `usize as *mut T` 就是一个完全不连贯的操作。将一个整数提升为指针（或者 CHERI 所说的*权能*）需要向它添加元数据。什么元数据？这个随机地址可能在什么范围内有效？从字面上看，没有办法回答这个问题！

现在你可能会想“好吧，但指针标签是一个超级基本的东西，你是说我们不能再这样做了吗？”。不！你完全仍然可以使用标签的技巧，但你需要更小心一些。这就是为什么 CHERI 实际上[引入了一个特殊的操作](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-947.pdf#page=28)。

```cpp
void* cheri_address_set(void* capability, vaddr_t address)
```

这将一个有效的指针（权能）和一个地址作为参数，并创建一个具有相同元数据但具有新地址的新指针。`vaddr_t` 是 CHERI 引入的一个新的整数类型，等同于其他“地址空间大小”的指针，如 `size_t`、`ptrdiff_t` 等。

嘿！这个操作看起来对来源*也*非常有用，不是吗？通过将我们的整数转指针的操作与一个现有的指针联系起来，我们*重新建立*了来源监管链：新的指针是从旧的指针派生出来的，编译器和内存模型可以高兴了！HMMMM...

### 2.2 引用做出了极强的断言

在 Rust 1.0 的旧时代，我们对裸指针的快速和随意性相当乐观。好吧，以大多数人的标准来看，我们实际上对指针是非常严格的。我们在很大程度上执行了 GEP inbounds 语义，在任何地方都要求对齐，[制定了如何处理 ZST 的方法，确保分配不能大于 `isize::MAX`](https://doc.rust-lang.org/nightly/nomicon/vec/vec-alloc.html)，等等。

但是我们在*别名*和*有效性*方面做得非常快和宽松。在旧 Rust 的概念中，引用只是一种*便利*。是的，他们断言了许多事情，但不是以一种*编译器优化*的方式。只是以一种“这个 API 保证这个”的方式。我们都模糊地知道我们*想要*优化的东西，但没有人花精力去解决这个问题。

如今，在[unsafe 代码指南](https://github.com/rust-lang/unsafe-code-guidelines)、[miri](https://github.com/rust-lang/miri) 和 [stacked borrows](https://plv.mpi-sws.org/rustbelt/stacked-borrows/) 之间，我们很多人已经在这方面花了很多心思。Miri 特别有用，因为它可以让我们在实际代码上“踢轮胎”，检查我们感兴趣的语义是否真的被真正的 unsafe Rust 代码所遵守。

他们没有！这就是为什么 [Learn Rust With Entirely Too Many Linked Lists][too-many-lists] 中的 [unsafe 队列](https://rust-unofficial.github.io/too-many-lists/fifth.html)会突然停顿下来，为此对 miri 和 stacked borrows 进行了 4000 字的深入研究！即使是像旧 Rust 中的链式队列这样无聊的东西，也有令人困惑和破灭的语义。

（如果你想了解 stacked borrows，请阅读那一章，我不在这里重述。）

最根本的问题是，在我们对 Rust 的现代理解中，即使*创建*一个引用也是在做一个极强的有效性断言，并对“stacked borrows”产生了副作用，这反过来又改变了哪些引用被认为是无效的。对于对 `T` 的引用来说，这可能包括：

- 该引用是对齐的
- 该引用是非空的
- 指向的内存被分配，并且至少有 `size_of::<T>()` 字节。
- 如果 `T` 存在[无效的值](https://doc.rust-lang.org/nomicon/what-unsafe-does.html)，那引用指向的内存不会是这种无效的值。

所有这些的结果是，*通常*你应该避免混合引用和 unsafe 指针。不安全的代码*通常*应该在其 API 边界提供一个安全引用接口，然后在内部放弃使用引用，并尝试使用 unsafe 指针。这样一来，你就可以把你对低层级数据结构的内存的强烈断言降到最低。

好了，够简单了，对吗？

### 2.3 偏移量和位置是个麻烦

所以你现在想负责任地与 unsafe 指针相处，是时候偏移一个指针了。这很容易，我们有 `ptr::offset/add/sub` 用来实现它！让我们偏移到这个结构的字段......呃......等等，这个字段的偏移是什么？

哦，Rust 只是，并没有告诉我，是吗？好吧，也许你可以做这样的事情：

```rust
&mut (*ptr).field
```

哦，不，等等，那创建了一个引用。是的，即使你马上把它转换成一个裸指针。你怎么能在不创建引用的情况下获取一个地址呢？另外，我用一个引用来初始化一段未初始化的内存是否真的没问题？

这是一个令人困惑的大麻烦。在很长一段时间里，我们试图制定某种“5 秒规则”，即如果你将引用转换为裸指针“足够快”，那么就没有问题，但这对于一个正式的模型来说显然是站不住脚的（我主张这样做，这样做会很好！）。所以人们[为原始地址提出了一个合适的 RFC](https://github.com/rust-lang/rfcs/blob/master/text/2582-raw-reference-mir-operator.md)，并且在很长一段时间里，我们有一个 hacky 的 [addr_of](https://doc.rust-lang.org/stable/std/ptr/macro.addr_of.html) 宏，可以让你这样做：

```rust
addr_of!((*ptr).field)
```

...是的，我也讨厌它。

即使这样也没有把钉子钉在棺材上。最近有[一个非常有经验的 rust 开发者发了一个帖子](https://lucumr.pocoo.org/2022/1/30/unsafe-rust/)，基本上是对目前用未初始化内存做这些事情的困惑的挫败感。同时，在 RFC 中提出的*实际的*事情[似乎已经停滞了多年](https://github.com/rust-lang/rust/issues/64490)，因为研究这个东西的专家们自己也被 [`addr_of` 的边角案例所迷惑](https://github.com/rust-lang/unsafe-code-guidelines/issues/319)。

更糟糕的是，`addr_of` 还使得人们*仍然*想要的静态 [`offsetof`](https://en.cppreference.com/w/cpp/types/offsetof) 很难实现。

我认为，这主要归结于两个事实：

- 对指针的解引用是假的废话
- [Places](https://doc.rust-lang.org/reference/expressions.html#place-expressions-and-value-expressions) 是极其令人困惑的魔法（place 表达式是 Rust 对左值的称呼）

就像如果我们思考一下“解引用”一个指针是什么......它实际上什么都不是？就像解除对指针的引用实际上并没有*做*任何事情。它让你进入“place 表达式”模式，让你指定一个偏移量到被指向者的子字段/索引，然后*实际*发生的事情只在最后指定。例如：

```rust
(*ptr).field1.field2[i].field4;     // load
(*ptr).field1.field2[i].field4 = 5; // store
&(*ptr).field1.field2[i].field4     // offset (*maybe*)
```

这当然是我们*熟悉*的语法，但它确实也是一种神奇的方式，与 Rust 中的自动解引用（autoderef）完全相同。也就是说，它有一定的意义，而且说实话，你不需要考虑它*在安全的 Rust 代码中*发生的事实，因为你知道编译器有你的支持，并且会在出错时帮助你。但在 unsafe Rust 代码中呢？这东西太模糊了。我简直无法判断索引是进入一个切片还是一个内联数组，或者这些 `.` 中的任何一个是否隐式地进行了解引用。

（另外，就内存模型而言，对于解引用本身是否*真的*没有内在的意义，或者它是否做了一些有效性的断言，实际上有一些分歧！）

正如我之前所说的，当你在用 unsafe 指针做一些事情时，你应*保持*在这种模式下。这在目前的 place 表达式设计中是*不可能*的，因为一旦你解引用，你就会处于安全和不安全之间的一个奇怪的模糊状态。

## 3 解决方案

好了，现在我开始走出轨道，并提议对 Rust 进行大修，几乎不考虑“解析”。

### 3.1 区分指针和地址

`usize` 和指针之间的联系需要彻底修改，我会用电锯锯掉它（使用适当的[版本和废弃期](https://doc.rust-lang.org/edition-guide/editions/index.html)）。

下面是我们有品位的电锯的高层次外观：

- 定义指针和地址之间的区别
- 重新定义 `usize` 为地址大小，它 <= 指针大小（通常是 ==）
- 定义 `ptr.addr() -> usize` 和 `ptr.with_addr(usize) -> ptr` 方法
- 弃用 `usize as ptr` 和 `ptr as usize`

#### 3.1.1 重新定义 `usize`

指针仍然是我们所知道的指针，但我们现在承认它指向了一个特定的*地址空间*。指针还包含一个*地址*，从概念上讲，它是这个地址空间内的一个偏移。

（就我而言，对于所有主要的架构*和* CHERI 来说，只有一个地址空间，但在这里有可能值得为正确讨论分段架构中的指针打开大门。尽管从线程的角度来看，将 x64 的 TLS（线程本地存储）实现作为一个独立的地址空间可能更诚实一些）。

一个 `usize` 大到足以包含该平台上所有地址空间的所有地址。对于主要的架构来说，这意味着一个 `usize` 仍然是指针大小。对于 CHERI 来说，这意味着 `usize` 可以（也应该）是一个 `u64`，相当于 CHERI 的 `vaddr_t`。为了保持兼容，我认为仍然要求 `usize`/`isize` 与目标 ABI 中的 `size_t` 和 `ptrdiff_t` 相同是合理的。

（再次希望这个通用的定义对于分段来说是有用的，尽管我已经听到了关于分段平台的讨厌的传言，这些平台实际上将 `size_t` 和 `ptrdiff_t` 解耦，这是可怕的，也许还是我们不想支持的。）

因此，编写*最大限度可移植*的 Rust 的人现在必须替换他们的假设：

```rust
size_of::<usize>() == size_of::<*mut u8>()
```

现在是：

```rust
size_of::<usize>() <= size_of::<*mut u8>()
```

也许应该有一个 `cfg(target_address_size_is_pointer_size)` 之类的东西，允许人们兼容*奇怪的*平台。

我不认为 Rust 需要定义 `intptr_t` 的等价物，如果新的转换 API 像我希望的那样顺利的话——对于大多数有效的目的来说，`*mut ()` 已经是 `intptr_t`。据我所知，CHERI 所做的 `intptr_t` 的把戏并不是“绝对好的和可取的”，而更多的是“可怕的 hacks 来让旧代码工作”。也就是说，人们一直模糊地提到内存映射的硬件是一个可能很重要的地方，但这超出了我的专业领域（而且*不管如何*显然需要特殊的分段/来源/能力来处理）。

#### 3.1.2 替换指针和整数的转换

接下来，用方法来代替转换。所有裸指针和整数之间的相互转换都将被废弃。这*主要*是指 `isize`/`usize`，但不管什么原因，Rust 也允许你把 `ptr as u8` 来做，这就*更加*胡扯了。

下面的新方法将被添加：

````rust
// 对 `*const T` 也一样

impl<T: ?Sized> *mut T {
    /// 获取指针的地址部分
    ///
    /// 在大多数平台上，这是一个 no-op，因为指针只是一个地址，
    /// 并且等价于被弃用的 `ptr as usize` 转换。
    ///
    /// 在更复杂的平台上，如 CHERI 和分段式架构，这可能会移除一些重要的元数据。
    /// 请参阅 [`with_addr`][] 以了解这一差异以及它的重要性。
    fn addr(self) -> usize;

    /// 用给定的地址创建一个新的指针。
    ///
    /// 这取代了被弃用的 `usize as ptr` 转换，该转换在根本上破坏了语义，
    /// 因为它不能恢复*分段*和*来源*。
    ///
    /// 一个指针在语义上有 3 个与之相关的信息：
    ///
    /// * 分段：它所处的地址空间的一部分。
    /// * 来源：一个允许其访问的分配（切片）。
    /// * 地址：它所指向的实际地址。
    ///
    /// 编译器和硬件在任何时候都需要正确理解这三个值，以正确执行你的代码。
    ///
    /// 分段和来源是由指针的构造*方式*隐式地定义的，并且通常一字不差地传播到所有派生指针。
    /// 因此，*不可能*将一个地址转换成一个指针，因为没有办法知道它的分段和来源是什么。
    ///
    /// 通过引入一个“有代表性的”指针，我们可以正确地构造一个具有分段和来源的新指针，
    /// 就像任何其他派生指针一样。这*应该*等同于将给定的指针 `wrapping_offset` 到新地址。
    /// 请参阅 `wrapping_offset` 的文档，了解其适用的限制。
    ///
    /// # 例子
    ///
    /// 下面是一个例子，展示了如何正确使用这个 API 来使用带标签的指针。
    /// 这里我们在最低位有一个标签：
    ///
    /// ```rust,ignore
    /// let my_tagged_ptr: *mut T = ...;
    ///
    /// // 获取地址，随便做一些我们喜欢的技巧
    /// let addr = my_tagged_ptr.addr();
    /// let has_tag = (addr & 0x1) != 0;
    /// let real_addr = addr & !0x1;
    ///
    /// // 用新的地址重构一个指针并使用它
    /// let my_untagged_ptr = my_tagged_ptr.with_addr(real_addr);
    /// *my_untagged_ptr = ...;
    /// ```
    fn with_addr(self, addr: usize) -> Self;
}
````

废弃这些转换可能看起来很极端，但在我看来，这和我们[废弃 `mem::uninitialized`](https://gankra.github.io/blah/initialize-me-maybe/) 的情况完全一样。在指针来源（Pointer Provenance）和 CHERI 下，这些转换的设计从根本上说是坏的。每个人都需要使用一个更好的设计，实际上有一个连贯的意义。

现在*从技术上*讲，你可以保留 `ptr as usize`，但我认为出于以下几个原因，替换这两者更好：

- 对两边的转换都发出弃用警告，对*每一个*用 usize 做任何事情的人发出一个巨大的红色标示，他们要做一些思考。
- 文档里不能缺少和转换相关的内容。正如我希望这篇文章所展示的，指针和整数的转换是非常微妙和可怕的，需要有很多详细的文档。
- `ptr as usize` 在实践中是非常笨重的，所以毁掉它说实话是一种怜悯。
- 直接的对称性/美学。只有一边是很奇怪的！

正如文档所指出的，新的 `with_addr` 方法允许我们重组许多东西：

- 地址去到哪个段（希望如此）
- 用于内存模型/别名分析的 _Provenance_
- 用于 CHERI 的元数据（但这只是对 provenance 的再解释）

......就是这些了！它修复了这个问题。这就是你需要修复 provenance 和 CHERI 的全部！(也许还可以支持分段）。

（实际上，可能需要增加一些特殊的 API 来满足现有的对指针和整数转换的使用，但这真的需要在 crates.io 上和社区中进行讨论。）

不清楚的细节：`get_addr`/`with_addr` 对于 [ARMv8.3 指针认证](https://www.qualcomm.com/media/documents/files/whitepaper-pointer-authentication-on-armv8-3.pdf)来说是否也是必要的/有用的？这是[苹果公司提供](https://github.com/apple/llvm-project/blob/next/clang/docs/PointerAuthentication.rst)的一项技术，涉及到一些指针签名/混淆，以使内存安全漏洞更难被利用。我还没有深入研究，不知道这“泄露”到什么抽象层次。我之所以知道，是因为它出现在 minidumps 中，而我们不得不[黑客式地试图将它剥开](https://github.com/rust-minidump/rust-minidump/blob/7eed71e4075e0a81696ccc307d6ac68920de5db5/minidump-processor/src/stackwalker/arm64.rs#L252)。

### 3.2 修复位置和偏移量

好吧，这里有两部分语法技巧的组合，可以使 unsafe 指针的工作变得更容易。我几乎可以*肯定*的是，我将会在这里的某个地方触犯解析器的歧义，但是嘿，这不是一个真正的 RFC，我可以制定规则。也许它工作得很好。也许它可以在一个版本中被修复。让我们来看看！

#### 3.2.1 路径偏移语法

嘿，你知道 Rust [从来没有真正地](https://github.com/rust-lang/reference/pull/1149)从[语法](https://doc.rust-lang.org/nightly/reference/tokens.html#punctuation)中移除波浪号（`~`）吗？

`~` 是吸引我在 2014 年开始接触 Rust 的原因之一，当我得知它已经被删除时，我非常难过。

重铸 `~` 荣光，我辈义不容辞。

以下是我的宏伟愿景，它将解决 Rust 在“保持 unsafe 指针模式”和一般处理偏移量方面的所有问题。如果你写 `my_ptr~field`（类似于 `my_struct.field`），它总是做一个原始的指针偏移，并且不改变间接的层级（但会改变被指向的类型）。

注意这与 C 的 `->` _不同_，因为 `ptr->field` 是 `(*ptr).field`，因此会使你进入“place 表达式”模式。嵌套的 `->` 是来自内存的实际的*加载* (因为你需要得到地址来开始下一级的偏移)。`~` 永远不会导致隐式加载/存储，因为它不会改变间接层次或进入“place 表达式”模式。它是纯粹的语法糖：

```rust
ptr.cast::<u8>().offset(field_offset).cast::<FieldTy>()
```

`ptr~field~subfield` 不需要借助于“place 表达式”，并且总是可以作为二元运算符 `~` 的分段应用来完成（和解析/实现）。

```rust
let ptr_field = ptr~field;
let ptr_subfield = ptr_field~subfield;
```

我们来看看一些例子。

```rust
// 带有一些字段的 non-POD 类型
struct MyType {
    field1: bool,
    field2: Vec, // non-POD!!!
    field3: [u32; 4],
}

// 在一个裸指针上使用 ~ 语法以干净地初始化 MaybeUninit
let init = unsafe {
    let mut uninit = MaybeUninit::<MyType>::uninit();
    let ptr = uninit.as_mut_ptr();

    ptr~field1.write(true);
    ptr~field2.write(vec![]);
    ptr~field3~[0].write(7);
    ptr~field3~[1].write(2);
    ptr~field3~[2].write(12);
    ptr~field3~[3].write(88);

    uninit.assume_init();
};
```

是的 `~[idx]` 有点古怪，但它很*清楚*和*简明*，这是最重要的事情。请注意，在目前的 Rust 中你要这么做：

```rust
let init = unsafe {
    let mut uninit = MaybeUninit::<MyType>::uninit();
    let ptr = uninit.as_mut_ptr();

    addr_of!((*ptr).field1).write(true);
    addr_of!((*ptr).field2).write(vec![]);
    addr_of!((*ptr).field3[0]).write(7);
    addr_of!((*ptr).field3[1]).write(2);
    addr_of!((*ptr).field3[2]).write(12);
    addr_of!((*ptr).field3[3]).write(88);

    uninit.assume_init();
};
```

或者，如果你很聪明，想利用 POD 类型可以不用 `write` 就可以初始化的事实，你的 place 表达式是由裸指针解引用派生出来的：

```rust
let init = unsafe {
    let mut uninit = MaybeUninit::<MyType>::uninit();
    let ptr = uninit.as_mut_ptr();

    (*ptr).field1 = true;
    addr_of!((*ptr).field2).write(vec![]);
    (*ptr).field3[0] = 7;
    (*ptr).field3[1] = 2;
    (*ptr).field3[2] = 12;
    (*ptr).field3[3] = 88;

    uninit.assume_init();
};
```

我*真正*喜欢 `~` 版本的原因是。

- 这都是后缀，就像我们在 `.await` 中学到的那样，是非常好的，也是很好的。看看 `addr_of!((*ptr).field2).write(vec![])` 是多么令人讨厌！
- 每个 `~` 都可以被单独求值——就程序员而言，不需要语言进入一个求值“place”的“模式”。(编译器可以在幕后自由地这样做，只是你不需要知道它）。
- 因为你待在裸指针模式下，所以对于像 `read` 和 `write` 方法这样的事情，就不那么痛苦了。这使得做更复杂的事情如 `read_volatile` 也同样容易，并且不鼓励你“聪明”地倚靠事情碰巧是 POD 的事实。所有的 `write` 都是一种非常好的*无意识的权利*。
- 你永远不必担心不小心被自动解引用（或是其他对安全代码来说很好但对不安全代码来说是十分危险的东西）给绊倒。
- 通过摆脱 `(*ptr).` “语法盐”，程序员就会有动力转而使用更漂亮、更强大的新语法。是的，我*真的*认为这种语法很好！它当然比 C 语言中的 `->` 要好！

我们还可以从概念上扩展这种语法，使其在类型上被允许，并计算出一个常数 `offsetof`（usize 类型的）。请注意，与在值上使用的 `~` 不同，这种语法不能“一步一步”地操作，需要更多地像“路径表达式”那样进行求值。在这种情况下，魔法就不那么重要了，因为任何错误都是一个编译器错误。下面是你如何使用它来做更“简单”的偏移。

```rust
// 在一个类型上使用 ~ 以得到常量 offsetof (usize 类型的)
const MY_FIELD_OFFSET: usize = MyType~field1~[2];

// 如果你想让这在语法上有区别，
// 我们可以要求一些数量的 `::`
//
// 选项 A: MyType::~field~[2];
// 选项 B: MyType::~field::~[2];

// 指向某段内存的指针
// !!IMPORTANT THAT THIS IS IN BYTES!!
let ptr: *mut u8 = ...;

// 我们正在做有害的事，因此使用 wrapping_add
let field_ptr = ptr.wrapping_add(MY_FIELD_OFFSET) as *mut FieldType;
// 以某种方式使用 field_ptr ...
```

我对常量 `offsetof` 不那么热心，但*似乎*可以让它发挥作用，我知道人们真的想要这些东西。

补充说明：

- 它应该传播 `*const` vs `*mut`
- 我不知道这是否有意义或者是一个好主意，但也许你可以在实际的引用和非引用（int、struct、array、tuple...）上使用 `~`，而不仅仅是裸指针？
  - 如果是这样的话，也许应该支持 `val~self` 作为一种更好的后缀方式来表达 `&mut val as *mut _`，但是如果你同时支持引用和非引用的话，这就很混乱了，因为这有可能会引起歧义，不知道你是想要一个裸指针到引用还是只是裸指针。

#### 3.2.2 后缀解引用

这是这里最不完善的想法，但这样我们能用漂亮干净的后缀语法来清理裸指针，如果我们也有后缀解引用，那就*真的*太好了。请注意，由于“创建引用会产生神奇的断言”，你*不能*提供类似 `deref(self) -> &T` 的方法，它实际上与 `*ptr` 有相同的语义！对裸指针的解除引用*必须*有一流的语法。

例如，考虑到试图访问一些多重间接指向的值。

目前的：

```rust
addr_of!((*(*(*ptr1).field1.ptr2).ptr3).field4).read()
```

使用偏移语法的：

```rust
(*(*ptr1~field1~ptr2)~ptr3)~field4.read()
```

用偏移语法和 `read`（记住，我们待在指针模式，所以 `ptr2.read()` 是在从内存加载 `ptr2`，所以我们实际上可以保持只使用 `~` 语法）。

```rust
ptr1~field1~ptr2.read()~ptr3.read()~field4.read()
```

使用后缀解引用：

```rust
// 最明显的语法，但在目前的 Rust 中可能会有歧义
ptr1~field1~ptr2.*~ptr3.*~field4.read()

// 潜在的可行的替代方案
ptr1~field1~ptr2~*~ptr3~*~field4.read()
```

（我想这是我独立想出来的 `.*` 语法，但我非常高兴地得知 Zig [实际上有这个语法](https://ziglang.org/documentation/master/#Pointers)，而且 `ptr.* += 1` 是有效的 Zig 语法！）

`.*` 在 Rust 中的歧义问题是，`1.` 是一个有效的浮点数字面，所以像 `1. * 1.` 是有效的语法，而且很危险地接近于 `1.*.1`。我当然能想象到令人讨厌的歧义！

关于 `~` 语法的一个好处是，因为一元 `*` 已经很弱地绑定了（这就是为什么我们需要做 `(*ptr).field`），如果你只需要做一个解引用，它就像使用普通引用一样干净：

```rust
*ptr~field1~field2 = 5;
```

所以实际上，我越是写这一节，就越觉得*不需要*后缀解引用，但在我的心里，我还是想要它。

哦，你也可以想象想要 `.&` 和 `.&mut`，但类似的语法混乱可能也会发生。

## 4 That’s All!

天哪，这真是太多了。

我经常想到 Rust 中的 unsafe 指针。

我一口气写完了这些，我真的需要嘉晚饭。

脑袋里只有指针。
