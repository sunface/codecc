# Rust 六边形架构

## 六边形架构基本介绍

六边形架构，是一种软件设计模式。依照这种架构创建的程序，能够在没有 UI
或数据库的情况下正常工作。所以即使没有数据库你依然可以进行开发和运行自动化测试，而且无需其他用户参与。

### 端口与适配器

六边形架构也叫做端口与适配器架构。

![](https://alistair.cockburn.us/wp-content/uploads/2018/02/Hexagonal-architecture-complex-example.gif)

什么是端口？

端口指的是六边形的边，属于应用程序的内部，是我们应用程序的入口和出口。它定义了一个接口，表示设备如何使用我们的用例。在 Rust 里就是由一个
Trait，以及一些 DTO 组成

什么是适配器？

适配器围绕着端口编写，能够将输入转化为符合端口的类型，并把输入转化为用用程序内部的方法调用。换句话说就是 `controller` 或者是命令行一条命令处理器。

当任何外部设备 (比如：WEB，APP，终端，测试...)
想要访问输入端口时，首先这个输入会被该设备对应的输入适配器，转化为符合要求的可用的方法调用，或者是消息，然后再传递给我们的应用程序。

当我们的应用程序需要向外发送数据时，首先它会把数据通过输出端口传递给输出适配器，然后再传递给输出对象(比如: 数据库，mock，API，邮件，消息队列...)

因此我们的应用程序外部是完全隔离的。

### 应用程序核心

使用六边形架构之后，我们应用程序的核心部分通常被称作域，域中有三个核概念：

- 实体 (Entities)：只是简单的定义了对象。
- 交互器 (Interactors)：实现复杂的业务逻辑，在本文里我们会将其称为用例 Usecase。
- 存储库 (Repositories)：只定义了操作实体的方法。

### 优点

在传统的分层架构中，只能从上往下调用。

而现在我们把接口保留在域中，域不再依赖外部的外部实现。保证了外部设备是可替换的。当业务逻辑需要用到数据存储时，直接调用抽象接口即可。

## 将要做些什么？

在接下来的文章里，我们将会实现一个 pokemon 服务。主要功能是增加、删除和查询。

为了体现六边形架构的优点，我们将会实现两套用户接口，包括 HTTP API 和 CLI。 三套存储库，包括内存，SQLite 数据库和 Airtable
(一个在线表格应用)。

六边形架构的核心不需要依赖于具体的存储库或者是用户接口，为了我们的程序能够稳定运行，我们也会非常注重于单元测试。一旦我们的域稳定下来，实现用户接口，替换存储库都会是非常简单快速的。

## 项目结构

```
src/
├── api
│   ├── fetch_pokemon.rs
│   └── mod.rs
├── cli
│   ├── fetch_pokemon.rs
│   └── mod.rs
├── domain
│   ├── entities.rs
│   ├── fetch_pokemon.rs
│   └── mod.rs
├── main.rs
└── repositories
    ├── mod.rs
    └── pokemon.rs
```

六边形架构的核心部分通常被称作域，就是下面的 `domain` 模块，里面包括实体的定义和用例的实现。`api` 和 `cli`
是两套用户接口。`repositories` 中则是存储库的定义与实现

## 延申阅读

- [六边形架构 ( hexagonal-architecture )](https://alistair.cockburn.us/hexagonal-architecture/)
- [干净架构 ( the-clean-architecture )](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [洋葱架构 ( The Onion Architecture )](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/)
- [软件架构编年史 ( The Software Architecture Chronicles )](https://herbertograca.com/2017/07/03/the-software-architecture-chronicles/)
