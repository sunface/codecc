
![](https://raw.githubusercontent.com/asur4s/rustt-assets/main/20220529-how-warp-work/1.jpeg)

Warp 是一个全新的高性能 Terminal，完全由 Rust 构建，使你和你的团队更有效率、CLI 更容易使用。命令的输入编辑器是一个完整的文本编辑器，支持选择、光标定位和快捷键。在视觉上，命令和它们的输出被分组到同一个块中，使用快捷键（上箭头、Ctrl-R）可以打开块的菜单，这些特性使得 Terminal 使用来更直观方便。

在这篇文章，我介绍我们如何建立 Warp 的基础：UI 界面，块，输入编辑器。构建这些基础设施帮助我们解锁了很多惊艳的功能特性。如无穷的历史记录、实时协作，共享环境变量。

设计 Warp 需要我们更加注意将每个层面的技术栈。在一开始，我们设置了关键的要求：
1. 渲染速度：在使用 Terminal 的时候，渲染速度是非常重要的。尤其是有大量输出的命令。在真实环境中，这意味着 Warp 应该始终保持 60FPS，即使是在 4K 和 8K 的显示器上。
2. 与已存在的 Shell 兼容：Warp 需要兼容已存在的主流 Shell，例如：Bash、ZSH、Fish。
3. 多平台，包括 Web：Warp 需要运行在多个平台，包括在浏览器中支持实时协同。
4. 和 Shell 集成并支持块（包括通过 SSH 连接）：支持块特性，Warp 需要与 Shell 中正在运行的 Session 深度整合。
5. 自由的 UI 元素：不像传统的 Terminal 那样只渲染字符，Warp 需要自由的渲染 UI 元素（消息条，菜单等）。
6. 本地化和直观的编辑：Warp 需要一个完整的文本编辑器去支持直观的编辑（如选中文本、光标定位和多光标），这些特性在其他现代程序都很常见。

# Rust + Metal

构建 Warp 最重要的事项之一是渲染速度。许多构建在 Electron 上的 Terminal 都是有力的工具，但很容易出现性能瓶颈。我们想要在 Terminal 上添加一层 UI，因此需要确保选择的技术栈能提供足够的性能，即使渲染是复杂的 UI 界面，Warp 的渲染速度依然处于 Terminal 的上游。

当程序输出文本到 Terminal 屏幕时，这里有几点可能会成为 Terminal 潜在的性能瓶颈，其中包括：
1. 从伪终端读取：从伪终端读取时，解析 ANSI 转移序列可能需要大量运算。尤其是程序正在运行，并且打印了很多输出到屏幕（如使用 `cat` 命令查看大文件）。
2. 渲染：取决于实现方式，在屏幕上渲染像素的花销是昂贵的。如果 Terminal 有输出并且在不断的变化，这很快就会导致 Terminal 低于 60 FPS。
3. 滚动：当 Terminal 的视图窗口满了，在渲染打印新的内容之前，需要将已经被打印内容滚动到上方。滚动速度与视图窗口可见行数成正比。

下图展示了 `vtebench` 在各种 Terminal 滚动时的输出。由于某些原因 Hyper 无法正常运行基准测试，在合理的时间范围内没有停止。

![](https://raw.githubusercontent.com/asur4s/rustt-assets/main/20220529-how-warp-work/2.png)

> 在各种终端中滚动单行的渲染速度。这是通过在不同的 Terminal 中运行 Vtebench 的滚动测试后计算得来，其中所有的 Terminal 保持在相同的窗口大小。由于一些原因，由于某些原因 Hyper 无法正常运行基准测试，在合理的时间范围内没有停止。

对 Electron 进行简短的实验之后，我们很快转到使用 Rust 来构建，并使用 Metal（Mac 的 GPU API）直接渲染。Rust 作为一个低级的系统语言，其极高的性能和强大的开发者社区非常吸引人，cargo 包管理器简单易用，我们整个团队（在这之前没有人写过 Rust）很快就上手了。更重要的，Rust 具有相当广泛的平台支持，我们可以只用一种语言编写，然后为 Mac、Linux、Windows 构建，还可以通过编译到 WASM 用于支持 Web。 

# 为什么使用 GPU 来渲染 UI？

大多数的应用都是在 CPU 上使用标准的系统框架渲染。例如，MacOS 提供了直接在 CPU 上渲染任意图形的 API 。在 CPU 上渲染图形是相对简单的，渲染 2D 图形和文本的速度也相当不错。在 GPU 上的渲染非常接近底层（没有用于渲染形状或者文本的 API），但是非常适合于 3D 图形或高性能任务。

有趣的是，最近很多 Terminal 已经支持使用 GPU 加速渲染（iTerm、Hyper、Kitty、Alacritty 都已经支持）。终端中大多是文本和 2D 图形，看上去在 CPU 上渲染更合适。然而，如今的 Terminal 的渲染比 80 年代复杂得多。今天的 Terminal 需要能在 4K 显示器上渲染高分辨率的文本，并且需要以非常高的 FPS 运行，有些显示器高达 240 hz 甚至更高。

高效地利用 GPU 渲染器（尽可能减小两帧之间的状态变化，对字形光栅化处理，最小化绘制调用的次数）能让 GPU 渲染达到 400+ FPS, 这在 CPU 中渲染是无法做到的。


消除了 CPU 上瓶颈，即使在 4K 显示器上，我们的渲染速度依然远超 144FPS。我们选择了 Metal 而不是 OpenGL 作为我们的 GPU API，是因为我们知道我们要把 MacOS 作为我们的第一个平台。Xcode 中的 Metal 调试工具非常好用，我们能够通过 Xode 检查纹理资源，并轻松测量重要的指标（如帧率、GPU内存大小）。

![](https://raw.githubusercontent.com/asur4s/rustt-assets/main/20220529-how-warp-work/3.png)

# 如何在 GPU 上渲染任意 UI 元素？

在 GPU 上支持渲染任意的 UI 元素是非常麻烦的，使用 Metal 在 GPU 上来渲染时非常底层的操作，这些 API 本质上只允许使用顶点着色器和片段着色器渲染三角形或简单的从纹理渲染图像。

我们对 Warp 进行拆分，发现只需要通过纹理图集来渲染矩形、图像、字体。这一发现极大的降低了使用 Metal 的复杂性。我们用了大约 200 行代码为每个基本元素构建了着色器，再使用基本元素来创建 UI 组件，将新的 UI 组件添加到 Warp 中时不需要再创建着色器，因为它们是由这些基本元素组成的。

然而，我们依然需要高度的抽象，以便在输出的字符上的渲染 UI 元素。总所周知，Rust 缺乏一个稳定的 UI 框架（请看 [areweguiyet.com](https://www.areweguiyet.com/)），如果想支持 Metal 作为渲染后端，可供选择的框架就更少了。我们考虑了 Azul、Druid，然而这两个框架都在实验阶段，并且不支持 Metal（Druid 还不支持任何 GPU 渲染后端，Azul 只支持 OpenGL）

鉴于缺乏稳定的选择，我们决定使用 Rust 构建我们自己的 UI 框架，这实质上相当于构建一个浏览器架构。从头构建框架会对工程造成很大的影响，我们和 Atom 文本编辑器的联合创始人 Nathan Sobo 合作，他已经开始构建 Rust UI 框架，其灵感大致来源于 Flutter。我们使用这个框架为我们的 App 构建了 UI 元素树，然后我们可以使用不同的渲染后端（目前只是 Metal，但当我们计划支持更多平台时，会继续添加对 OpenGL 和 WebGL 的支持）。

![](https://raw.githubusercontent.com/asur4s/rustt-assets/main/20220529-how-warp-work/4.png)

在渲染层面，我们从建立基础元素开始（如矩形、图像、字形）。我们再利用这些基本元素来构建高层次的元素（如消息条、右键菜单、块）。我们希望将元素和实际的 GPU 渲染分离，这可以帮助我们更好的支持其他 GPU 的渲染：我们只需要在其他 GPU API 上重新实现这些基本元素（< 250 行着色器代码），不需要改动更高层次的东西。我们希望在迭代的过程中将这个 UI 框架开源，我们认为它已经是一个可用的 Rust UI 框架。

迄今为止，在 GPU 上渲染已经有非常显著的成效，即使有许多 UI 元素和大量的 Terminal 输出，我们依然能保持帧率大于 144 FPS。例如，过去一周，Warp 重绘整个屏幕的平均时间只有 1.9 ms！

![](https://raw.githubusercontent.com/asur4s/rustt-assets/main/20220529-how-warp-work/5.png)

# 实现块

![](https://raw.githubusercontent.com/asur4s/rustt-assets/main/20220529-how-warp-work/6.png)

在大多数 Terminal 中，你从未看见过像块一样的特性，因为 Terminal 不知道哪个程序正在运行，也不知道在 Shell 中发生了什么。Terminal 在更高的层次，通过从伪终端读取和写入字符实现和 Shell 交互。这个技术是非常过时的——Shell 本质上认为它是和一个物理的 TTY 交互，尽管物理 TTY 已经有 30 多年没有在实践中使用了。

Terminal 实现了 VT100 标准，这种标准将信息从 Shell 传递给 Terminal。例如，如果 Shell 想要告诉 Terminal 将文本渲染成红色、粗体、下划线，将会发送下面这段转义代码，然后 Terminal 将会解析转移代码并且使用适当的样式渲染。

```
\033[31;1;4mHello\033[0m
```
> https://stackoverflow.com/questions/4842424/list-of-ansi-color-escape-sequences 很好的回答了如何通过转义序列表示颜色，它只是我最喜欢的 StackOverflow 回答之一——请务必阅读其中的插曲。

所有的 Terminal 都只能看见字符的输入和输出，仅使用从伪终端的字符去判断命令何时被运行，这几乎是不可能的。

值得庆幸的是，大多数的 Shell 都提供了提示符出现前（Zsh 中叫做 Precmd）和命令执行前的钩子（preexec）。 Zsh 和 Fish已经内建支持这些钩子，尽管 Bash 并未支持这些钩子，但存在像 bash-preexec 的脚本来模拟其他 shell 的行为。使用这些钩子，我们从运行中 session 向 warp 发送自定义的 DCS（设备控制字符串），这个 DCS 包含了一个编码的 JSON 字符串，字符串包含了我们想要呈现的会话的元数据。在 Warp 中，我们可以解析这个 DCS，将得到的 JSON 反序列化，并在我们的数据模型中创建一个新的块。

# 数据模型

VT100 标准用行和列表示视图窗口，这很自然的导致大多数 Terminal 使用网格作为他们的内部数据模型。我们借用了 Alacritty 的数据模型代码，它非常适合我们。因为它通过 Rust 编写，并尽可能的做到性能优化。Alacritty 的两位主要维护者 Nathan Lilienthal 和 Joe Wilm 也是 Warp 的早期支持者，他们帮助我们审查早期的设计文稿，并提供了宝贵的意见。

然而，我们很快就意识到，单一的网格系统不支持构建像块一样的特性：将命令相互分离需要确保 Terminal 的输出被写入不同行上，并且不能被另一个命令的输出覆盖。VT100 标准不能保证上面的要求。考虑下面这样的命令：

```bash
bash-5.1$ printf "hello"
hellobash-5.1$
```

这里的 `printf "hello"` 输出了 `"hello"`文本，但是没有换行，所以下一行的提示符被插入到了同一行。ANSI 转义序列也存在一些的问题，它允许在可视屏幕内中移动光标，允许其他命令覆盖已执行的命令。我们并不希望命令的输出能覆盖上一个块内容。

为了解决这些问题，我们根据 precmd/preexec 钩子的输出，为每个命令和输出都创建单独的网格。这种单独的网格确保我们能将命令和他们的输出分开，而不必处理一个命令的输出会覆盖另一个命令的输出的问题。我们发现这种方法很好的平衡了 VT100 标准和使用 Alacritty 网格代码来渲染块。这个数据模型给了我们很高的灵活性，可以为单一的块实现 Termianl 中的传统功能，例如，如果我们将每个命令和输出分离到它们自己的子模型中，那就可以很轻松的实现每个块搜索、单独复制命令/输出。

# 输入编辑器

![](https://raw.githubusercontent.com/asur4s/rustt-assets/main/20220529-how-warp-work/7.png)

我们和 Nathan Sobo 一起构建了完整的文本编辑器作为 Warp 输入编辑器。在编辑器中实现了多光标、选中等功能，我们还可以将这个编辑器作为 Shell 中默认的编辑器。

我们构建自己的命令行编辑器，因此我们需要重新实现用户已经习惯的快捷键（如上箭头、Ctrl+R、Tab补全），同时加入新的快捷键，如在文本中移动光标和选中所有。我们在框架中构建了一个新的动作调度系统，它能够被键盘绑定来触发，实现这些快捷键并不困难。调度系统还可以根据 Warp 的状态来启用和禁用不同的快捷键，例如 Ctrl-R（搜索先前的命令）应该只能被用于输入编辑器为可视状态时。

构建编辑器也非常依赖于 SumTree，这是一种基于 Rope 的数据结构。这种类型可以容纳通用类型，并且可以在不同维度进行索引类型。

![](https://raw.githubusercontent.com/asur4s/rustt-assets/main/20220529-how-warp-work/8.png)

就如同许多文本编辑器，我们使用 SumTree 去保存缓冲区的内容，我们也能用 SubTree 对编辑器进行改造，这样不会影响到缓冲区的文本内容（例如代码折叠，显示无可选择和不能编辑的标注等）。使用 SumTree 还可以支持高效的搜索缓存文本和已显示文本中的内容。我们还使用 SumTree 保存编辑操作。为了后期支持实时协作，编辑器中还实现了 CRDT。

# 展望

展望未来，我们的技术栈有能力实现我们对 Warp 的全部愿景。通过选择 Rust 和构建一个与渲染平台无关的 UI 框架，我们可以通过编译到 WASM 和使用 WebGL 渲染来拓展到其他平台，甚至是 Web 平台（我们犯了一个错误，早期没有考虑哪些板块需要编译到 WASM）。

实时协作是 Web 端的功能之一。你可以分享一个安全连接给你的团队成员来实时定位问题。https://github.com/warpdotdev/warp/issues/2。

块模型让用户可以通过更多的方式来搜索历史记录。https://github.com/warpdotdev/warp/issues/1。

我们的 precmd 和 preexec 钩子为环境变量共享提供必要的技术基础。 https://github.com/warpdotdev/warp/issues/3。


# 总结

性能是我们最重要的特性之一——它是大多数用户在日常工作中不会注意到的功能。但在构建终端产品时，性能是必不可少的特性。

如果你有兴趣了解更多，请查看我们的网站，并在这里申请提前进入我们的测试版。我们依然处在实施我们愿景的早期阶段，但在我们前进的过程中，希望能得到您的反馈和意见。


> *本文选自 [Rust 翻译计划](https://rustt.org)，由 [RustCn](https://hirust.cn) 荣誉推出*
> 
> **翻译：[Asura](https://github.com/asur4s)**
> 选题：[Asura](Asura)
>
> 原文链接: https://www.rust-lang.org