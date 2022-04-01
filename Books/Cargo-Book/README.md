# Cargo Book
Rust 语言的名气之所以这么大，保守估计 `Cargo` 的贡献就占了三分之一。

`Cargo` 是包管理工具，可以用于依赖包的下载、编译、更新、分发等，与 `Cargo` 一样有名的还有 [`crates.io`](https://crates.io)，它是社区提供的包注册中心：用户可以将自己的包发布到该注册中心，然后其它用户通过注册中心引入该包。

> 本书由 [RustTT 翻译小组](https://rusttt.com) 进行翻译，并对内容进行了一些调整，更便于国内读者阅读
> 
> 英文原文 [The Cargo Book](https://doc.rust-lang.org/stable/cargo/index.html)


## 目录
- [上手使用](cargo/getting-started.md)
- [基础指南](cargo/guide/intro.md)
    - [为何会有Cargo](cargo/guide/why-exist.md)
    - [下载并构建Package](cargo/guide/download-package.md)
    - [添加依赖](cargo/guide/dependencies.md)
    - [Package目录结构](cargo/guide/package-layout.md)
    - [Cargo.toml vs Cargo.lock](cargo/guide/cargo-toml-lock.md)
    - [测试和CI](cargo/guide/tests-ci.md)
    - [Cargo缓存](cargo/guide/cargo-cache.md)
    - [Build缓存](cargo/guide/build-cache.md)
- [进阶指南](cargo/reference/intro.md)
    - [指定依赖项](cargo/reference/specify-deps.md)
    - [依赖覆盖](cargo/reference/deps-overriding.md)
    - [Cargo.toml清单详解](cargo/reference/manifest.md)
    - [Cargo Target](cargo/reference/cargo-target.md)
    - [工作空间Workspace](cargo/reference/workspaces.md)
    - [条件编译Features](cargo/reference/features/intro.md)
        - [Features示例](cargo/reference/features/examples.md)
    - [发布配置Profile](cargo/reference/profiles.md)
    - [通过config.toml对Cargo进行配置](cargo/reference/configuration.md)
    - [发布到crates.io](cargo/reference/publishing-on-crates.io.md)
    - [构建脚本 build.rs](cargo/reference/build-script/intro.md)
        - [构建脚本示例](cargo/reference/build-script/examples.md)


<img src="https://doc.rust-lang.org/stable/cargo/images/Cargo-Logo-Small.png" />
