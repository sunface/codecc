# 由 Java/C#/C/C++ 开发者转为 Rustacean

正如我之前所说，我正在写一本关于生命周期的书。或者说，这只是一个长篇系列 —— 我还没有决定具体内容。它跟我的其他系列或书一样，篇幅很长，不会直入主题，而是从其他方面开始循序渐进地到达主题。

换句话说 —— 如果你想要开启一次冒险之旅（真正掌握 Rust 确实是这样的体验），是很棒的。但是如果你对 Rust 编译器还有困惑，最好还是先加深对其的理解。

因此，我们尝试用另一种方法来解决问题的症结 —— 期望是一种更直接的方法。

## 我想要构建一个 app

假如我想构建一个应用程序。再假如，我之前写过其他的语言 —— 可能是 Java、C#、C 或 C++，也可能是其他语言。

```bash
$ cargo new an-app
     Created binary (application) `some-app` package
```

假设我想要构建的是一个图形化的 app，包含一个带有大小属性的窗口。我可以这么写：

```rust
// in `src/main.rs`

const WIDTH: usize = 1280;
const HEIGHT: usize = 720;

fn main() {
    println!("Should make a {}x{} window.", WIDTH, HEIGHT);
}
```

然后构建：

```bash
$ cargo run --quiet
Should make a 1280x720 window.
```

但是如果我之前是一个 Java/C<sup>#</sup>/C/C++ 开发者，我会觉得这么写不优雅，所以不这么写，而是（既然是要创建一个 app，那么就）创建一个 `App` 结构体：

```rust
struct App {
    width: usize,
    height: usize,
}

fn main() {
    let app = App {
        width: 1280,
        height: 720,
    };

    println!("Should make a {}x{} window.", app.width, app.height);
}
```

现在感觉好多了。它看起来更优雅 —— 而且它把相关的值组合在一起了。它不仅仅源码中的“两个全局变量”，更是应用程序的一部分 —— 应用程序的窗口有大小属性。

我们再假设，这个应用程序的窗口需要有自己的标题，因为我强烈地认为人们应该在窗口中而不是全屏使用我的应用程序 —— 所以他们肯定会看到标题栏。

所以，我查了一下 `rust string`，我发现有一个名为 `String` 的类型 —— 这感觉很熟悉。Java 也有一个名为 `String` 的类型。C也有 —— 它甚至有 `string` 关键字作为它的别名。C 中的字符串到现在还是个未解决的问题，而 C++ 有一大堆的字符串类型。

```rust
struct App {
    width: usize,
    height: usize,
    title: String,
}

fn main() {
    let app = App {
        width: 1280,
        height: 720,
        title: "My app",
    };

    println!(
        "Should make a {}x{} window with title {}",
        app.width, app.height, app.title
    );
}
```

但不幸的是，上面代码不能运行：

```bash
cargo check --quiet
error[E0308]: mismatched types
  --> src/main.rs:11:16
   |
11 |         title: "My app",
   |                ^^^^^^^^
   |                |
   |                expected struct `std::string::String`, found `&str`
   |                help: try using a conversion method: `"My app".to_string()`

error: aborting due to previous error
```

但我没有被吓到，因为在我想放弃学习 Rust 的时候，我听到很多人说："你会明白的，这很难，但编译器会支持你的，所以就顺其自然吧"。

所以，我参照编译器的提示去做：

```rust
    let app = App {
        width: 1280,
        height: 720,
        // new:
        title: "My app".to_string(),
    };
}
```

```bash
$ cargo run --quiet
Should make a 1280x720 window with title My app
```

到目前为止，一切都很好。

在构建应用程序的过程中，我意识到现在我们有三个字段，其中两个字段看起来应该同时使用：这个 app 有很多图形元素，其中一些元素肯定会有一个宽度和一个高度。

第一个结构写完了，再写另一个：

```rust
struct App {
    dimensions: Dimensions,
    title: String,
}

struct Dimensions {
    width: usize,
    height: usize,
}

fn main() {
    let app = App {
        dimensions: Dimensions {
            width: 1280,
            height: 720,
        },
        title: "My app".to_string(),
    };

    println!(
        "Should make a {}x{} window with title {}",
        app.dimensions.width, app.dimensions.height, app.title
    );
}
```

现在，我的第一个 Rust 项目就建好了，它一个图形化的应用程序。

既然我要开始学习 Rust，那么就需要有一个我真正感兴趣的项目。我知道这听起来像是在炫耀，但我以前确实用 Java、C#、C 或 C++做过很多图形应用程序，所以我“只”需要弄清楚 Rust 和它们之间的一些小差别，这应该没什么问题。

如果我对 Rust 有更高的要求，我可能会想在 `Dimensions`上实现 `Display` 或 `Debug` 特性，因为我想要有大量的打印调试，而 `println！`所在的行有点长。

但是现在，我还不知道什么是 `trait`（难道他们不能直接叫它类或接口吗？），所以我还是用这段没有 `trait` 的代码，它的好处就是简单易懂。

## 现在回到 app 本身

虽然我很高兴我解决了第一个编译器错误，但还有很多事情要做。我通过查阅 `crate`（他们就不能叫它库吗？），并改编其 README 中的代码，很快地创建了一个窗口（为简洁起见在此省略）。

我不得不说，`cargo` 很好。我还不知道我对 `rustc` 的感觉如何，但 `cargo` 很不错。我当然希望Java、C#、C 或 C++有这样的东西。

**它们真的没有 `cargo` 这样的东西吗？**

它们是某些工具，但是并不像 `cargo` 这样。

但现在我已经有了一个窗口，并且正在运行，我开始考虑我的应用程序的逻辑问题。它将如何工作？

如果这是 Java、C#、C 或 C++，我完全知道我应该怎么做。我会让 `App` 来处理窗口的创建，也许还有键盘、鼠标和游戏板的输入，以及所有通用的、日常的设置和记账操作，然后我会把所有项目特定的逻辑放在另一个类（或结构）中。

我以前已经做过很多次了。事实上，我已经自己写了一套*框架* —— 用 Java、C#、C 或 C++ 写的，可以让我跳过无聊的设置和记账部分，而专注于逻辑。

在我的 Java、C# 或 C++ 框架中，我有一个基类 `Client`，它有 `update` 和 `render` 等方法 —— 它们默认什么都不做。但是当我对它进行子类化时，对于我的每个项目，我只需要重写 `update` 和 `render` 方法，以完成项目实际需要做的事情。

而我的 `App` 类 —— 我到处都在重复使用这个类。它包含一个引用（Java/C#）或一个指针（C++），或一个指向数据指针 + 一个指向由所有函数指针构成的结构体的指针（C），每当 `App` 决定执行 `update` 时，它就调用 `this.client.update(...)`，最后它会使用 `Project47::update` —— 逻辑是项目 47 的逻辑，而不是什么都不做的 `Client::update` 的逻辑。

**你在 Java、C#、C 或 C++ 中不会引入示例代码吗？**

不，我不会 —— 因为要么读者知道我在说什么，因为他们已经写 Java、C#、C 或 C++ 很多年了；要么他们不知道，在这种情况下，他们对于这个问题没有什么概念，他们在读一些其他的文章时会从另一个角度遇到这个问题。

**那么他们应该停止阅读本文吗？他们已经在本文花了 6 分钟了。**

没有必要 —— 我*将会*展示 Rust 代码是如何“模拟”他们*以前*写的 Java、C#、C 或 C++ 代码的。

因此，

基于之前的经验，我做了一点研究，看看我如何用 Rust 实现同样的事情。我并不特别高兴地发现 Rust 没有类。

这意味着我现在有*两样*东西要学：生命周期（Rust 的倡导者们一直在强调这一点）和特征。

由于这一切看起来令人生畏，因此在我构建一个框架以使我能够在没有所有模板的情况下制作图形 app 之前，我尝试用简单的方法来做 —— 直接将逻辑塞入 `App`。

我还决定，我的第一个图形化 app 实际上*不是*图形化的，它只是简单地输出行到终端，所以暂时我不必担心我是否应该使用 OpenGL、Vulkan、或 wgpu，或者也许我应该基于 macOS/iOS 的构建做我自己的抽象层，虽然我真的喜欢 DirectX 12，或许我应该提供一个转换层？

我之后再考虑这个问题 —— 现在我把视窗代码注释掉，只专注于制作一个能在终端运行的简单 app。

