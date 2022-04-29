pr前请阅读本文第一章节，确保语法通顺，表意清楚且正确

# Rust's Type System is Turing-Complete

*（注意：“fuck” 在本文中出现多次，但这个词在这里并不是污言秽语。）* 

不久前，有人在 [Reddit Rust 子论坛](https://www.reddit.com/r/rust/comments/5y4x9r/challenge_rusts_type_system_is_not_turing_complete/)上提出挑战：虽然每个人都喜欢说 Rust 的类型系统是图灵完备的，但实际上似乎还没有人给出确凿的证据。作为对那篇文章的回应，我以使用[Rust 实现 Smallfuck](https://github.com/sdleffler/tarpit-rs)（一种已知的图灵完备语言）的方式给出了一个证明，说明 Rust 的类型系统是图灵完备的。这篇文章将阐述 Smallfuck 的 Rust 实现的内部工作原理，以及这对 Rust 类型系统的意义。



那么什么是图灵完备性呢？图灵完备性是大多数编程语言的一个属性，它表明这些语言可以模拟通用图灵机。与之相关的概念是图灵等效。图灵等效语言可以模拟图灵机也可以被图灵机模拟——因此，如果你有任何两种图灵等效语言，则一定可以将使用一种语言编写的任何程序翻译成另一种语言编写的程序。大多数图灵完备的系统也被认为是图灵等效的。（[wiki](https://en.wikipedia.org/wiki/Turing_completeness#Formal_definitions)）



关于图灵完备性，有几件重要的事情需要注意。我们知道[停机问题](https://en.wikipedia.org/wiki/Halting_problem)， 如果一个系统是图灵完备的，那么也意味着停机问题不可解决。也就是说，如果你有一种图灵完备的语言，它可以模拟任何通用图灵机，那么它必须能够无限循环。这就是为什么知道 Rust 的类型系统是否是图灵完备的很有用——这意味着，如果你能够将图灵完备的语言编码到 Rust 的类型系统中，那么检查 Rust 程序以确保它的类型是否正确的过程必须是[不可判定的问题](https://en.wikipedia.org/wiki/Undecidable_problem)。类型检查器必须能够进行无限循环。



我们如何证明 Rust 类型系统是图灵完备的？最直接的方法（实际上我不知道其他方法）是在其中实现一种已知的图灵完备语言。如果你可以用一种语言实现图灵完备的语言，那么你可以清楚地在其中模拟任何通用图灵机，也就说明这种语言是图灵完备的。



## Smallfuck: 有用 & 无用

那么，什么是 [Smallfuck](https://esolangs.org/wiki/Smallfuck)？ Smallfuck 是一种极简的编程语言，当内存限制被解除时，它就是图灵完备的。我选择将它在 Rust 类型系统中实现而不是其他图灵完备的语言，就是因为它很简单。



实际上 Smallfuck 非常接近图灵机本身。 根据 Smallfuck 的原始规范声明，它是运行在内存有限的机器上的。然而，如果我们解除这个限制并允许它访问理论上无限的内存数组，那么 Smallfuck 就变成了图灵完备的。所以在这里，我们考虑一种具有无限内存的 Smallfuck 的变体。 本文中 Smallfuck 机器由无限的内存磁带组成，该磁带由包含位的“单元”以及指向该单元的指针组成。

```
                 Pointer |
                         v
...000001000111000001000001111...
```

Smallfuck 程序是由五个指令组成的字符串：

```
< | Pointer decrement
> | Pointer increment
* | Flip current bit
[ | If current bit is 0, jump past the matching ]; else, go to the next instruction
] | Jump back to the matching [ instruction
```

利用这几种指令就能够选择内存单元格并制作循环程序。这是一个简单的示例程序：

```
>*>*>*[*<]
```

这是一个非常简单且完全没用的程序（大多数 Smallfuck 程序都没啥用，这要归因于它完全没有任何类型的 I/O），它只是将指针所指位置的后三位设置为 1，然后使用循环将它们全部置回 0，最后指针停在它开始时候的位置。下面我们来看一下这个过程：

首先，以下是初始状态：

```
Instruction pointer
|               Memory pointer
v               v
>*>*>*[*<] | ...0...
```

第一条指令将指针向右移动。所有单元格默认为 0：

```
 v               v
>*>*>*[*<] | ...00...
```

下一条指令是“翻转当前位”指令，因此我们将指针处的位从 0 翻转到 1。

```
 v               v
>*>*>*[*<] | ...01...
```

这种情况发生了 3 次。我们跳过重复来到循环的开头：

```
      v            v
>*>*>*[*<] | ...0111...
```

现在我们处于循环的开始。 `[` 指令是说，“如果当前位为零，则跳转到匹配的 `]`；否则，转到下一条指令。” 当前指针所指是 1，所以我们转到下一条指令。

```
       v           v
>*>*>*[*<] | ...0111...
```

这会将当前位翻转回零；然后，我们将内存指针移回一个位置。

```
         v        v
>*>*>*[*<] | ...0110...
```

现在我们来到循环结束符`]`，无条件跳回到循环开始处。

```
      v           v
>*>*>*[*<] | ...0110...
```

现在我们再次分支。当前单元格为 0 吗？显然不为 0，然后我们继续：



```
       v          v
>*>*>*[*<] | ...0110...

        v         v
>*>*>*[*<] | ...0100...

         v       v
>*>*>*[*<] | ...0100...

      v          v
>*>*>*[*<] | ...0100...
```

在最后一次循环之后，我们结束了：

```
      v         v
>*>*>*[*<] | ...0000...
```

这正是我们开始的地方：所有单元格归零，并且指针位于其起始位置。



## Smallfuck 在 Rust 中的 Runtime

那么在 Rust 中这个简单的实现会是什么样子呢？首先我要介绍我实现的 Smallfuck Runtime 的实现，以验证 Rust type-level 和 Smallfuck 运行时实现是否一致。我们会将 Smallfuck 程序存储为枚举类型定义的 AST 节点，如下形式：

```rust
enum Program {
    Empty,
    Left(Box<Program>),
    Right(Box<Program>),
    Flip(Box<Program>),
    Loop(Box<(Program, Program)>),
}
```

这种表示方法对于字符串形式的 Smallfuck 程序并非十分精确，但它更容易理解。我们还需要一个类型来表示正在运行的 Smallfuck 程序的状态：

```rust
struct State {
    ptr: u16,
    bits: [u8; (std::u16::MAX as usize + 1) / 8],
}
```

尽管要实现图灵完备，我们在技术上需要无限长的磁带，但我们的运行时实现仅仅是为了检查 Rust 类型系统的语义正确性。所以说，这里我们使用一个有限长的磁带作为近似就可以了。通过使用长度为`(std::u16::MAX as usize + 1) / 8`的`bits`，我们确保每个`u16`指针可以指向的范围内都是可用的。现在我们来定义实现 Smallfuck 程序解释器时需要用到的一些操作：

```rust
impl State {
    fn get_bit(&self, at: u16) -> bool {
        self.bits[(at >> 3) as usize] & (0x1 << (at & 0x7)) != 0
    }

    fn get_current_bit(&self) -> bool {
        self.get_bit(self.ptr)
    }

    fn flip_current_bit(&mut self) {
        self.bits[(self.ptr >> 3) as usize] ^= 0x1 << (self.ptr & 0x7);
    }
}
```

这是一些标准的位操作。我们将 8 位存储到我们的位数组中的一个单元格中，这样一来要找出给定的指针指向哪个单元格：将指针向右移位三位，然后使用指针的低三位作为单元格内部的索引，就找到了目标 bit 位。

```
ptr = 0b1100011010011001
Index into the bytes of `bits` --> 1100011010011 \ 001 <-- Which bit in the byte
```

具体来说，这个寻址操作就如是上面的函数`get_bit`所示，将指针`at>>3`作为单元格寻址，通过`bits[(at >> 3) as usize]`取出单元格，然后使用`0x7`即`0b111`作为掩码取`at`的低三位，作为单元格内的偏移量，将`0x1`左移偏移量，再按位与上单元格就得到了目标bit是1还是0，对应函数返回true和false。同样的`flip_current_bit`也按相同的方式取出目标bit位，然后异或上`0x1`，将原bit位取反。

现在我们有了这些原语，让我们继续实现我们的解释器。我们将通过递归调用一个在枚举类型 Program 上进行模式匹配的函数来实现：

```rust
impl Program {
    fn big_step(&self, state: &mut State) {
        use self::Program::*;

        match *self {
            Empty => unimplemented!(),
            Left(ref next) => unimplemented!(),
            Right(ref next) => unimplemented!(),
            Flip(ref next) => unimplemented!(),
            Loop(ref body_and_next) => unimplemented!(),
        }
    }
}
```

`Empty` 程序不修改任何状态，因此其实现非常简单：

```rust
Empty => {},
```

`Left` 和 `Right` 只是将指针加/减一。由于我们在简单实现中选择使用有限的内存磁带，因此我们使用 wrapping_add 和 wrapping_sub：

```rust
Left(ref next) => {
    state.ptr = state.ptr.wrapping_sub(1);
    next.big_step(state);
},
Right(ref next) => {
    state.ptr = state.ptr.wrapping_add(1);
    next.big_step(state);
},
```

`Flip` 非常简单，因为我们已经写了方便的 `flip_current_bit` 函数：

```rust
Flip(ref next) => {
    state.flip_current_bit();
    next.big_step(state);
},
```

最后是循环。我们需要检查当前位。如果是 1，那么我们执行循环体，然后以更新的状态再次执行循环指令。如果为 0，我们继续循环体后的下一条指令：

```rust
Loop(ref body_and_next) => {
    let (ref body, ref next) = **body_and_next;
    if state.get_current_bit() {
        body.big_step(state);
        self.big_step(state);
    } else {
        next.big_step(state);
    }
},
```

注意，我们必须对 `body_and_next` 进行双重解引用，因为我们是将一个元组`(Program, Program)`装入`Box<>`中（见 Program 的定义）。我们通过 `ref` 来避免将 body 和 next 的所有权从 Box 中移走。最终我们为枚举类型 Program 完成的 `.big_step()` 函数如下所示：

```rust
impl Program {
    fn big_step(&self, state: &mut State) {
        use self::Program::*;
    
        match *self {
            Empty => {},
            Left(ref next) => {
                state.ptr = state.ptr.wrapping_sub(1);
                next.big_step(state);
            },
            Right(ref next) => {
                state.ptr = state.ptr.wrapping_add(1);
                next.big_step(state);
            },
            Flip(ref next) => {
                state.flip_current_bit();
                next.big_step(state);
            },
            Loop(ref body_and_next) => {
                let (ref body, ref next) = **body_and_next;
                if state.get_current_bit() {
                    body.big_step(state);
                    self.big_step(state);
                } else {
                    next.big_step(state);
                }
            },
        }
    }
}
```

我们不会为`State`实现任何类型的调试打印，因为我们不需要。当使用我们的实现 Smallfuck 运行时解释器来检查 Rust type-level 解释器时，我们将通过检查 Rust type-level 解释器的输出，和 Smallfuck 运行时解释器输出，的一致性来完成检验。现在，我们终于可以开始做些有趣的事情了！



## Type-Level Smallfuck in Rust

Rust 有一个称为 *traits* 的特性。 Traits 是一种进行编译时静态分派的方法，也可以用于执行运行时分派（尽管在实践中很少见）我在这里假设读者知道 Rust 中有这些特征并且已经大量使用过。为了实现 Smallfuck，我们将依赖于被称为 *associated types* 的特性。



### Trait resolution and unification

