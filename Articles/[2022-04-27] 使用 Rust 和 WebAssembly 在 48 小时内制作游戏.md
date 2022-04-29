> 原文链接: https://ianjk.com/rust-gamejam/
>
> **翻译：[子殊](https://github.com/allenli178)**
>
> 选题：[trdthg](https://github.com/trdthg)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

# 使用 Rust 和 WebAssembly 在 48 小时内制作游戏

![wasm](https://github.com/rustt-org/rustt-assets/blob/main/20220426%20Making%20a%20Game%20in%2048%20hours%20with%20Rust%20and%20WebAssembly/wasm.png)

[Ludum Dare](https://ldjam.com/)，世界上首次 48 小时个人游戏开发比赛在四月份举办。

Ludum Dare 有两个路线：“Compo” 和 “Jam”。

Compo 规则要求在 48 小时内制作游戏（源代码除外）。 最后，其他参赛者将对你的游戏进行评分，同时你的游戏将与同行进行排名。

由于新冠病毒，每个人都在家中隔离，今年四月的 Ludum Dare 参赛者比平时多得多。 共有 1383 人进入 Compo，Jam 有 3576 个条目。 这是一个很荒谬的游戏数量！

我已经使用 Unity 进过[几次 Compo](https://ldjam.com/events/ludum-dare/43/blueberry-bounce)，但这次我想单纯使用相对较新的编程语言 Rust 来编写游戏。

这篇文章开始是专注于使用 Rust 的体验，但后来变成了对游戏技术和设计过程的总体概述。

[Rust](https://www.rust-lang.org/) 是一种专注于性能和安全性的“系统编程语言”。 “安全”意味着 Rust 可以帮助您避免某些类别的漏洞和崩溃。 我一直在将 Rust 用于个人工作，并决定现在是时候试一试 gamejam 了。

## 为什么不用 Unity?

我已经使用 Unity c# 参加了至少五个过去的 Ludum Dares，而 Unity 搭配 C# 是迄今为止 Ludum Dare 最常用的工具集。 但我只是最近没那么喜欢 Unity 罢了。

Unity 是一个巨大的游戏引擎，但我发现很难进入流畅状态。 有太多的旋钮、功能和实现的方法。 翻转一堆开关并获得不错的效果很容易，但这与我的个人设计流程不符。 我更喜欢在干净简洁的环境中工作，而不是复杂的编辑器。

此外，Unity 的 WebGL 构建在我的笔记本电脑上大约需要 20 分钟，这绝对是痛苦的。

## 我的设置：计算机、基本代码和工具

我专门在 2016 Macbook Pro 上编写代码已经有一段时间了。 这不是最好的，但也不是最差的。

我本可以使用现有的 Rust 游戏引擎，但我对流行的引擎并不熟悉。 相反，我将 Rust 库和各种个人 Rust 脚本拼凑在一起来制作游戏。

Rust 通过包管理器 [crates.io](https://crates.io/) 使得使用其他库（称为 crates）变得非常容易。

对于这个项目，我使用了以下 crates：

- [kApp](https://github.com/kettle11/kApp) 我个人的窗口和输入库旨在快速构建。
- [wasm-bindgen](https://github.com/rustwasm/wasm-bindgen) Rust 生态系统从 Rust 调用 Javascript 的标准方式。
- [glow](https://github.com/grovesNL/glow) “GL on whatever” OpenGL 和 WebGL 的封装。

使用 Rust 和 WebAssembly (Wasm) 的推荐方法是通过名为 [wasm-pack](https://github.com/rustwasm/wasm-pack) 的命令行工具。 我跳过了使用 `wasm-pack`，而是使用了一个两行 bash 脚本，该脚本将运行 Rust 构建，然后直接调用 `wasm-bindgen` 来生成 Web 绑定。

快速的迭代时间对于 gamejam 来说绝对是至关重要的。 以下工具有助于缩短迭代时间：

- [cargo watch](https://github.com/watchexec/cargo-watch) 会在每次保存代码时自动运行构建脚本。
- [devserver](https://crates.io/crates/devserver/0.1.0) 托管本地网页并在文件更改时自动重新加载。
- Visual Studio Code 配置为自动保存。

`cargo watch`、`devserver` 和 Visual Studio Code 设置的组合意味着我可以在 Rust 代码中编辑一个值，比如颜色，并几乎立即在网页上观察它的变化。

在处理这个项目时，典型的 Rust 构建时间大约是 1-3 秒，但有时它们会莫名其妙地上升到大约 10 秒。 Rust 以缓慢的构建时间而闻名，而这些快速的迭代时间只有通过仔细选择 crate 才能实现。 我的目标是在切换到浏览器窗口时始终在网页上准备好新版本。

## 代码结构

最佳实践只有 48 小时，但即便如此，我还是做出了一些早期选择，以帮助尽可能轻松地编写新代码。 我想要确保的关键事项之一是，如果我要声明一个新变量或加载一个新资产，它只需要声明一次。 许多 Rust 框架都遵循类似于以下的模式：

```rs
struct Game {
    player: Player,
    /* Other stuff */
}

impl Game {
    fn setup(context: &Context) -> Game {
        Game {
            player: Player::new(),
        }
    }

    fn update(context: &Context) {
        /* Respond to user input and redraw here*/
        /* 在这里响应用户输入并重绘 */
    }
}
```

以上在常规情况下工作正常，但是当你添加一个像“player”这样的新变量时，它需要在多个地方声明。 相反，我使用了一个类似于以下的结构：

```rs
fn main() {
    let player = Player::new();
    /* Other setup code here */

    loop {
        /* Respond to user input and redraw here forever*/
        /* 响应用户输入并在这里一直重绘*/
    }
}
```

这让我可以声明一个变量并立即使用它，无需大惊小怪。

## 异步

不幸的是，上述结构不适用于网络。 在 Web 上，主循环必须将控制权返回给浏览器。

解决这个问题的方法是传递一个事件发生时浏览器调用的闭包：

```rs
fn main() {
    let player = Player::new();
    /* Other setup code here */

    run(move |context| {
        /* Respond to events and draw here */
    });
}
```

Rust 窗口和输入框架（如 [Winit](https://github.com/rust-windowing/winit)）使用上述方法。

Web 上的另一个问题是加载静态资源。 在非 Web 平台上，只需等待资源加载完成即可：

```rs
let wind_sound_handle = std::fs::read("wind.wav").unwrap();
```

在网络上，上述操作是不可能的，因为它会阻止将控制权返回给浏览器。 在等待资源从服务器返回时锁定是不合适的，因此所有加载都是异步的。

在 Rust 中可以使用如下方法：

```rs
let wind_sound_handle = fetch_asset("wind.wav");

run(move |context| {
    let wind_sound = None;
    /* Respond to events and draw requests here */
    if let Some(loaded_asset) = wind_sound_handle.get_asset() {
        window_sound = loaded_asset;
    }

    /* Respond to events and draw here */
});
```

但是这种方法可能会导致繁琐的记账，这与您想要的 gamejam 相反。

相反，我决定使用 Rust 相对较新的特性：`async`。 幸运的是，我最近为 `kApp` 添加了 `async` 支持。

Rust 的异步功能为函数生成状态机，允许它暂停并在准备好后恢复。

异步函数看起来与传统的无限游戏循环非常相似：

```rs
async fn run(app: Application, events: Events) {
    let wind_sound = audio::load_audio("wind.wav").await.unwrap();
    loop {
        match events.next_event().await {
            /* Respond to events and draw here */
        }
    }
}
```

这让我可以用一行加载资源，而不必担心任何记账。 完美的！

## 游戏设计

Ludum Dare 的主题是“Keep It Alive”，鉴于新冠病毒的迅速传播，这立即让我觉得过于病态。 我无法激励自己创造一个关于死亡的游戏，我几乎决定完全退出游戏。

相反，我寻找解释主题的替代方法。 当我凝视着公寓窗外的旧金山山丘时，想到了人们在上下班时低下头的方式。 这是一个美丽的世界，但当日常生活变得沉重时，却很难每天注意到美丽。

如果你扮演一个具有这种精神的人来提升人们呢？ 你可以让他们的“奇迹”保持活力。 你会扮演某种人，也许你会把一个通勤的骑自行车的人举到天空中，在那里你会带他们四处走动并向他们展示星星。

我想象你会引导骑自行车的人在天空中收集星星，地面上的人会注意到并敬畏地指出。

很难弄清楚如何将这个想法与游戏玩法结合起来，但我决定的是一款受老式 Flash 游戏 [Line Rider](https://www.linerider.com/) 启发的游戏。

你会为骑自行车的人画线，他们会沿着这些线滚动并撞上可收藏的星星。

## 实现

计划是将角色的物理特性实现为滚动球，然后用角色艺术代替自行车手。

我需要的第一件事是线条渲染。 从之前的 Rust 项目中提取了一些代码，该项目将通过在末端创建带有圆形的矩形来生成网格线段。 我不确定这种粗暴的线连接方法是否会导致问题。 每个单独的线段都是由一堆三角形组成的，所以我觉得可能会因为出现太多线而导致游戏滞后。

我通过一遍又一遍地在整个屏幕上写满线条来进行压力测试，当帧率根本没有下降时，我感到震惊。

对于物理，我做了一些数学运算来找到离球最近的点。 如果该点在球内，则检查球的速度是否朝着该点移动。 如果球正朝着该点移动，则通过推动球的相反方向来“反弹”球。

碰撞代码的核心一点也不长：

```rust
 fn check_lines(&mut self, points: &[Vector3]) {
    let len = points.len();

    for i in (1..len).step_by(2) {
        let (distance, p) = point_with_line_segment(self.position, points[i - 1], points[i]);

        if distance
            < (self.radius + LINE_RADIUS - 0.001/* Allow ball to sink slightly into surface*/)
        {
            let normal_of_collision = (self.position - p).normal();
            let velocity_along_collision = Vector3::dot(normal_of_collision, self.velocity);
            if velocity_along_collision < 0.0 {
                self.velocity -= normal_of_collision * velocity_along_collision * 1.4;
            }
            self.position += normal_of_collision * 0.0001;
        }
    }
}
```

（如果我知道迭代器上的 [chunks](https://doc.rust-lang.org/std/primitive.slice.html#method.chunks) 方法，代码可能会更简洁一些。）

我很惊讶在没有太多调整的情况下球的物理感觉有多棒。 当我开始这个设计时，物理学对我来说就像一个很大的未知数。 我觉得我可能无法在 48 小时内正确完成它们，但编写物理代码实际上是项目中最短的功能之一。

我非常担心球的物理性能会滞后，毕竟球代码天真地测试了与每条线段的碰撞。 我再一次在屏幕上写满了线条以使帧率下降。 但它没有下降。 在延迟出现之前，它需要在屏幕上写满多次。 所以我决定不做任何优化。 又一次出人意料的胜利！ 感谢 Wasm 和 Rust！

## 现实的打击

很明显，我没有时间画一个骑自行车的人，或者一个敬畏地仰望天空的孩子。 这令人担忧，因为很难用更抽象的图像来传达我想要的主题。

我开始头脑风暴。 这个球让我想起了 花札(Hanafuda) 卡片中的月亮，我开始思考从极简主义中汲取的美学。

![A Hanafuda card with a moon](https://github.com/rustt-org/rustt-assets/blob/main/20220426%20Making%20a%20Game%20in%2048%20hours%20with%20Rust%20and%20WebAssembly/hanafuda.jpeg)

带月亮的一张花札卡片

我跳到 Figma 中来模拟从这个灵感中绘制的美学：

![Using gradients to evoke a sense of night and early morning](https://github.com/rustt-org/rustt-assets/blob/main/20220426%20Making%20a%20Game%20in%2048%20hours%20with%20Rust%20and%20WebAssembly/Screen%20Shot%202020-04-18%20at%201.22.26%20AM.png)

使用渐变色来唤起夜晚和清晨的感觉

新的美学对我来说是一种看起来很棒并且可以实现的东西。 感觉这将是一个美丽的小时刻，点击月亮，看着它摇晃并从天空中滚出。 月亮可以滚动并收集可以代表其他星星或露珠的小圆圈。 玩家将画线以引导月亮进入圆圈。

我想起了日本的[Yūgen](https://en.wikipedia.org/wiki/Japanese_aesthetics#Y%C5%ABgen)概念：

&nbsp;&nbsp;&nbsp;&nbsp;[Yūgen] 描述诗中隐隐约约的事物的微妙深奥

不幸的是，感觉很难用这种美学来传达游戏如何与“保持活力”主题相关联。

此时已经是凌晨 1 点，我决定睡一觉。 我睡着了，想着用这种新的极简主义美学来传达主题的方法。

## 发现隐秘的力量

![pic](https://github.com/rustt-org/rustt-assets/blob/main/20220426%20Making%20a%20Game%20in%2048%20hours%20with%20Rust%20and%20WebAssembly/Screen%20Shot%202020-04-19%20at%2011.23.24%20PM.png)

当我进入梦乡时，我的思绪发现了游戏设计中隐藏的力量。 我在游戏设计中最喜欢的事情之一就是找到提供不成比例价值的简单技术。 让人“嗯，那真是太酷了”的技术，但需要很少的努力。

我昏昏欲睡时的思考过程是这样的：

需要为玩家设置障碍的关卡。

但是我该如何设计关卡呢？

我可以使用我已经使用的设计工具：使用鼠标绘制线条。

我的输入将被记录下来，以在关卡开始时播放绘图线。

第二天早上，我迅速着手编写这个想法。 一个按键将开始记录输入，另一个按键将倒带，另一个将数据保存为文件中的一堆数字。

在播放输入时，我试图保留我用鼠标所做的暂停。 这是为了保留绘图的一些人性化设置，而不是让回放感觉过于数字化和不人道。 如果我暂停的时间过长，我会将暂停时间缩短到最大长度，但该代码很挑剔，有时即使在最终版本中也会暂停时间过长。

## 叙述

因为我正在努力捕捉人类绘画和写作的品质，所以我开始思考如何通过一系列手写笔记来传达简短的叙述。

我开始思考人们是如何给别人写感人的信的，我开始想象一个抽象的故事，一个人给身体不好的人写信。 我想过给医院里的人写一封信，让他们的“奇迹”保持活力，给他们一些愉快的时刻。 那里有一个二元性，通过让某人的“奇迹”保持活力，你也可以帮助保持他们的身体活力。 也许这款游戏既能唤起在这些困难时期保持人类奇迹活力的感觉，又能唤起个人的奇迹？

我对触及如此沉重的主题犹豫不决，但我决定将这些想法抽象地编织到关卡中。

一个关卡会有手写笔记，上面写着“还记得星夜吗？” 附上一张图。 每个音符都旨在作为对人类的信息，也许是对另一个人的信息。 通过扮演滚动球，有时是月亮，你会成为提醒这些美好回忆的力量。

不幸的是，除了线条之外没有时间做艺术，所以我决定在游戏中编织更多类型的记忆。 后来我添加了一些愚蠢的记忆，因为我觉得游戏的基调需要一些轻浮的时刻。

## 同情

整个计划的不幸缺陷是我的笔迹非常糟糕。 我连接了绘图板，但即使使用绘图板，我也发现自己一遍又一遍地重新绘制关卡，试图让文字清晰易读。 如果我只是有更好的笔迹，游戏会明显更好。

但是当我一遍又一遍地重写这些信息时，我开始真正感觉到我正在给玩家写信息。 当时很多人因为冠状病毒而感到困惑和害怕。

随着我写下的每一个小记忆，我开始思考，“嘿，也许这真的会让某人感觉更好？” 这种基调开始贯穿整个比赛。

我松散地认为文本是一个想象中的人给另一个人的信息，一个自然给人类的信息，它变成了我自己给玩家的信息。

它很活泼，但我很困，我发现自己被吸引了。

## 保持活力

游戏是否成功地传达了对“Keep it Alive”主题的这种不同寻常且极其抽象的诠释？

好吧，让我们看看审稿人是怎么想的：

![reviewer](https://github.com/rustt-org/rustt-assets/blob/main/20220426%20Making%20a%20Game%20in%2048%20hours%20with%20Rust%20and%20WebAssembly/Screen%20Shot%202020-07-13%20at%206.18.22%20PM.png)

游戏未能将主题传达给大多数玩家。 主题通过独特的诠释在美学上连接起来，但没有机械连接。 尽管在这个类别中没有成功，但我很高兴我在对主题的解释中承担了设计风险。

## 关卡设计

因为在关卡设计工具上花了很长时间，所以制作它们的过程既有趣又流畅。 游戏的设计长度足以让玩家满意，但又不会太长以至于人们会在中途停止玩游戏。

由于游戏的主题，我试图将游戏设计得非常简单。 如果玩家被挑战到沮丧的地步并感到恼火而不是更快乐，那将是很糟糕的。 回想起来，我在简单的难度下做得有点过头了。 希望玩家可以通过创造性地使用机制来找到乐趣，但我可以设计关卡来推动他们更具创造力。

## 零碎的东西

对于声音，我做了我一直做的事情：在最后一两个小时内惊慌失措。

我借了一个麦克风，用嘴发出风声作为背景氛围。为了收集星星的声音，我把厨房里的每一个物体和玻璃都碰在一起，直到我找到最像星星闪烁的声音。然后我将声音调高一点，感觉更神奇一点。

当它们都是同一个音符时，这些集合的声音感觉太枯燥了。我尝试随意改变音高，但随后一系列快速收集感觉像是不和谐的混乱。所以我所做的就是根据关卡中收藏品的垂直位置向上或向下移动声音。当球收集一系列音符时，这具有演奏令人愉悦的音阶的良好效果。保留了一点点随机性，因此一系列快速收集的球听起来不会太相似。

这些调整之所以成为可能，是因为在一开始就花费了精力来建立一个能够快速迭代的流程。

对于滚动的月亮/球，再次使用风声，但球的速度调节了声音的音量。这些声音有助于给球带来一种漂浮的感觉。

## 结果和结语

该游戏总排名第 71 位，Fun 排名第 16 位。 这是相当不错的！

完整的评分和评论在[这里](https://ldjam.com/events/ludum-dare/46/wonder)。

我对收视率很满意，Rust 很棒，而且我没有崩溃。 总而言之，我对结果非常满意。

对于未来的游戏，我希望有更多的 Rust 帮助代码可以使用。 将 WebAudio Javascript 调用组合在一起是低点之一，并且可以通过更多准备轻松避免。

我鼓励其他人学习 Rust 和 WebAssembly。 如果您不害怕自己发明一些东西，这是一个很棒的组合。

Wonder 的代码非常混乱，但您可以在[此处](https://github.com/kettle11/LD46)查看。

你可以在 itch.io 上玩 [Wonder](https://kettlecorn.itch.io/wonder)！

![wonder](https://github.com/rustt-org/rustt-assets/blob/main/20220426%20Making%20a%20Game%20in%2048%20hours%20with%20Rust%20and%20WebAssembly/Screen%20Shot%202020-04-22%20at%204.36.25%20PM.png)