这将是一个 "jack in the box" 游戏 —— 你不停地转动曲柄，然后 Jack 就会跳出来。

基于以前 Java、C# 或 C++的知识，我写了一个*看起来应该可以正常运行*的东西（可能涉及的 C 只知识多一点）。

```rust
// THE FOLLOWING CODE DOES NOT COMPILE 
// (and is overall quite different from valid Rust code)

use std::{time::Duration, thread::sleep};

fn main() {
    let app = App {
        title: "Jack in the box".to_string(),
        ticks_left: 4,
        running: true,
    };

    println!("=== You are now playing {} ===", app.title);

    loop {
        app.update();
        app.render();

        if !app.running {
            break;
        }
        sleep(Duration::from_secs(1));
    }
}

struct App {
    title: String,
    ticks_left: usize,
    running: bool,

    fn update() {
        this.ticks_left -= 1;
        if this.ticks_left == 0 {
            this.running = false;
        }
    }

    fn render() {
        if this.ticks_left > 0 {
            println!("You turn the crank...");
        } else {
            println!("Jack POPS OUT OF THE BOX");
        }
    }
}
```

这*根本*运行不了。Rust 编译器报了很多错。

```bash
$ cargo check --quiet                                  
error: expected identifier, found keyword `fn`                                                                
  --> src/main.rs:28:5                                                                                        
   |                                                                                                          
28 |     fn update() {                                 
   |     ^^ expected identifier, found keyword                                                                
                                                                                                              
error: expected `:`, found `update`                                                                           
  --> src/main.rs:28:8                                 
   |                                                                                                          
28 |     fn update() {                                                                                        
   |        ^^^^^^ expected `:`                                                                               
                                                                                                              
error[E0599]: no method named `update` found for struct `App` in the current scope
  --> src/main.rs:13:13
   |
13 |         app.update();
   |             ^^^^^^ method not found in `App`
...
23 | struct App {
   | ---------- method `update` not found for this

error[E0599]: no method named `render` found for struct `App` in the current scope
  --> src/main.rs:14:13
   |
14 |         app.render();
   |             ^^^^^^ method not found in `App`
...
23 | struct App {
   | ---------- method `render` not found for this

error: aborting due to 4 previous errors
```

我被迫去网上搜索了一下，发现与 Java、C# 或 C++ 不同的是，在包含字段的 `struct` `class` 的代码块内不能实现方法。

很奇怪的设计，但没关系 —— 显然，我需要的是一个 `impl` 块。

```rust
struct App {
    title: String,
    ticks_left: usize,
    running: bool,
}

impl App {
    fn update() {
        this.ticks_left -= 1;
        if this.ticks_left == 0 {
            this.running = false;
        }
    }

    fn render() {
        if this.ticks_left > 0 {
            println!("You turn the crank...");
        } else {
            println!("Jack POPS OUT OF THE BOX");
        }
    }
}
```

这显然不足以使我的例子发挥作用（不过对我来说看起来不错）--现在它在抱怨这个问题。

显然我的例子还不能运行（虽然我看起来没问题）—— 现在编译器报 `this` 的错误：

```rust
cargo check --quiet
error[E0425]: cannot find value `this` in this scope
  --> src/main.rs:31:9                                                                                        
   |                   
31 |         this.ticks_left -= 1;
   |         ^^^^ not found in this scope
                           
(cut)
```

经过几次谷歌搜索后，我了解到，与 Java、C# 或 C++ 不同，没有 "静态方法" 和 "非静态" 方法的概念。(`static` 关键字确实存在，但它只具有 C++ 中 `static` 的*其中一个*功能）。

然而，我所写的 `update` 和 `render` 实际上已经非常接近"静态方法"了。我了解到，在 Rust 中，不存在隐含的 `this` 指针。

`impl` 块中的所有 `fn 项`（函数）必须声明*所有的输入*，如果我们想使用 `this`（在 Rust 中称为“接收者”），我们也需要把它写出来。

**在 Rust 中有 `self`，没有 `this`**

如果第一个参数*不是*接收者，那么它就不是“方法”，而是“关联函数”，你可以这样调用：

```rust
    let app = App { /* ... */ };

    loop {
        // over here:
        App::update();
        App::render();

        if !app.running {
            break;
        }
        sleep(Duration::from_secs(1));
    }
```

但是它不能访问我们上面初始化的 `App` 的实例 `app` 的字段。

如果我们想要用 `this`，那么就需要显式地添加 `add`：

```rust
struct App {
    title: String,
    ticks_left: usize,
    running: bool,
}

impl App {
    fn update(self) {
        self.ticks_left -= 1;
        if self.ticks_left == 0 {
            self.running = false;
        }
    }

    fn render(self) {
        if self.ticks_left > 0 {
            println!("You turn the crank...");
        } else {
            println!("Jack POPS OUT OF THE BOX");
        }
    }
}
```

现在代码中没有错误了。

我把 `app.update()` 改成了 `App::update()` 来展示“关联函数”，但现在报 loop 的错误：

```rust
$ cargo check --quiet
error[E0061]: this function takes 1 argument but 0 arguments were supplied
  --> src/main.rs:13:9
   |
13 |         App::update();
   |         ^^^^^^^^^^^-- supplied 0 arguments
   |         |
   |         expected 1 argument
...
30 |     fn update(self) {
   |     --------------- defined here

error[E0061]: this function takes 1 argument but 0 arguments were supplied
  --> src/main.rs:14:9
   |
14 |         App::render();
   |         ^^^^^^^^^^^-- supplied 0 arguments
   |         |
   |         expected 1 argument
...
34 |     fn render(self) {
   |     --------------- defined here
```

当我看到报错信息时，明白了接收者 —— 在本例中为 `self` —— 只是一个普通的参数。

我可以自己传入这个参数：

```rust
    loop {
        App::update(app);
        App::render(app);

        if !app.running {
            break;
        }
        sleep(Duration::from_secs(1));
    }
```

我可以用方法调用语法，而这是我在前面想要做的：

```rust
    loop {
        app.update();
        app.render();

        if !app.running {
            break;
        }
        sleep(Duration::from_secs(1));
    }
```

基于我的 Java 和 C# 的经验，我以为现在已经可以正常运行了，但是并没有。

```bash
$ cargo check --quiet                                                                                         
error[E0382]: use of moved value: `app`
  --> src/main.rs:14:9
   |                                                   
4  |     let app = App {
   |         --- move occurs because `app` has type `App`, which does not implement the `Copy` trait
...
13 |         app.update();
   |         --- value moved here
14 |         app.render();
   |         ^^^ value used here after move

(cut)
```

然而，根据我的 C++ 经验，我对整个事情感到不安。Java 和 C# 对类有 "引用语义"（而我们真的在尽力实现一些像类的东西），所以在这些语言中类似的代码绝对应该是可行的。

但在 C++ 中，情况并非如此。在 C++ 中，我们知道，如果我们只是试图传递一个类的实例，它要么被 "移动"，要么被 "复制"，这取决于我们的选择。

通过查找一些关于方法和接收者的更多文档，我终于发现了这一点。

```rust
impl App {
    fn update(self) {
        // ...
    }
}
```

简记如下：

```rust
impl App {
    fn update(self: App) {
        // ...
    }
}
```

这再次强化了这样一个概念：尽管 `self`很特别，虽然它允许使用 "方法调用语法"，但实际上它只是一个普通的参数。

再看一下编译器的错误，我可以看到，当有一个 `self::App` 类型的参数时，做的事与 C++ 中一样的：要么移动（就像这里一样），要么被复制 —— 如果我们的类型 "实现了 `Copy` 特征"。

总之，不管是用我的 C# 或 Java 经验 —— 对引用语义与复制语义的理解有一定基础，还是用我的 C++ 直觉来得出同样的结论，我最终意识到我并不想得到一个 `App`，而是想得到一个 `App` 的引用。

代码如下：

```rust
impl App {
    fn update(self: &App) {
        // ...
    }
}
```

而根据我刚刚学到的知识，可以简记如下：

```rust
impl App {
    fn update(&self) {
        // ...
    }
}
```

因此，我按如上方法修改了`fn update` 和 `fn render` 。

```bash
$ cargo check --quiet
error[E0594]: cannot assign to `self.ticks_left` which is behind a `&` reference
  --> src/main.rs:31:9
   |
30 |     fn update(&self) {
   |               ----- help: consider changing this to be a mutable reference: `&mut self`
31 |         self.ticks_left -= 1;
   |         ^^^^^^^^^^^^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be written

(cut)
```

