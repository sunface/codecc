# 手把手实现 Rust 链表

> 其它语言：兄弟，语言学了吗？来写一个链表证明你基本掌握了语法。
> 
> Rust 语言: 兄弟，语言精通了吗？来写一个链表证明你已经精通了 Rust！


上面的对话非常真实，我们在之前的章节也讲过[初学者学习 Rust 应该避免的坑](https://course.rs/sth-you-should-not-do.html#千万别从链表或图开始练手)，其中最重要的就是 - 不要写链表或者类似的数据结构！

而本章，你就将见识到何为真正的深坑，看完后，就知道没有提早跳进去是一个多么幸运的事。总之，在专题中，你将学会如何使用 Rust 来实现链表。

> 本书由 [Rustt 翻译组](https://rustt.org) 进行翻译，并对内容进行了一些调整，更便于国内读者阅读
>
> **专题内容翻译自英文开源书 [Learning Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/)，但是在内容上做了一些调整(原书虽然非常棒，但是在一些内容组织和文字细节上我觉得还是可以优化下的 ：D)，希望大家喜欢。**


- [手把手带你实现链表](intro.md)
    - [我们到底需不需要链表](do-we-need-it.md)
    - [不太优秀的单向链表：栈](bad-stack/intro.md)
      - [数据布局](bad-stack/layout.md)
      - [基本操作](bad-stack/basic-operations.md)
      - [最后实现](bad-stack/final-code.md)
    - [还可以的单向链表](ok-stack/intro.md)
      - [优化类型定义](ok-stack/type-optimizing.md)
      - [定义 Peek 函数](ok-stack/peek.md)
      - [IntoIter 和 Iter](ok-stack/iter.md)
      - [IterMut以及完整代码](ok-stack/itermut.md)
    - [持久化单向链表](persistent-stack/intro.md)
      - [数据布局和基本操作](persistent-stack/layout.md)
      - [Drop、Arc 及完整代码](persistent-stack/drop-arc.md)
    - [不咋样的双端队列](deque/intro.md)
      - [数据布局和基本操作](deque/layout.md)
      - [Peek](deque/peek.md)
      - [基本操作的对称镜像](deque/symmetric.md)
      - [迭代器](deque/iterator.md)
      - [最终代码](deque/final-code.md)
    - [不错的unsafe队列](unsafe-queue/intro.md)
      - [数据布局](unsafe-queue/layout.md)
      - [基本操作](unsafe-queue/basics.md)
      - [Miri](unsafe-queue/miri.md)
      - [栈借用](unsafe-queue/stacked-borrow.md)
      - [测试栈借用](unsafe-queue/testing-stacked-borrow.md)
      - [数据布局2](unsafe-queue/layout2.md)
      - [额外的操作](unsafe-queue/extra-junk.md)
      - [最终代码](unsafe-queue/final-code.md)
    - [使用高级技巧实现链表](advanced-lists/intro.md)
      - [生产级可用的双向链表](advanced-lists/unsafe-deque.md)
      - [双单向链表](advanced-lists/double-singly.md)
      - [栈上的链表](advanced-lists/stack-allocated.md)


