---
> 原文链接: https://blog.logrocket.com/custom-blockchain-implementation-rust-substrate/
> 
> **翻译：[Akagi201](https://github.com/Akagi201)**
>
> 选题：[Sunface](https://github.com/sunface)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

# 使用 Rust 和 Substrate 构建自己的区块链平台

在之前的文章中，我们探讨了 [Substrate](https://substrate.io/) 区块链框架背后的核心概念。在本教程中，我们将实际动手，使用 Substrate 在 Rust 中实现一个非常基础的自定义区块链应用。

严格来说，没有必要阅读介绍性文章，但如果一些术语对你来说是陌生的或令人困惑的，那么略过 ["Substrate 区块链开发：核心概念"](https://blog.logrocket.com/substrate-blockchain-framework-core-concepts/)可能会有意义，然后再继续学习本教程。

我们将建立一个简单的基于区块链的博客应用程序的后端，其中用户可以提交博客文章，评论用户的博客文章，并在喜欢他们的内容时向其他博客文章作者发送提示。

所有这些操作都会根据操作的执行时间，让用户花费少量的货币。要了解这个概念的更多信息，你可以查看 [Substrate 的文档](https://docs.substrate.io/v3/runtime/weights-and-fees/)。

这个应用程序的基础是 [Substrate Node Template](https://github.com/substrate-developer-hub/substrate-node-template)，一个完全设置好的 Substrate 节点，可以进行定制。

在本教程中，我们将只关注应用程序的后端部分，并依靠 [Substrate Front End Template](https://github.com/substrate-developer-hub/substrate-front-end-template) 和我们的区块链的自定义功能进行交互。在未来的文章中，我们可能会扩展这个例子，并基于这个模板为其构建一个自定义的前端应用程序。

## 起步

为了设置我们的节点模板，我们将使用 [Kickstart](https://github.com/Keats/kickstart)，你可以通过执行以下命令来安装它。

```sh
cargo install kickstart
```

Kickstart 是一个简单的 CLI 工具，用于根据预先制作的模板初始化项目。幸运的是，有一个[用于 Kickstart 的 Substrate 模板](https://github.com/sacha-l/kickstart-substrate)。

现在你可以在你工作区的根目录下运行这个命令。

```sh
❯ kickstart https://github.com/sacha-l/kickstart-substrate
Tmp dir: "/var/folders/r4/qqzf2l0s5hn0y81xxwn793sm0000gp/T/"
What are you calling your node? [default: template]: blogchain
What are you calling your pallet? [default: template]: blogchain

Everything done, ready to go!
```

这将提示你输入两个个名字。只需都输入 `blogchain`，然后它将下载并设置一个本地 Substrate 节点模板，并按照你给的名字设置一个完整的 pallet。

让我们来看看 `blogchain` 项目中的文件结构：

![blogchain](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402-custom-blockchain-implementation-rust-substrate/blogchain_tree.png)

正如你所看到的，我们在这里有相当多的文件和文件夹。最重要的是，我们有我们的 `pallets/blogchain` 区域，我们将在这里做大部分的定制和实现我们所有的业务逻辑。

`node` 包含所有我们需要运行底层节点模板的代码。在运行时，所有内部和外部的 pallets 是连在一起的，所以它们可以在节点内运行。

我们将只对 `pallets/blogchain/lib.rs` 和 `runtime/src/lib.rs` 文件进行修改。

如果你在某个时候迷失了方向，或者没有成功，你可以参考这个仓库并查看提交历史，这与我们在本教程中的路径大致吻合。

首先，在 `runtime/src/lib,rs` 中，修改以下一行：

```Rust
TemplateModule: pallet_blogchain,
```

到

```Rust
Blogchain: pallet_blogchain,
```

这对于确保我们的 pallet 以后在前端显示正确的名称是很重要的。

另外，出于设置的考虑，我们将为我们的 pallet 添加 `Currency` trait，因为我们以后要把一些货币从一个用户转移到另一个用户。

在 `pallets/blogchain/lib.rs` 中，改变以下内容：

```Rust
use frame_support::pallet_prelude::*;
```

到：

```Rust
use frame_support::{pallet_prelude::*, traits::Currenty};
```

在范围内：

```Rust
#[pallet::config]
pub trait Config: frame_system::Config {
    type Event: From<Event<Self>> + IsType<<Self as frame_system::Config>::Event>;
    type Currency: Currency<Self::AccountId>; // <-- new
}
```

这个添加 `Currency` trait 到我们的 pallet. 我们也添加这个到 `runtime/lib.rs`:

```Rust
pub use frame_support::{
...
        traits::{Currency, KeyOwnerProofSystem, Randomness, StorageInfo},
...
};

impl pallet_blogchain::Config for Runtime {
        type Event = Event;
        type Currency = Balances; // <-- new
}
```

好了，设置就到此为止了。确保所有的东西都能编译，让我们继续讨论一些数据结构和状态定义。

## 数据结构和存储

在本节中，我们将实现一个带有评论和小费功能的简单博客系统。

首先让我们在 `/pallets/blogchain/lib.rs` 中为博客文章和评论添加一些数据结构：

```Rust
use frame_support::{
...
inherent::Vec,
...
};


#[derive(Clone, Encode, Decode, PartialEq, RuntimeDebug, TypeInfo)]
#[scale_info(skip_type_params(T))]
pub struct BlogPost<T: Config> {
        pub content: Vec<u8>,
        pub author: <T as frame_system::Config>::AccountId,
}

#[derive(Clone, Encode, Decode, PartialEq, RuntimeDebug, TypeInfo)]
#[scale_info(skip_type_params(T))]
pub struct BlogPostComment<T: Config> {
        pub content: Vec<u8>,
        pub blog_post_id: T::Hash,
        pub author: <T as frame_system::Config>::AccountId,
}
```

这个看起来有些复杂，所以让我们来拆开他。

我们先来看看结构体上面的宏。我们派生了几个著名的 helper trait，如 `Clone`、`PartialEq` 等，但我们也派生了 `scale_info::TypeInfo` 特性并使用 `scale_info` 宏。这些用于 Substrate 内部的序列化和反序列化，以及[在运行时检索关于类型的编译时信息](https://docs.substrate.io/rustdocs/latest/scale_info/index.html)。

这些结构本身相当简单。文章只是有一个字节类型的内容向量，而作者基本上只是系统中的一个 `AccountId`。`frame_system::Config` 本质上是底层的 pallet 配置。`frame_system` 模块为所有底层 pallet 提供功能和数据类型。这也是为什么这些结构体是 `T:Config` 的泛型，因为它们总是在一个 pallet 配置的上下文中使用。

`BlogPostComment` 有一个内容和一个作者，但它也持有 `blog_post_id`，它是一个 `T::Hash`。这是配置的散列函数的输出。我们将对所有东西使用 Substrate 的默认值，但你可以想象，如果你想将这个托盘用于不同的区块链系统，而这些系统依赖于不同的配置，例如不同的散列算法，那么这将是非常有用的。这样一来，不需要做任何必要的改变就能工作。

下一步是定义我们想在区块链上保留的状态。为此目的，我们将创建两个 `StorageMap` 实例，保存文章和评论。

```rust
/// Storage Map for BlogPosts by BlogPostId (Hash) to a BlogPost
#[pallet::storage]
#[pallet::getter(fn blog_posts)]
pub(super) type BlogPosts<T: Config> = StorageMap<_, Twox64Concat, T::Hash, BlogPost<T>>;

/// Storage Map from BlogPostId (Hash) to a list of BlogPostComments for this BlogPost
#[pallet::storage]
#[pallet::getter(fn blog_post_comments)]
pub(super) type BlogPostComments<T: Config> =
        StorageMap<_, Twox64Concat, T::Hash, Vec<BlogPostComment<T>>>;
```

通过 `pallet::storage` 宏，我们告诉系统这是一块持久性存储，而 `pallet::getter` 宏使得我们可以在以后根据给定的函数名查询。

对于博文，我们使用一个 `StorageMap`，它本质上是一个哈希字典，从 `T::Hash` 映射到 `BlogPost`，也就是从一个博文 id 到一个博文。

对于博文的评论，我们想从一个博文的 ID 映射到一个评论列表，这样我们就可以查询一个特定博文的评论。

这就是我们在这个例子中想要保留的所有状态，但是你可以想象，可能还有其他有用的数据点和对数据的看法。我们可能想持久化，以便我们的客户以后查询。

在 Substrate 中，有一个 `Event` 的概念，它可以用来通知客户和用户发生了什么事情 -- 例如，考虑使用 WebSockets 来通知你一个帖子被评论了。

为了这个目的，我们需要为我们的每一个 action 添加一个 event:

```rust
#[pallet::event]
#[pallet::generate_deposit(pub(super) fn deposit_event)]
pub enum Event<T: Config> {
        BlogPostCreated(Vec<u8>, T::AccountId, T::Hash),
        BlogPostCommentCreated(Vec<u8>, T::AccountId, T::Hash),
        Tipped(T::AccountId, T::Hash),
}
```

我们为博文的创建添加事件，包括博文的内容、作者和 ID，对评论也是如此。当用户为一篇博文提供小费时，我们会发送小费者的账户 ID 和博文的 ID，该博文的作者已经得到小费。

在继续实现创建博文、评论和小费的实际逻辑之前，我们还需要一些类型来处理错误。我们想限制博文和评论的大小。博客文章的长度应该在 64 到 4,096 字节之间，评论的长度应该在 64 到 1,024 字节之间。这主要是为了展示后面的一些错误处理，但无论如何，这样做都是一个好主意。

首先，我们在我们的 pallet 配置中添加一些常量。

```rust
#[pallet::config]
pub trait Config: frame_system::Config {
        type Event: From<Event<Self>> + IsType<<Self as frame_system::Config>::Event>;

        type Currency: Currency<Self::AccountId>;

        #[pallet::constant]
        type BlogPostMinBytes: Get<u32>;// <-- new

        #[pallet::constant]
        type BlogPostMaxBytes: Get<u32>;// <-- new

        #[pallet::constant]
        type BlogPostCommentMinBytes: Get<u32>;// <-- new

        #[pallet::constant]
        type BlogPostCommentMaxBytes: Get<u32>; // <-- new
}
```

和一些错误类型：

```rust
// Errors inform users that something went wrong.
#[pallet::error]
pub enum Error<T> {
        /// Error names should be descriptive.
        NoneValue,
        /// Errors should have helpful documentation associated with them.
        StorageOverflow,
        BlogPostNotEnoughBytes, // <-- new
        BlogPostTooManyBytes, // <-- new
        BlogPostCommentNotEnoughBytes,// <-- new
        BlogPostCommentTooManyBytes,// <-- new
        BlogPostNotFound,// <-- new
        TipperIsAuthor,// <-- new
}
```

很好！现在我们需要在 `runtime/lib.rs` 处为我们的常量定义值。

```rust
parameter_types! {
        pub const Version: RuntimeVersion = VERSION;
        pub const BlockHashCount: BlockNumber = 2400;
        /// We allow for 2 seconds of compute with a 6 second average block time.
        pub BlockWeights: frame_system::limits::BlockWeights = frame_system::limits::BlockWeights
                ::with_sensible_defaults(2 * WEIGHT_PER_SECOND, NORMAL_DISPATCH_RATIO);
        pub BlockLength: frame_system::limits::BlockLength = frame_system::limits::BlockLength
                ::max_with_normal_ratio(5 * 1024 * 1024, NORMAL_DISPATCH_RATIO);
        pub const SS58Prefix: u8 = 42;
        pub const BlogPostMinBytes: u32 = 64; // <-- new
        pub const BlogPostMaxBytes: u32 = 4096;// <-- new
        pub const BlogPostCommentMinBytes: u32 = 64;// <-- new
        pub const BlogPostCommentMaxBytes: u32 = 1024;// <-- new
}

...
...

/// Configure the pallet-blogchain in pallets/blogchain.
impl pallet_blogchain::Config for Runtime {
        type Event = Event;
        type Currency = Balances;
        type BlogPostMinBytes = BlogPostMinBytes; // <-- new
        type BlogPostMaxBytes = BlogPostMaxBytes; // <-- new
        type BlogPostCommentMinBytes = BlogPostCommentMinBytes; // <-- new
        type BlogPostCommentMaxBytes = BlogPostCommentMaxBytes; // <-- new
}
```

类型和存储就这样了。接下来，我们将实现我们的功能。

## 函数和外部交易

让我们从创建博客文章的外部交易开始。

```rust
#[pallet::weight(10000)]
#[transactional]
pub fn create_blog_post(origin: OriginFor<T>, content: Vec<u8>) -> DispatchResult {
        let author = ensure_signed(origin)?;

        ensure!(
                (content.len() as u32) > T::BlogPostMinBytes::get(),
                <Error<T>>::BlogPostNotEnoughBytes
        );

        ensure!(
                (content.len() as u32) < T::BlogPostMaxBytes::get(),
                <Error<T>>::BlogPostTooManyBytes
        );

        let blog_post = BlogPost { content: content.clone(), author: author.clone() };

        let blog_post_id = T::Hashing::hash_of(&blog_post);

        <BlogPosts<T>>::insert(blog_post_id, blog_post);

        let comments_vec: Vec<BlogPostComment<T>> = Vec::new();
        <BlogPostComments<T>>::insert(blog_post_id, comments_vec);

        Self::deposit_event(Event::BlogPostCreated(content, author, blog_post_id));

        Ok(())
}
```

让我们从顶部开始。通过 `pallet::weight`，我们定义了这个外部交易的计算 weight。在这个例子中，我们对它进行硬编码，因为这只是一个简单的例子。这个值应该与这个操作所需的处理资源量有关系，执行这个操作的费用将包括这个 weight，所以 weight 越高，执行一个外在交易的费用就越高。

`transactional` 宏使得这个外部交易的任何状态变化只有在它没有返回错误时才会进行。这类似于关系型数据库的原子交易行为，只有当所有的变化都通过时，你才想持久化一个变化。我们在这里需要这样做，因为我们同时添加了一个新的博文和一个空的评论列表。

我们做的第一件事是验证交易和交易的作者。这给了我们触发外部交易的用户的账户 ID。我们还定义了一个传入的内容向量，我们也期望在此由调用者给出。

然后，我们使用 `ensure!` 宏，它类似于断言，来验证博文是否在我们的长度限制之内，如果不是，则触发错误。

一旦验证通过，我们就用内容、作者和 `blog_post_id` 创建一个博文结构体，`blog_post_id` 是这个结构体的 hash。

然后这个结构体被添加到 `BlogPosts` StorageMap 中，我们初始化一个空的评论列表并将其持久化到 `BlogPostComments` StorageMap 中。这意味着，这个函数实际上改变了区块链上的状态。

一旦一切完成，我们就会触发 `BlogPostCreated` 事件，客户可以监听它，看看是否有什么变化。

这其实是很简单的，对吗？这类似于你[在网络应用中实现一个处理程序](https://blog.logrocket.com/a-guide-to-react-onclick-event-handlers-d411943b14dd/)的方式。这就是添加一个简单的外部交易的全部内容，它使用户能够交互和操纵我们应用程序的共享状态。不需要任何魔法。

添加评论和小费的实现同样简单明了。让我们接下来看看这些。

```rust
#[pallet::weight(5000)]
pub fn create_blog_post_comment(
        origin: OriginFor<T>,
        content: Vec<u8>,
        blog_post_id: T::Hash,
) -> DispatchResult {
        let comment_author = ensure_signed(origin)?;

        ensure!(
                (content.len() as u32) > T::BlogPostMinBytes::get(),
                <Error<T>>::BlogPostCommentNotEnoughBytes
        );

        ensure!(
                (content.len() as u32) < T::BlogPostMaxBytes::get(),
                <Error<T>>::BlogPostCommentTooManyBytes
        );

        let blog_post_comment = BlogPostComment {
                author: comment_author.clone(),
                content: content.clone(),
                blog_post_id: blog_post_id.clone(),
        };

        <BlogPostComments<T>>::mutate(blog_post_id, |comments| match comments {
                None => Err(()),
                Some(vec) => {
                        vec.push(blog_post_comment);
                        Ok(())
                },
        })
        .map_err(|_| <Error<T>>::BlogPostNotFound)?;

        Self::deposit_event(Event::BlogPostCommentCreated(
                content,
                comment_author,
                blog_post_id,
        ));

        Ok(())
}
```

正如你所看到的，这里的 weight 要小一些；要做的事情比较少，我们想让评论比发帖更便宜一些，以鼓励讨论。

处理逻辑类似：我们检查作者，验证内容，并为评论创建一个结构体。

然而，这一次，我们不是简单地在现有的地图上添加一个值，而是需要操作一个现有的条目。为此，我们在 StorageMap 上使用 mutate 方法。如果我们没有找到一个值，我们就会触发一个 BlogPostNotFound 错误。如果用户发送了一个无效的博文 ID，就会发生这种情况。

如果找到了博文，我们只需将博文评论推送到评论的向量上，并触发事件来通知客户这一变化 -- 很简单。

对于我们的小费功能，我们希望用户提供一个博文 ID 和一个他们想发送给作者的金额。让我们来看看如何操作。

```rust
#[pallet::weight(500)]
pub fn tip_blog_post(
        origin: OriginFor<T>,
        blog_post_id: T::Hash,
        amount: <<T as Config>::Currency as Currency<<T as frame_system::Config>::AccountId>>::Balance,
) -> DispatchResult {
        let tipper = ensure_signed(origin)?;

        let blog_post = Self::blog_posts(&blog_post_id).ok_or(<Error<T>>::BlogPostNotFound)?;
        let blog_post_author = blog_post.author;

        ensure!(tipper != blog_post_author, <Error<T>>::TipperIsAuthor);

        T::Currency::transfer(
                &tipper,
                &blog_post_author,
                amount,
                ExistenceRequirement::KeepAlive,
        )?;

        Self::deposit_event(Event::Tipped(tipper, blog_post_id));

        Ok(())
}
```

正如你所看到的，我们希望得到 `blog_post_id`，一个 `T::Hash`，和 `amount`，这是我们 pallet 中配置的 `Currency` 的 `Balance`。

我们验证 `tipper`，并检查给定的博文 ID 是否真的指向一个现有的博文，如果不是，则返回错误。

如果我们找到一篇博文，我们需要从结构中检索其作者，并确保小费与作者不一样。这不会是世界末日，但支付费用给自己送钱也是没有意义的。

然后，如果一切正常，我们在 pallet 的 `Currency` 上使用 `T::Currency::transfer` 函数 -- 记得，这就是为什么我们在教程的开头给 pallet 添加了 `Currency` 处理方法 -- 将小费者的资金数额发送到 `blog_post_author`。在这篇文章中，我们不去担心 `ExistenceRequirement` 标志，但如果你有兴趣，你可以在 Substrate 文档中阅读它。

这就是我们目前要实现的所有功能。你可以想在这里实现许多更有趣的功能，比如作者能够删除他们的博客文章，这可以退还一部分小费。或者你可以实现一个审核功能，用户可以标记出不合适的文章。你甚至可以实施一个经济激励系统，奖励那些提供和确认/否认标记报告的用户。

一旦你开始从这个激励系统和博弈论的角度思考，能够轻松实现和调整这些系统是相当酷的。有很多空间可以进行实验，提出新的想法来运行协作系统。

让我们来测试一下我们的系统，看看它是否工作！

## 测试

现在，为了测试我们的简单应用，让我们先编译一个 release 版本：

```bash
cargo build --release
```

接下来，我们将在开发模式下启动一个本地节点，每次启动时都有一个新的状态。

```bash
./target/release/node-blogchain --tmp --dev
```

为了和我们的应用交互，我们有两个选项：

* [Substrate Front End Template](https://github.com/substrate-developer-hub/substrate-front-end-template)
* [Polkadot-JS App UI](https://polkadot.js.org/apps/#/explorer)

在这里，我们将使用 Substrate 前端模板，但可以随意玩玩 Polkadot-JS 应用，将其设置为 `Development -> Local Node`.

要下载、设置和启动前端模板，只需执行：

```bash
git clone https://github.com/substrate-developer-hub/substrate-front-end-template.git
cd substrate-front-end-template
yarn install
yarn start
```

这将会启动一个前端应用在 <http://localhost:8000/substrate-front-end-template>

在那里，你已经可以看到你的区块链应用状态。在右上方，你可以看到选定的账户，并在现有账户之间进行切换。

![substrate-front-end-template-overview](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402-custom-blockchain-implementation-rust-substrate/substrate-front-end-template-overview.jpeg)

下面，你可以看到每个账户的余额：

![substrate-front-end-template-balances](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402-custom-blockchain-implementation-rust-substrate/substrate-front-end-template-balances.jpeg)

现在，让我们通过使用左下方的 `Pallet Interactor` 来创建我们的第一篇博文。我们选择 `blogchain` pallet 和 `createBlogPost` 外部交易。然后，在内容栏里写一些文字（至少 64 字节，最多 4096 字节），然后点击 `Signed`。

![substrate-front-end-template-create-blog-post](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402-custom-blockchain-implementation-rust-substrate/substrate-front-end-template-create-blog-post.jpeg)

很好！博文的创建成功了，我们可以在右边的 `Events` 流中看到。

让我们通过选择 `Query` 单选按钮来查询我们的博文。然后，再次选择 `blogchain` pallet 和 `blogPosts` 存储（这是我们实际的 `StorageMap`，我们上面定义的）。
然后，从 `Events` 流中，复制博文的 ID，将其粘贴到表格中，并点击 `Query`。

![substrate-front-end-template-blog-post-id](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402-custom-blockchain-implementation-rust-substrate/substrate-front-end-template-blog-post-id.jpeg)

![substrate-front-end-template-queried-blog-post](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402-custom-blockchain-implementation-rust-substrate/substrate-front-end-template-queried-blog-post.jpeg)

如果我们使用一个有效的博文 ID，我们将以如下形式获得这个条目的链上的持久化数据。

```json
{"content":"0x5468697320697320616e20696e746572657374696e672c20696e6e6f76617469766520616e642077656c6c207772697474656e20626c6f6720706f73742e20496e207468697320626c6f6720706f73742077652077696c6c206578706c6f7265207665727920696e746572657374696e6720636f6e636570747321","author":"5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY"}
```

其内容是十六进制编码的字符串。你可以使用一个[免费的在线工具](https://codebeautify.org/hex-string-converter)将其转换为字符串。

![substrate-front-end-template-content-plain-text](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402-custom-blockchain-implementation-rust-substrate/substrate-front-end-template-content-plain-text.jpeg)

你也可以在 JSON 中找到 `author` 字段，它对应于你在 `Balances` 表中可以找到的账户 ID。

接下来，让我们创建一个评论。在右上方切换到 `Bob` 的账户，使用 `Pallet Interactor` 找到 `blogchain → createBlogPostComment`。

![substrate-front-end-template-create-comment](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402-custom-blockchain-implementation-rust-substrate/substrate-front-end-template-create-comment.jpeg)

再次输入博文 ID 和一个评论（长度为 64-1024 字节）。点击 `Signed` 后，我们可以再次看 `Events` 流 ，这表明费用已经从 Bob 的账户中提取，并且外部交易工作。

现在我们也可以查询评论了：

![substrate-front-end-template-create-comment-2](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402-custom-blockchain-implementation-rust-substrate/substrate-front-end-template-create-comment-2.jpeg)

![substrate-front-end-template-comment-content-plain-text](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402-custom-blockchain-implementation-rust-substrate/substrate-front-end-template-comment-content-plain-text.jpeg)

最后，让我们给这篇博文的作者送上一份小费。

![substrate-front-end-template-tip-author](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402-custom-blockchain-implementation-rust-substrate/substrate-front-end-template-tip-author.jpeg)

然后，我们可以看到余额如何变化：

![substrate-front-end-template-balance-before-tip](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220402-custom-blockchain-implementation-rust-substrate/substrate-front-end-template-balance-before-tip.jpeg)

就是这样 -- 它工作了！非常酷的东西。

请随意通过前端模板对应用程序进行更多的操作。例如，你可以在事件反馈中看到错误情况下发生的案例（例如，一篇博客文章太短）。探索愉快！

## 结论

基于 [Substrate Node Template](https://github.com/substrate-developer-hub/substrate-node-template) 并使用官方文档，我们能够使用 Substrate 构建一个非常简单的自定义区块链应用程序。

当然，这篇文章只触及了用 Substrate 进行区块链开发的表面，但正如你在这个例子中所看到的，扩展 Substrate 模板以在上面构建自定义逻辑是相当简单的。

在开发体验上，一个肯定需要改进的领域是编译时间。进行修改和测试这些修改的周期需要相当长的时间，即使在强大的硬件上也是如此，这在你学习和测试的时候会让人感到沮丧。

也就是说，Substrate 作为一个框架，尽管它和整个区块链开发生态系统都很年轻，但已经被证明是相当强大的，有精彩的文档、例子和教程可供借鉴，以及它背后充满活力和提供帮助的社区。