错误数量陡然减少 —— 我们又回到了 "Rust编译器给了我很好的建议 "的场景，所以我终于开始找回了一些信心，只需要按照指示去做：

```rust
impl App {
    fn update(&mut self) {
        self.ticks_left -= 1;
        if self.ticks_left == 0 {
            self.running = false;
        }
    }

    fn render(&self) {
        if self.ticks_left > 0 {
            println!("You turn the crank...");
        } else {
            println!("Jack POPS OUT OF THE BOX");
        }
    }
}
```

根据我之前的经验，这次应该可以运行了。Rust 也有一个`常量性`的概念，它只是选出而不是选进。如果这是 C++，`render` 会使用常量引用，而 `update` 会使用常规引用。

只是在 Rust 中，"常规"意味着不可变。

**表示不可变应该用 `immutable` 还是 `const`？**

immutable。"常量性 "是一个复杂的、容易出错的概念(关键词表示什么？是否允许内部修改？)。不可变性则不然。

**有其他默认是不可变的东西吗？**

我不知道，我*刚入门 Rust*，为什么要问*我*呢？我们还是看下如何编译通过和运行吧：

```bash
$ cargo check --quiet
error[E0596]: cannot borrow `app` as mutable, as it is not declared as mutable
  --> src/main.rs:13:9
   |
4  |     let app = App {
   |         --- help: consider changing this to be mutable: `mut app`
...
13 |         app.update();
   |         ^^^ cannot borrow as mutable
```

**还有其他东西默认*是*不可变的。**

还有 “this” —— `app` 变量。

**让我查阅一下，不可变的东西并不是“变量”。**

不是吗？

**并，它是一种“绑定”。**

**我猜“变量”这个名字意味着它可以改变，而绑定默认是不可变的。**

绑定是什么？

**根据我的理解，它只是你给一个值起的名字。**

**这里说一个典型的错误是把 "变量 "看作一个单一的东西，比如 "x 是一个整数"，但它实际上是两件事：在某个地方有一个 "整数值"（在堆栈中，在堆上，或者在常数所在的地方），还有你给它的名字 —— 绑定。**

绑定*只是*个名字，对吗？

**它可能是可变的也可能是不可变的。**

我觉得我理解了。当我写下面的代码时：

```rust
let app = App { /* ... */ };
```

在内存的*某个地方*有一个 `App` 类型的值。由于所有的内存都是可读可写的，它肯定是可变的（改变、修改、写入、更改、更新）。

但是我把它绑定到一个名字上，`app` —— 因为我没有*显式地*说这是一个可变的绑定，那么这意味着我永远不能通过 `app` 来修改它。

**我想说这*听起来*正确，但我现在还有很多疑问，一会儿再问。我还有*很多*关于 Rust 的知识要学习。**

**关于你的 app，编译器有对错误的建议吗？**

是的。

```rust
fn main() {
    // new: `mut` keyword
    let mut app = App {
        title: "Jack in the box".to_string(),
        ticks_left: 4,
        running: true,
    };

    // etc.
}
```

我认为它现在应该可以运行了。

<iframe src="https://asciinema.org/a/352973/iframe?" id="asciicast-iframe-352973" name="asciicast-iframe-352973" scrolling="no" allowfullscreen="true" style="overflow: hidden; margin: 0px; border: 0px; display: inline-block; width: 100%; float: none; visibility: visible; height: 429px;"></iframe>

## 现在来看框架

现在我已经弄明白了这些，我不想每次做另一个图形应用程序时都要重复这些。我希望有一些可重复使用的代码。

我的意思不是说 "只是通过我的 USB 输入进行复制粘贴"，我的意思是 —— 有一个实际的框架，一个  "crate"，就像他们说的那样，也许我甚至会发布它，这样我只要在我的 `Cargo.toml`上加一行就可以开始做一个新的项目。

所以，基于我之前在 C#、Java、C 或 C++方面的经验，我按照我的计划行事。我们没有类，但我们有特征 —— 乍一看，感觉有点像 Java 接口。

那么，"这个项目的逻辑" 的接口应该是什么样子的呢？

让我们再看一下 `App`。

```rust
impl App {
    fn update(&mut self) {
        // ...
    }

    fn render(&self) {
        // ...
    }
}
```

好的，所以我们需要一个 `update` 方法，它需要能够修改 `self`，还有一个 `render` 方法，它不需要修改 `self`，但仍然需要读取 `self`—— 因为如果它不能读取，它如何渲染 app 的状态？

我们可以用这个来做一个特性。

```rust
trait Client {
    fn update(&mut self);
    fn render(&self);
}
```

然后我们可以从（`MyClient`）项目特有逻辑中分离出（`App`）通用的模板：

```rutst
struct App {
    title: String,
    running: bool,
}

struct MyClient {
    ticks_left: usize,
}
```

我们需要为 `MyClient` 结构体实现 `Client` 特征：

```rust
impl Client for MyClient {
    fn update(&mut self) {
        self.ticks_left -= 1;
        if self.ticks_left == 0 {
            self.running = false;
        }
    }

    fn render(&self) {
        if self.ticks_left > 0 {
            println!("You turn the crank...");
        } else {
            println!("Jack POPS OUT OF THE BOX");
        }
    }
}
```

但是 `running` 不是 `MyClient` 的成员 —— 它是 `App` 的成员。

所以我猜 `Client` 也需要能访问 `App`。

不要紧，我可以使它有 `App` 的可变引用 —— 就像有一个 `MyClient` 的可变引用那样。

我需要修改 `trait` 和 `impl` 块两个地方：

```rust
trait Client {
    // new: `app: &mut App`
    fn update(&mut self, app: &mut App);
    fn render(&self);
}

impl Client for MyClient {
    // changed here, too:
    fn update(&mut self, app: &mut App) {
        self.ticks_left -= 1;
        if self.ticks_left == 0 {
            // and now we can change `running`
            app.running = false;
        }
    }
    fn render(&self) {
        // etc.
    }
}
```

最后，我可以把 `update` 和 `render` 从 `impl App` 块移除 —— 实际上，我认为只需要一个 `run` 方法，方法内部有一个 loop 包含所有的逻辑。

```rust
impl App {
    fn run(&mut self) {
        println!("=== You are now playing {} ===", self.title);

        loop {
            self.client.update(self);
            self.client.render();

            if !self.running {
                break;
            }
            sleep(Duration::from_secs(1));
        }
    }
}
```

当然，`App` 没有 `client` 成员。在我的新 `main` 方法中，我在哪里可以指定我想让 `MyClient` 运行呢？

```rust
fn main() {
    let mut app = App {
        title: "Jack in the box".to_string(),
        running: true,
    };

    app.run();
}
```

`Client::update` 方法看起来如下：

```rust
trait Client {
    fn update(&mut self, app: &mut App);
}
```

简记如下：

```rust
trait Client {
    fn update(self: &mut Self, app: &mut App);
}
```

在这个例子中 `Self` 表示 `Client`（因为现在是在 `trait Client` 块中）：

```rust
trait Client {
    fn update(self: &mut Client, app: &mut App);
}
```

很明显我们需要一个 `&mut Client`

```rust
struct App {
    title: String,
    running: bool,
    client: &mut Client,
}
```

在 `main` 函数中，我可以这么做：

```rust
fn main() {
    let mut client = MyClient { ticks_left: 4 };

    let mut app = App {
        title: "Jack in the box".to_string(),
        running: true,
        client: &mut client,
    };

    app.run();
}
```

但这编译不过。

编译器告诉的第一件事是，我不能使用 `&mut Client`。

`Client` 是个特征 —— 它的具体类型可以使任何东西！在语言发觉的某个阶段，人们决定为了*清楚地*表达意思，应该把它定义为 `dyn Client`。


又引入了第二个错误：

```rust
 $ cargo check --quiet
 error[E0106]: missing lifetime specifier
   --> src/main.rs:18:13
    |
 18 |     client: &mut dyn Client,
    |             ^ expected named lifetime parameter
    |
 help: consider introducing a named lifetime parameter
    |
 15 | struct App<'a> {
 16 |     title: String,
 17 |     running: bool,
 18 |     client: &'a mut dyn Client,
    |
 
 error[E0228]: the lifetime bound for this object type cannot be deduced from context; please supply an explicit bound
   --> src/main.rs:18:18
    |
 18 |     client: &mut dyn Client,
    |                  ^^^^^^^^^^
```

由于我是 Rust 初学者，因此我不懂这是什么意思。

我推测，可能是生命周期的边界？

下面的所有的上下文：

```rust
 fn main() {
     let mut client = MyClient { ticks_left: 4 };
 
     let mut app = App {
         title: "Jack in the box".to_string(),
         running: true,
         client: &mut client,
     };
 
     app.run();
 }
```

看到了吗？`client` 绑定引入的值的生命周期直到 `main` 结束。

它*肯定*比 `app` 的生命周期长。这不就是你需要推断 `App::client` 生命周期的所有上下文吗？

另外 —— `App` 包含其他字段，对吗？它有一个 `bool`，它甚至有一个 `String`，`String` 不仅仅是一个原始类型（整数，等等）。而编译器并没有报关于它们的错误？`&mut dyn Client` 跟它们有什么不同呢？

当然，当然，编译器告诉了我如何解决这个问题。它说如果我这样做：

```rust
 struct App<'a> {
     title: String,
     running: bool,
     client: &'a mut dyn Client,
 }
```

问题就可以解决了。

```rust
 $ cargo check --quiet
 error[E0726]: implicit elided lifetime not allowed here
   --> src/main.rs:47:6
    |
 47 | impl App {
    |      ^^^- help: indicate the anonymous lifetime: `<'_>`
 impl App<'_> {
     fn run(&mut self) {
         // etc.
     }
 }
```

现在一切正常了。

等下，还没有：

```rust
 $ cargo check --quiet
 error[E0499]: cannot borrow `*self.client` as mutable more than once at a time
   --> src/main.rs:52:13
    |
 52 |             self.client.update(self);
    |             ^^^^^^^^^^^^------^----^
    |             |           |      |
    |             |           |      first mutable borrow occurs here
    |             |           first borrow later used by call
    |             second mutable borrow occurs here
 
 error[E0499]: cannot borrow `*self` as mutable more than once at a time
   --> src/main.rs:52:32
    |
 52 |             self.client.update(self);
    |             ----------- ------ ^^^^ second mutable borrow occurs here
    |             |           |
    |             |           first borrow later used by call
    |             first mutable borrow occurs here
```

好的。好的。那里有很多事情要做。也许我*不能*借鉴我以前在 Java、C#、C 或 C++中的经验。

也许 Rust 的引用真的很复杂。

那么指针呢？C 和 C++ 有指针。C#... 嗯，很复杂。Java 没有指针，但由于某种原因，它仍然有 `NullPointerException`。

我们就不能使用指针吗？我们就不能让借用检查器一次关闭吗？

让我们试一试吧。

```rust
 // no lifetime parameter
 struct App {
     title: String,
     running: bool,
     // raw pointer
     client: *mut dyn Client,
 }
 
 struct MyClient {
     ticks_left: usize,
 }
 
 trait Client {
     // now takes a raw pointer to app
     fn update(&mut self, app: *mut App);
     fn render(&self);
 }
 
 impl Client for MyClient {
     fn update(&mut self, app: *mut App) {
         self.ticks_left -= 1;
         if self.ticks_left == 0 {
             // this is fine, probably
             unsafe {
                 (*app).running = false;
             }
         }
     }
 
     fn render(&self) {
         if self.ticks_left > 0 {
             println!("You turn the crank...");
         } else {
             println!("Jack POPS OUT OF THE BOX");
         }
     }
 }
 
 impl App {
     fn run(&mut self) {
         println!("=== You are now playing {} ===", self.title);
 
         loop {
             // this converts a reference to a raw pointer
             let app = self as *mut _;
             // this converts a raw pointer to a reference
             let client = unsafe { self.client.as_mut().unwrap() };
             // ..which we need because the receiver is a reference
             client.update(app);
             client.render();
 
             if !self.running {
                 break;
             }
             sleep(Duration::from_secs(1));
         }
     }
 }
```

现在编译通过且能运行了。我现在想要使用这个框架构建其他的图形化应用程序。它有所有需要的东西：命令行窗口和事件循环。

**但是，你是不是把 Rust 所有的安全保证都丢弃了？**

仅仅是因为我用了一些 `unsafe` 块？

**是的。**

好吧，如果他们想让我从 Rust 的安全保证中受益，也许他们不应该把它们弄得如此混乱。也许我不应该为了理解编译器为什么要提出某种建议而去上一门该死的大学课程。

**但是**

不，不，我知道，我太夸张了。但我还是很沮丧，因为编译器会告诉我 "添加一个生命周期参数"或其他什么东西，而显然我的代码不需要它就能完全正常工作！我不知道该怎么办。

谁需要引用？****（译注：和谐社会，文明翻译）的引用！****的 borrowck！

好吧好吧，我会冷静下来的。我并不是说 Rust 是一种糟糕的语言 —— 我喜欢其中的一些语法。而且我不用考虑头文件......还有一个整洁的小的包管理器和构建系统工具，它很整洁。我可能会继续使用它。只是......用指针，我再想想。

**在你开始之前：我已经阅读了大量的资料，我想我可以稍微为你解释一下。**

**你愿意听我为你解释吗？不会花费太长时间。**

时间不会太长？

**我保证。**

## 现在进入小熊课堂

**首先，我们需要明确一点，Rust \*不\* 像 C#、Java、C 或 C++。**

我不知道它有指针、引用和其他东西。它甚至还有*花括号*。

**是的，但不是。它是不同的。它从根本上说是不同的。它允许你编写做同样事情的软件，但你必须以不同的方式思考它 —— 因为 Rust 优先考虑的东西与其他语言不同。**

**而你必须学会如何以不同的方式思考。**

我明白，但是网上的资料不会教我*如何*以不同的方式思考。

**是的 —— 这是很难教的。因为要避免很多习惯。新语言有很多新的概念。`trait` 不适合实现 `class`。**

**生命周期不只是一个 "选入的安全功能"，它不是一个事后的想法，它是整个语言的基础。**

那么，你认为我第一个项目*不*应该构建成一个图形化的 app？

**时间会证明一切。挑选你感兴趣的项目是件好事，但即使你以前用其他语言做过这种项目—— 很多次 —— 也不意味着用 Rust 做起来一定很容易。**

那么，“Rust 优先考虑的东西与其他语言不同” 到底是什么意思？

**Rust 很注重“内存安全”。**

是的，所有我要继续学下去。

**例如，在你的程序中使用指针也不是内存安全的。**

**一点也不安全。**

但是它*能正常运行*。我查看了代码很多次，但是看不出哪里有问题。

**它确实能正常运行。**

**但如果你修改某些东西呢？**

我为什么要修改某些东西。

**如果其他人用了你的框架，然后改了某些东西：**

```rust
 fn main() {
     let mut app = App {
         title: "Jack, now outside the box".to_string(),
         running: true,
         client: /* ??? */,
     };
 
     app.run();
 }
```

**他们可能会有疑问？“client？client 是什么？“由于他们之前可能写过 C#、Java、C 或 C++，他们可能会”哦，我可以传入 null，问题不大“。**

然而 Rust 中没有 `null` 关键字。

但是 Rust 中有 `std::ptr::null_mut()`。

```rust
 fn main() {
     let mut app = App {
         title: "Jack, now outside the box".to_string(),
         running: true,
         client: std::ptr::null_mut() as *mut MyClient,
     };
 
     app.run();
 }
```

等等，他们从哪里获取 `MyClient`？

**在这个假设的场景下，也许你的 crate 有一个示例 client，而这个示例 client 可能会避免引起解决 "需要类型注释"的错误。**

假设代码如上，会发生什么事？

**编译通过！**

**当运行代码时，输出如下：**

```rust
 $ cargo run --quiet
 === You are now playing Jack, now outside the box ===
 thread 'main' panicked at 'called `Option::unwrap()` on a `None` value', src/main.rs:65:35
 note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

很明显运行部了 —— 他们没有正确使用你的框架。

问题不大 —— 他们可以用 `RUST_BACKTRACE=1` 参数运行，然后查看问题在哪里。

```rust
 $ RUST_BACKTRACE=1 cargo run --quiet
 === You are now playing Jack, now outside the box ===
 thread 'main' panicked at 'called `Option::unwrap()` on a `None` value', src/main.rs:65:35
 stack backtrace:
 (cut)
   13: core::panicking::panic
              at src/libcore/panicking.rs:56
   14: core::option::Option<T>::unwrap
              at /home/amos/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/src/libcore/macros/mod.rs:10
   15: some_app::App::run
              at src/main.rs:65
   16: some_app::main
              at src/main.rs:10
 (cut)
 note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

然后他们解决了遇到的问题，能正常运行了。

是的。这不是最糟糕的场景。

如果你没有使用 `as_mut()` 辅助函数，会导致代码中对空指针进行解引用，这才是最糟糕的：

```rust
 impl App {
     fn run(&mut self) {
         println!("=== You are now playing {} ===", self.title);
 
         loop {
             let app = self as *mut _;
 
             // before:
             // let client = unsafe { self.client.as_mut().unwrap() };
 
             // after:
             let client: &mut dyn Client = unsafe { std::mem::transmute(self.client) };
 
             client.update(app);
             client.render();
 
             if !self.running {
                 break;
             }
             sleep(Duration::from_secs(1));
         }
     }
 }
```

我其实并没有真正看出区别来，我来试一下：

```rust
 $ RUST_BACKTRACE=1 cargo run --quiet
 === You are now playing Jack, now outside the box ===
 [1]    1082353 segmentation fault (core dumped)  RUST_BACKTRACE=1 cargo run --quiet
```

**现在你遇到了一个段错误。**

是的，段错误 —— 类似 C 或 C++ 中的段错误。C/C++ 的段错误并不难解决 —— 你只需用调试器获取堆栈信息，然后找到出现空指针的地方，马上就可以正常运行你的 app 了。

**是的，一会儿工夫就可以解决了。**

**但是，这仍不是最糟糕的场景。**

得了吧，每次都要考虑最糟糕的场景。你是不是杞人忧天！

**是的，我确实考虑了很多，但这不是重点。**

**最糟糕的场景不是空指针 —— 而是悬垂指针。**

```rust
 fn main() {
     let client_ptr = {
         let mut client = MyClient { ticks_left: 4 };
         &mut client as _
     };
 
     let mut app = App {
         title: "Jack, now outside the box".to_string(),
         running: true,
         client: client_ptr,
     };
 
     app.run();
 }
```

这段代码会发生什么？你能给我讲讲吗？

等等，不，我想我理解了 —— 好的，所以你声明了一个新的绑定 `client_ptr`，而右边的表达式是......一个代码块？

**是的，在 Rust 中代码块是表达式。**

代码块创建了一个新的 `MyClient`，它绑定到了 `client` —— 一个可变的绑定。然后是 `as`，我觉得在之前也见过 `as`，一个转换操作，但是你要转换成 `_`？

**是的！我转换的目标就是：”rustc，请自己推断“。**

**由于我在下面的 `App {...}`初始化中也用了它，而且左手边是 `&mut MyClient`，因此 rustc 可以推断出要转换成 `\*mut MyClient` 类型。**

由于 `MyClient` 的值是在一个代码块中创建的......它有自己的作用域......它会在 `App` 释放之前被释放，而且指针将成为悬垂指针。

**现在你理解了！**

但是 —— 我仍没有理解问题的根源在哪里。这也会导致一个段错误，我之前也说过，C/C++ 开发者很熟悉段错误。

我不认为你的论据像你认为的那样有说服力。

**是吗？**

**你为什么不试着运行它呢？**

```bash
 $ RUST_BACKTRACE=1 cargo run --quiet
 === You are now playing Jack, now outside the box ===
 You turn the crank...
 You turn the crank...
 You turn the crank...
 Jack POPS OUT OF THE BOX
```

**能正常运行了。**

**现在使用 release 模式运行。**

```bash
 $ RUST_BACKTRACE=1 cargo run --quiet --release
 === You are now playing Jack, now outside the box ===
 You turn the crank...
 You turn the crank...
 You turn the crank...
 You turn the crank...
```

继续等等看

```bash
 You turn the crank...
 You turn the crank...
 You turn the crank...
 You turn the crank...
 You turn the crank...
```

但它为什么停不下来了？

```bash
 You turn the crank...
 You turn the crank...
 You turn the crank...
 You turn the crank...
 You turn the crank...
```

为什么一直不停？

```bash
 You turn the crank...
 You turn the crank...
 You turn the crank...
 You turn the crank...
 You turn the crank...
```

我需要让它停下来！

**好吧，你只需要按下 `Ctrl + C`。**

它终于停下了。到底发生了什么？

**也没什么。只是最糟糕的场景：你看不见的错误。**

是的，似乎是这样！对不起，我大喊大叫了！经历过刚才停不下来的状况，我还惊魂未定！

问题是...没看到所有的失败。有些失败没有显示出来，这才是最糟糕的。

更糟糕的是，程序在 debug 模式和 release 模式的构建中表现不一致。即使你写测试代码，你也看不到失败的原因，因为测试代码默认是在 debug 模式中构建的。

这就不妙了！

**很不妙。现在的场景就是在一个黑盒中摸索，而你并知道要找的答案到底在什么位置。**

**想象以下，如果你现在写的是一个操作系统、web 浏览器或维持某个人生命的设备，会有什么后果？**

那会相当可怕！

是的。

这些错误并不只是理论上存在的错误。微软在 2019 年说，[70% 的安全漏洞是内存安全问题](https://www.zdnet.com/article/microsoft-70-percent-of-all-security-bugs-are-memory-safety-issues/)。

而且他们[不是唯一这样说的人](https://www.zdnet.com/article/chrome-70-of-all-security-bugs-are-memory-safety-issues/)。

好吧，我想这是一个相当重要的问题，而使用 Rust 会对内存安全有帮助吗？

**答案是肯定的。帮助不止一点点。**

Rust 是怎么做的？

**让我们从一个简单的例子开始。**

**下面是一个能打印一个 `i64`（有符号的 64 位整型）的值的函数。**

```rust
 fn show(x: i64) {
     println!("x = {}", x);
 }
```

**函数很厉害！但它只能打印 `i64` 类型。**

是的。如果我们想打印其他的东西，要么创建其他的函数，要么把 `show` 泛型化。

**就像下面这样**

```rust
 fn show<T>(x: T) {
     println!("x = {}", x);
 }
```

是的，泛型。我之所以知道泛型，是因为我之前写过 C++、C# 或 Java。

**它可能不能正常运行，因为我们并没有说 `T` 一定是能打印的。**

能被打印？我们需要怎么声明？

**我们加一个约束：**

```rust
 use std::fmt::Display;
 
 fn show<T: Display>(x: T) {
     println!("x = {}", x);
 }
```

或者长版的可以这么写：

```rust
 use std::fmt::Display;
 
 fn show<T>(x: T)
 where
     T: Display,
 {
     println!("x = {}", x);
 }
```

我可以在标准库文档中查到 `Display`，`Display` 是一个特征？

**是的。`Display` 是一个表示每个类型某种属性的特征：被展示的能力。**

**你觉得这两个值有什么区别：**

```rust
 fn main() {
     {
         let one = MyClient { ticks_left: 4 };
     }
 
     let two = MyClient { ticks_left: 4 };
 
     // some other code
 
     println!("Thanks for playing!");
 }
```

`one` 是在它自己的作用域内，所有在初始化后它马上被释放了。而 `two` 在整个 `main` 函数能都可用。

**完全正确！他们的生命周期不同。**

**你能告诉我，代码中的生命周期在什么地方吗？**

我没有看到它们。

你看不到它们，因为编译器已经推断出了它们。它们确实存在，而且很重要，它们决定了你能用 `one` 和 `two` 做什么，但你在代码中看不到它们，你也不能给它们命名。

我明白了。

**由于 `one` 和 `two` 的声明周期不一样，因此它们的类型不一样。**

是吗？

**如果你写这么一个函数：**

```rust
 fn show_ticks(mc: &MyClient) {
     println!("{} ticks left", mc.ticks_left);
 }
```

**那么你就不能写泛型函数了。**

什么？上面的函数不是泛型 —— 尖括号在哪里？

如果你愿意，你可以加上：

```rust
 fn show_ticks<'wee>(mc: &'wee MyClient) {
     println!("{} ticks left", mc.ticks_left);
 }
```

所以，另一个版本只是简写版？

**是的！**

这个函数是超越了引用的生命周期的泛型。

**技术上来说，是超越了引用指向的值的生命周期。**

**你也可以不把函数泛型化，输入参数指定一个明确的声明周期，比如 `'static` 表示入参的一直不会被释放。**

```rust
 fn show_ticks(mc: &'static MyClient) {
     println!("{} ticks left", mc.ticks_left);
 }
```

我们还能像下面那样使用它吗？

```rust
 fn main() {
     let one = MyClient { ticks_left: 4 };
     show_ticks(&one);
 }
```

**你可以用，但不是想上面那么用：**

```bash
 cargo check --quiet
 error[E0597]: `one` does not live long enough
  --> src/main.rs:5:16
   |
 5 |     show_ticks(&one);
   |     -----------^^^^-
   |     |          |
   |     |          borrowed value does not live long enough
   |     argument requires that `one` is borrowed for `'static`
 6 | }
   | - `one` dropped here while still borrowed
```

**只有当你有个一直不会被释放的 `MyClient` 类型的值时，才能这么用。而在 `main` 结束的地方，`one` 会被释放，因此这个例子不能正常运行。而你并不知道怎么获取一个一直不会被释放的 `MyClient` 类型的值。**

我可以！

```rust
 static FOREVER_CLIENT: MyClient = MyClient { ticks_left: 4 };
 
 fn main() {
     show_ticks(&FOREVER_CLIENT);
 }
```

**是的，它能正常运行 —— 这就是前面那个例子中生命周期命名为 `'static` 的原因，静态变量（在可执行文件中的[数据段](https://fasterthanli.me/series/making-our-own-executable-packer/part-2)）一直不会被释放。**

**而我更建议这么写：**

```rust
 fn main() {
     let client = MyClient { ticks_left: 4 };
 
     let client_ref = Box::leak(Box::new(client));
     show_ticks(client_ref);
 }
```

我看不懂。

**好吧 —— 当函数的某些参数是引用时，函数会成为泛型。**

**因为”引用指向的值“的生命周期是该引用的\*类型\*的一部分。我们节省了大量时间，因为编译器几乎都能推断出来。**

我刚要问 —— 如果我们回到刚才有编译器的建议的版本：

```rust
 struct App<'a> {
     title: String,
     running: bool,
     client: &'a mut dyn Client,
 }
 
 impl App<'_> {
     fn run(&mut self) {
         println!("=== You are now playing {} ===", self.title);
 
         loop {
             let app = self as *mut _;
             self.client.update(app);
             self.client.render();
 
             if !self.running {
                 break;
             }
             sleep(Duration::from_secs(1));
         }
     }
 }
```

它需要明确的生命周期标注。尽管 —— 我想强调这一点 —— 尽管它确切地知道 `app` 和 `client` 的生命周期是什么：

```rust
 fn main() {
     let mut client = MyClient { ticks_left: 4 };
 
     let mut app = App {
         title: "Jack in the box".to_string(),
         running: true,
         client: &mut client,
     };
 
     app.run();
 }
```

**是的！请考虑下这个场景：生命周期需要遍及\*整个\*程序，而不仅仅是 `main` 函数的范围。**

编译器仍然知道我是怎么使用的。无论我在哪里使用，编译器都会知道。编译器不能处理吗？

**是也不是！**

**你使用生命周期标注（和边界/约束）的原因与你使用类型的原因相同。如果 `add(x, y)` 只知道如何添加 `i64` 值，那么它只能支持 `i64` 类型的参数。**

**如果一个函数需要能一直访问一个值，那么它需要一个 `&'static T` 的参数。**

你可以为我举出更多的例子吗？

**当然！**

```bash
 $ cargo new some-examples
      Created binary (application) `some-examples` package
 struct Logger {}
 
 static mut GLOBAL_LOGGER: Option<&'static Logger> = None;
 
 fn set_logger(logger: &'static Logger) {
     unsafe {
         GLOBAL_LOGGER = Some(logger);
     }
 }
 
 fn main() {
     let logger = Logger {};
     set_logger(Box::leak(Box::new(logger)));
 }
 
```

对，一个 logger。我理解你想要一直访问那个值的原因。

你为什么用 `unsafe`？

**可变的静态变量是很危险的。**

接下来呢？

```rust
 use std::time::SystemTime;
 
 fn log_message(timestamp: SystemTime, message: &str) {
     println!("[{:?}] {}", timestamp, message);
 }
 
 fn main() {
     log_message(SystemTime::now(), "starting up...");
     log_message(SystemTime::now(), "shutting down...");
 }
```

这次没有生命周期标注了 —— 我猜 `message` 只在 `log_message` 的调用范围内有效。

你不必一直维护它，因为你只是需要读取它，然后马上就不访问了。

现在需要一个好的例子，来展示生命周期超过函数调用的范围，但是不遍及整个程序运行周期。

**这就是有意思的地方！**

```rust
 #[derive(Default)]
 struct Journal<'a> {
     messages: Vec<&'a str>,
 }
 
 impl Journal<'_> {
     fn log(&mut self, message: &str) {
         self.messages.push(message);
     }
 }
 
 fn main() {
     let mut journal: Journal = Default::default();
     journal.log("Tis a bright morning");
     journal.log("The wind is howling");
 }
```

等等，`Default`?`#derive`?

```rust
 struct Journal<'a> {
     messages: Vec<&'a str>,
 }
 
 impl Journal<'_> {
     fn new() -> Self {
         Journal {
             messages: Vec::new(),
         }
     }
 
     fn log(&mut self, message: &str) {
         self.messages.push(message);
     }
 }
 
 fn main() {
     let mut journal = Journal::new();
     journal.log("Tis a bright morning");
     journal.log("The wind is howling");
 }
```

我明白了。 `message` 的生命周期应该与 `journal` 本身相同。

我自己来试一下。编译不过！

```bash
 $ cargo check -q
 error[E0312]: lifetime of reference outlives lifetime of borrowed content...
   --> src/main.rs:13:28
    |
 13 |         self.messages.push(message);
    |                            ^^^^^^^
    |
 note: ...the reference is valid for the lifetime `'_` as defined on the impl at 5:14...
   --> src/main.rs:5:14
    |
 5  | impl Journal<'_> {
    |              ^^
 note: ...but the borrowed content is only valid for the anonymous lifetime #2 defined on the method body at 12:5
   --> src/main.rs:12:5
    |
 12 | /     fn log(&mut self, message: &str) {
 13 | |         self.messages.push(message);
 14 | |     }
    | |_____^
```

我们没有正确传递约束？

是的！

如果我们不用简写的方式，看起来就容易了：

```rust
 impl<'journal> Journal<'journal> {
     fn new() -> Self {
         Journal {
             messages: Vec::new(),
         }
     }
 
     fn log<'call>(&'call mut self, message: &'call str) {
         self.messages.push(message);
     }
 }
```

为什么我看到到处都是 `'a`？

**当你只有一个生命周期时，它看起来像是数学中的 `x`。**

运行上面的代码，出现错误：

```bash
 $ cargo check -q
 error[E0312]: lifetime of reference outlives lifetime of borrowed content...
   --> src/main.rs:13:28
    |
 13 |         self.messages.push(message);
    |                            ^^^^^^^
    |
 note: ...the reference is valid for the lifetime `'journal` as defined on the impl at 5:6...
   --> src/main.rs:5:6
    |
 5  | impl<'journal> Journal<'journal> {
    |      ^^^^^^^^
 note: ...but the borrowed content is only valid for the lifetime `'call` as defined on the method body at 12:12
   --> src/main.rs:12:12
    |
 12 |     fn log<'call>(&'call mut self, message: &'call str) {
    |            ^^^^^
```

看起来比之前好多了。通过使用简写的方式，我们不小心只在调用期间借用了它，而实际上我们想在 `Journal` 的整个生命周期内借用它。

**是的，因为我们实际是在 `Journal` 中保存的。**

因此，我们真正希望的是这样：

```rust
 impl<'journal> Journal<'journal> {
     // omitted: `fn new`
 
     fn log<'call>(&'call mut self, message: &'journal str) {
         self.messages.push(message);
     }
 }
```

如果我想用简写的方式，应该是这样：

```rust
 impl<'journal> Journal<'journal> {
     // cut
 
     fn log(&mut self, message: &'journal str) {
         self.messages.push(message);
     }
 }
```

我可以用 `‘_` 进一步简化吗？

**不能。就像 `x as _` 会转换成`自动推测`类型， `'_` 会转成`我不知道什么类型`的生命周期。如果你用 `impl Journal<'_>` 和 `&'_ str`，那么它默认为转成 `&'call`，而不是 `&'journal`。**

我理解了。

Rust 会在整个程序运行期间检测生命周期，对吗？

**是的。**

但我有一个想法 —— 如果我希望从某个函数返回一个 `Journal` 呢？

**分情况讨论。**

**如果你的代码是这样：**

```rust
 fn main() {
     let journal = get_journal();
 }
 
 fn get_journal() -> Journal {
     let mut journal = Journal::new();
     journal.log("Tis a bright morning");
     journal.log("The wind is howling");
     journal
 }
 
```

**会有编译错误。**

我来试一下：

```bash
 $ cargo check --quiet
 error[E0106]: missing lifetime specifier
   --> src/main.rs:21:21
    |
 21 | fn get_journal() -> Journal {
    |                     ^^^^^^^ expected named lifetime parameter
    |
    = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
 help: consider using the `'static` lifetime
    |
 21 | fn get_journal() -> Journal<'static> {
    |                     ^^^^^^^^^^^^^^^^
```

现在我不只要写 `Journal`，还要写 `Journal<something>`。

如果我写成这样：

```rust
 fn get_journal<'a>() -> Journal<'a> {
     let mut journal = Journal::new();
     journal.log("Tis a bright morning");
     journal.log("The wind is howling");
     journal
 }
```

看起来可以了！发生了什么？

**这个结果的原因是，你所有的字符串常量都有个 `static` 的生命周期。**

所以导致 `Journal` 也有 `'static` 生命周期？

我能确认吗？

**可以：**

```rust
 fn main() {
     let journal: Journal<'static> = get_journal();
 }
```

它*只是个类型*，我可以写出来。

在什么时候它会阻止我写错误的代码？

**像下面这样：**

```rust
 fn get_journal<'a>() -> Journal<'a> {
     let s = String::from("Tis a dark night. It's also stormy.");
 
     let mut journal = Journal::new();
     journal.log(&s);
     journal
 }
```

我检查下：

```bash
 $ cargo check --quiet
 error[E0515]: cannot return value referencing local variable `s`
   --> src/main.rs:26:5
    |
 25 |     journal.log(&s);
    |                 -- `s` is borrowed here
 26 |     journal
    |     ^^^^^^^ returns a value referencing data owned by the current function
 
 error: aborting due to previous error
```

很好。我很喜欢。基于我之前的 C 和 C++ 经验，我看到了一些警告。

**在 Java 或 C# 语言中，这不是个问题，因为它们有垃圾回收机制。**

那么Rust 有*近似* C# 或 Java 中的这种东西吗？

**Rust 中有引用计数！**

```rust
 use std::sync::Arc;
 
 #[derive(Default)]
 struct Journal {
     messages: Vec<Arc<String>>,
 }
 
 impl Journal {
     fn log(&mut self, message: Arc<String>) {
         self.messages.push(message);
     }
 }
 
 fn main() {
     let _journal: Journal = get_journal();
 }
 
 fn get_journal() -> Journal {
     let s = Arc::new(String::from("Tis a dark night. It's also stormy."));
 
     let mut journal: Journal = Default::default();
     journal.log(s);
     journal
 }
```

等等，生命周期去哪儿了？

我们不需要它们了！因为有了诸如 `Arc` 和 `Rc` 的智能指针，只要我们有某个值的至少*一个*引用（一个 `Arc<T>`），值就不会被释放。

但是再等等，`&T` 和 `Arc<T>` 看起来是完全不同的语法，但是它们都是指针？

**它们都是指针。**

当我有个 `Arc<T>` 而我需要一个 `&T` 时呢？我应该继续写入参类型为 `Arc<T>` 的方法吗？

**不，不应该。**

**你可以借用（获取引用） `Arc<T>` 的内容 —— 只要 `Arc<T>` 没有被释放，它就一直可用。**

有例子吗？

**当然：**

```rust
 use std::sync::Arc;
 
 struct Event {
     message: String,
 }
 
 #[derive(Default)]
 struct Journal {
     events: Vec<Arc<Event>>,
 }
 
 impl Journal {
     fn log(&mut self, message: String) {
         self.events.push(Arc::new(Event { message }));
     }
 
     fn last_event(&self) -> Option<&Event> {
         self.events.last().map(|ev| ev.as_ref())
     }
 }
 
 fn main() {
     // muffin
 }
```

你说的是 `last_event`，对吗？即使我们维护的是 `Arc<Event>` 类型的值，我们也可以转成 `&Event`。

**是的，而且通常情况下会自动强制转换。**

**例如，下面的代码能运行：**

```rust
 impl Event {
     fn print(&self) {
         println!("Event(message={})", self.message);
     }
 }
 
 fn main() {
     let ev = Arc::new(Event {
         message: String::from("well well well."),
     });
     ev.print();
 }
```

我猜 `last_event` 可以返回一个 `Arc<Event>`，对吗？既然它*实际上是个指针*，我们不应该对它的引用计数加 1 然后返回它？

**是的，通过克隆它！我的意思是通过智能指针，而不是 `Event`：**

```rust
 impl Journal {
     fn last_event(&self) -> Option<Arc<Event>> {
         self.events.last().map(|x| Arc::clone(x))
     }
 }
```

**现在这里使用的是 `Option`，因为如果 jornal 是空的，那么就不会有 last event。**

**此外，`Option` 有个简写的方式：**

```rust
 impl Journal {
     fn last_event(&self) -> Option<Arc<Event>> {
         self.events.last().cloned()
     }
 }
```

但是等一下！这意味着我在应用程序中只需要用 `Arc<T>` 就可以避免生命周期的问题了吗？

**100% 可以！**

**但你需要这么做吗？**

”我需要这么做“是什么意思？

**我的意思是：你需要\*共享所有权\*吗？你会有 `Client` 的其他引用吗？**

我不知道。可能我不需要，可能 `App` 有 `Client` 的唯一所有权也是 OK 的。

**在这个例子中，你需要一个 `Box<T>`。**

那是什么？又一个智能指针？

**是的！它与引用的大小是一样的。**

```rust
 struct Foobar {}
 
 fn main() {
     let f = Foobar {};
     let f_ref = &f;
 
     let f_box = Box::new(Foobar {});
 
     println!("size of &T     = {}", std::mem::size_of_val(&f_ref));
     println!("size of Box<T> = {}", std::mem::size_of_val(&f_box));
 }
 $ cargo run --quiet
 size of &T     = 8
 size of Box<T> = 8
```

是的，8 个字节，因为我们是在 64 位机器上。

我只需要用一个 `Box`？我试一下：

```rust
 // back in `some-app/src/main.rs`
 
 fn main() {
     let client = MyClient { ticks_left: 4 };
 
     let mut app = App {
         title: "Jack in the box".to_string(),
         running: true,
         client: Box::new(client),
     };
 
     app.run();
 }
 
 struct App {
     title: String,
     running: bool,
     client: Box<dyn Client>,
 }
 
 impl App {
     // unchanged
 }
```

确实可以运行了。这段代码与有生命周期标注的版本有什么不同？

**如果是自动生成的代码，那么就没有区别。他们构建出的二进制文件时一样的（至少在 release 模式中是）。**

**但是关于所有权：在有生命周期标注时，`App` 在 `fn main` 中借用\*了 `Client`。**

**现在，`App` 拥有了 `Client` 的所有权 —— `Client` 的生命周期与 `App` 一样了。**

好吧，这是完全新的知识。但他们的功能是一样的，对吗？

**完全一样！**

我不需要修改其他的代码：

```rust
 impl App {
     fn run(&mut self) {
         println!("=== You are now playing {} ===", self.title);
 
         loop {
             let app = self as *mut _;
             self.client.update(app);
             self.client.render();
 
             if !self.running {
                 break;
             }
             sleep(Duration::from_secs(1));
         }
     }
 }
```

`Client::update` 的入参是 `&mut self`，`self.client` 现在是一个 `Box<dyn Client`，是因为它也会在有需要时强制把智能指针转成 `&T` 或 `&mut T`？

**它不是魔术，而是两个特征：[Deref](https://doc.rust-lang.org/stable/std/ops/trait.Deref.html) 和 [DerefMut](https://doc.rust-lang.org/stable/std/ops/trait.DerefMut.html)。**

事情看起来明朗了。但我们的代码中还有个 `*mut App`：

```rust
 let app = self as *mut _;
```

我们把它传给了 `Client::update`

```rust
 trait Client {
     // hhhhhhhhhhhhhhhhhhhhhhhere.
     fn update(&mut self, app: *mut App);
     fn render(&self);
 }
```

现在我们能避免吗？

**我不知道，试一下，看看编译器会不会报错。**

```rust
 trait Client {
     fn update(&mut self, app: &mut App);
     fn render(&self);
 }
 
 impl Client for MyClient {
     fn update(&mut self, app: &mut App) {
         self.ticks_left -= 1;
         if self.ticks_left == 0 {
             app.running = false;
         }
     }
 
     fn render(&self) {
         // omitted
     }
 }
 
 impl App {
     fn run(&mut self) {
         println!("=== You are now playing {} ===", self.title);
 
         loop {
             // remember `&mut self` is just `self: &mut Self`,
             // so `self` is just a binding of type `&mut App`,
             // which is the exact type that `Client::update` takes.
             self.client.update(self);
             self.client.render();
 
             if !self.running {
                 break;
             }
             sleep(Duration::from_secs(1));
         }
     }
 }
```

现在编译它：

```bash
 $ cargo check --quiet
 error[E0499]: cannot borrow `*self.client` as mutable more than once at a time
   --> src/main.rs:52:13
    |
 52 |             self.client.update(self);
    |             ^^^^^^^^^^^^------^----^
    |             |           |      |
    |             |           |      first mutable borrow occurs here
    |             |           first borrow later used by call
    |             second mutable borrow occurs here
 
 error[E0499]: cannot borrow `*self` as mutable more than once at a time
   --> src/main.rs:52:32
    |
 52 |             self.client.update(self);
    |             ----------- ------ ^^^^ second mutable borrow occurs here
    |             |           |
    |             |           first borrow later used by call
    |             first mutable borrow occurs here
```

看到了吗？同样的错误。

**这个时候就应该”用 Rust 的方式思考问题“。**

但这是什么意思？

**现在 `App` 有了所有的东西，对吗？**

**有 `title`，有 `running` 标志，还有 `client`。**

是的，没错。

**问题是，因为 `Client::update` 有个入参是 `&mut Client`，通过调用 `self.client.update(...)`，我们以可变的方式借用了一次 `self`。**

啊哈。

**`Client::update` 还有个入参是 `&mut App`，因此我们需要再一次以可变的方式借用自己。**

是的。

**这是不被允许的。**

看起来是出于某种原因，万万不可做的。

如果我们把它们分开处理呢？

一方面，我们会得到 client；另一方面，我们会得到 app 的状态。

”分开“是什么意思？创建一个新的结构体？

**是的！把它命名为 `AppState` 或其他名字。**

我们来试一下。

```rust
 struct App {
     client: Box<dyn Client>,
     state: AppState,
 }
 
 struct AppState {
     title: String,
     running: bool,
 }
```

**现在我们仔细看一下 `MyClient::update` 方法。**

我把它分离出来看下：

```rust
 impl Client for MyClient {
     fn update(&mut self, app: &mut App) {
         self.ticks_left -= 1;
         if self.ticks_left == 0 {
             app.running = false;
         }
     }
 }
```

**仔细考虑下。**

**`MyClient::update` 需要访问整个 `App` 吗？**

它做的唯一的事，是触发 `running` 标志。

**因此，`MyClient` 只需要一个对 `AppState` 的可变引用！**

```rust
 trait Client {
     fn update(&mut self, state: &mut AppState);
     fn render(&self);
 }
 
 impl Client for MyClient {
     fn update(&mut self, state: &mut AppState) {
         self.ticks_left -= 1;
         if self.ticks_left == 0 {
             state.running = false;
         }
     }
 
     fn render(&self) {
         // omitted
     }
 }
```

**然后你需要更新下应用程序中的其他东西...**

首先修改 `main` 函数：

```rust
 fn main() {
     let client = MyClient { ticks_left: 4 };
 
     let mut app = App {
         state: AppState {
             title: "Jack in the box".to_string(),
             running: true,
         },
         client: Box::new(client),
     };
 
     app.run();
 }
```

然后修改 `App::run`：

```rust
 impl App {
     fn run(&mut self) {
         println!("=== You are now playing {} ===", self.state.title);
 
         loop {
             self.client.update(&mut self.state);
             self.client.render();
 
             if !self.state.running {
                 break;
             }
             sleep(Duration::from_secs(1));
         }
     }
 }
```

之后运行：

```bash
 $ cargo check
     Finished dev [unoptimized + debuginfo] target(s) in 0.00s
```

我现在有点懂了。但是，我们不是仍第二次以可变的方式借用 `self` 吗？

**你是以可变的方式借用了 `self` 的不同部分。**

因此分开处理能正常运行？因为它让追踪变量的生命周期变得简单？

**非常好。很推荐。**

**实际上，在其他语言中也推荐这么做。当你写了很长时间的 Rust 后，如果你再写 C、C++，你会发现写代码的方式有些变化了。**

你对我的图形化 app 有其他的建议吗？

**实际上我有！**

**有一个建议，在尤其是像 Rust 这种语言中，你不应该写”能正常运行“但只有你了解其中隐患（比如，当做了一点修改时，会突然崩溃）的代码，而是应该像下面这样。**

**不要直接执行修改，而是考虑返回能\*反应\*是否该修改的值。**

我不太理解，你可以举一个例子吗？

当然！在你的图形化 app 中 —— 你传递的是整个 `&mut AppState`，仅仅是为了退出应用程序，对吗？

是的。

**而那是 `AppState` 需要做的唯一的事。你永远不会把 `running` 设成 `true`，你永远不会访问 `AppState` 的其他部分。**

**因此你只需要它返回一个布尔值：当它应该运行时返回 `true`，当应该退出时返回 `false`。**

我来试一下：

```rust
 trait Client {
     // returns false if the app should exit
     fn update(&mut self) -> bool;
     fn render(&self);
 }
 
 impl Client for MyClient {
     fn update(&mut self) -> bool {
         self.ticks_left -= 1;
         self.ticks_left > 0
     }
 
     fn render(&self) {
         if self.ticks_left > 0 {
             println!("You turn the crank...");
         } else {
             println!("Jack POPS OUT OF THE BOX");
         }
     }
 }
 
 struct App {
     client: Box<dyn Client>,
     state: AppState,
 }
 
 struct AppState {
     title: String,
 }
 
 impl App {
     fn run(&mut self) {
         println!("=== You are now playing {} ===", self.state.title);
 
         loop {
             let running = self.client.update();
             self.client.render();
 
             if !running {
                 break;
             }
             sleep(Duration::from_secs(1));
         }
     }
 }
```

我喜欢这样。引用减少了！更少依赖借用检查器了。我会习惯这样。

**对吗？虽然我们在清晰度上有一点损失 —— 为什么 `update()` 会返回一个 `bool` 不太好理解。我可以说出这个例子中的意思，是因为你加了一个注释来说明它的实际作用。**

我确实是出于这个原因添加了注释。

**但你没有在所有的地方加注释，对吗？`impl Client for MyClient` 里没有注释，在 `update` 被调用的 `App::run` 处也没有注释。**

是的。我猜你需要花点功夫来研究它做了什么。

**如果我告诉你，不需要这么做呢？**

不需要？

**是的！**

**Rust 有枚举，你应该使用它们。**

**你需要写一个这样的枚举：**

```rust
 enum UpdateResult {
     None,
     QuitApplication,
 }
```

这就是 `update` 要返回的类型？

来试一下：

```rust
 trait Client {
     fn update(&mut self) -> UpdateResult;
     fn render(&self);
 }
 
 impl Client for MyClient {
     fn update(&mut self) -> UpdateResult {
         self.ticks_left -= 1;
 
         if self.ticks_left == 0 {
             UpdateResult::QuitApplication
         } else {
             UpdateResult::None
         }
     }
 }
 
 impl App {
     fn run(&mut self) {
         println!("=== You are now playing {} ===", self.state.title);
 
         loop {
             let res = self.client.update();
             self.client.render();
 
             match res {
                 UpdateResult::None => {}
                 UpdateResult::QuitApplication => {
                     return;
                 }
             }
             sleep(Duration::from_secs(1));
         }
     }
 }
```

