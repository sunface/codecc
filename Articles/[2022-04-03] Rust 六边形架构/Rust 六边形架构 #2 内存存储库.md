> 原文链接: https://alexis-lozano.com/hexagonal-architecture-in-rust-2/
>
> 翻译：[trdthg](https://github.com/trdthg)
>
> 选题：[trdthg](https://github.com/trdthg)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

# 2021-08-24 - Rust 六边形架构 #2 - 内存中的存储库

这篇文章是下面系列的一部分

- [Hexagonal architecture in Rust #1 - Domain](https://alexis-lozano.com/hexagonal-architecture-in-rust-1/)
- [Hexagonal architecture in Rust #2 - In-memory repository](https://alexis-lozano.com/hexagonal-architecture-in-rust-2/)
- [Hexagonal architecture in Rust #3 - HTTP API](https://alexis-lozano.com/hexagonal-architecture-in-rust-3/)
- [Hexagonal architecture in Rust #4 - Refactoring](https://alexis-lozano.com/hexagonal-architecture-in-rust-4/)
- [Hexagonal architecture in Rust #5 - Remaining use-cases](https://alexis-lozano.com/hexagonal-architecture-in-rust-5/)
- [Hexagonal architecture in Rust #6 - CLI](https://alexis-lozano.com/hexagonal-architecture-in-rust-6/)
- [Hexagonal architecture in Rust #7 - Long-lived repositories](https://alexis-lozano.com/hexagonal-architecture-in-rust-7/)

> 免责声明：在本文中，我对存储库会使用到一个简单的可变引用，因为现在我们只是在测试中使用它。在下一篇文章，我会进行一些优化 : )

在上一篇文章中，我们已经开始搭建基本的项目架构。我们已经有了域模块，里面包含一个用例和一些实体：

```
src
├── domain
│   ├── create_pokemon.rs
│   ├── entities.rs
│   └── mod.rs
└── main.rs
```

## 内存存储库

让我们回到我们的 `create_pokemon` 用例。
目前，它可以在成功时返回宝可梦的数量，当请求参数不符合业务规则时会返回一个错误。现在我们并没有一个实际存储宝可梦的地方。让我们来解决这个问题！你应该知道我喜欢从什么开始：一个测试
: )。这个测试将检查我们不能有两个相同编号的宝可梦。

```rs
use crate::repositories::pokemon::InMemoryRepository;

#[test]
fn it_should_return_a_conflict_error_when_pokemon_number_already_exists() {
    let number = PokemonNumber::try_from(25).unwrap();
    let name = PokemonName::try_from(String::from("Pikachu")).unwrap();
    let types = PokemonTypes::try_from(vec![String::from("Electric")]).unwrap();
    let mut repo = InMemoryRepository::new();
    repo.insert(number, name, types);
    let req = Request {
        number: 25,
        name: String::from("Charmander"),
        types: vec![String::from("Fire")],
    };

    let res = execute(&mut repo, req);

    match res {
        Response::Conflict => {}
        _ => unreachable!(),
    }
}
```

在个用例的测试中，我们直接在存储库中插入一个宝可梦。然后我们尝试使用用例再次插入一个具有相同编号的宝可梦。用例应该返回一个冲突错误。

像之前一样，它现在还不能通过编译，因为这里的很多代码都没有实现。让我们首先将 `Conflict` 错误添加到 `Response` 枚举中：

```rs
enum Response {
    ...
    Conflict,
}
```

接着，在宝可梦类型中增加一个火属性

```rs
enum PokemonType {
    Electric,
    Fire,
}

impl TryFrom<String> for PokemonType {
    type Error = ();

    fn try_from(t: String) -> Result<Self, Self::Error> {
        match t.as_str() {
            "Electric" => Ok(Self::Electric),
            "Fire" => Ok(Self::Fire),
            _ => Err(()),
        }
    }
}
```

您可能想知道内存中的存储库是什么。它是在我们不知道客户将要使用什么作为存储库时，暂时使用的存储库。这是我们的第一个实现，它主要用于测试。因为它可以像真正的存储库一样工作，所以我们能够使用它去向客户展示我们的进度并要求他提供反馈。如你所见，存储库
`repo` 被当作参数传递给用例：

```rs
use crate::repositories::pokemon::Repository;

fn execute(repo: &mut dyn Repository, req: Request) -> Response {
```

这里需要注意的一点是，`execute` 函数并不会得到具体的存储库实现，而是任何实现了 `Reposity` 特征的结构体。让我们在之前的两个测试用例中也补充
repo 参数：

```rs
#[test]
fn it_should_return_a_bad_request_error_when_request_is_invalid() {
    let mut repo = InMemoryRepository::new();
    let req = Request {
    ...
    let res = execute(&mut repo, req);
    ...
}

#[test]
fn it_should_return_the_pokemon_number_otherwise() {
    let mut repo = InMemoryRepository::new();
    let number = 25;
    ...
    let res = execute(&mut repo, req);
    ...
}
```

接下来，我们将在新模块 _repositories/pokemon.rs_ 中去定义 `InMemoryRepository` 结构体 和
`Repository` 特征：

```
src
├── domain
│   ├── create_pokemon.rs
│   ├── entities.rs
│   └── mod.rs
├── repositories
│   ├── mod.rs
│   └── pokemon.rs
└── main.rs
```

```rs
pub trait Repository {}

pub struct InMemoryRepository;

impl Repository for InMemoryRepository {}
```

不要忘了引用模块

```rs
// main.rs
mod repositories;

// repositories/mod.rs
pub mod pokemon;
```

接下来，让我们实现 `InMemoryRepository` 的 `new` 方法。在这里，`InMemoryRepository`
内部只是简单的存储了一个宝可梦列表

```rs
use crate::domain::entities::Pokemon;

pub struct InMemoryRepository {
    pokemons: Vec<Pokemon>,
}

impl InMemoryRepository {
    pub fn new() -> Self {
        let pokemons: Vec<Pokemon> = vec![];
        Self { pokemons }
    }
}
```

现在，终于是实现 `Pokemon` 实体的时候了：

```rs
pub struct Pokemon {
    pub number: PokemonNumber,
    name: PokemonName,
    types: PokemonTypes,
}

impl Pokemon {
    pub fn new(number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> Self {
        Self {
            number,
            name,
            types
        }
    }
}
```

同时，我们需要将 `entities.rs` 转为公开的：

```rs
// domain/mod.rs
pub mod entities;
```

现在唯一没有被实现的就剩下 `insert` 方法了，我们希望能够在任何实现了 `Repository` 特征的存储库上都能调用该方法，所以需要在 Trait
上添加一个函数签名, 并为 `InMemoryRepository` 结构体实现 `insert` 方法：

```rs
use crate::domain::entities::{Pokemon, PokemonName, PokemonNumber, PokemonTypes};

pub trait Repository {
    fn insert(&self, number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> PokemonNumber;
}

impl Repository for InMemoryRepository {
    fn insert(&self, number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> PokemonNumber {
        number
    }
}
```

让我们尝试运行测试：

```
cargo test
running 3 tests
test it_should_return_a_bad_request_error_when_request_is_invalid ... ok
test it_should_return_the_pokemon_number_otherwise ... ok
test it_should_return_a_conflict_error_when_pokemon_number_already_exists ... FAILED
```

第三个测试失败了，插入成功时，insert 应该返回一个宝可梦的编号，如果对应的编号已经存在，需要返回一个冲突的错误。所以我们现在要填加一个 `Insert`
结构体表示这两种结果，同时要将原本的不可边借用变为可变借用：

```rs
pub enum Insert {
    Ok(PokemonNumber),
    Conflict,
}

pub trait Repository {
    fn insert(&mut self, number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> Insert;
}
```

接下来实现 InMemoryRepository 的 insert 方法：

```rs
impl Repository for InMemoryRepository {
    fn insert(&mut self, number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> Insert {
        if self.pokemons.iter().any(|pokemon| pokemon.number == number) {
            return Insert::Conflict;
        }

        let number_clone = number.clone();
        self.pokemons.push(Pokemon::new(number_clone, name, types));
        Insert::Ok(number)
    }
}
```

为了让 `clone` 和 `==` 通过编译，我们还要为 `PokemonNumber` 实现 `PartialEq` 和 `Clone` 两个特征：

```rs
use std::cmp::PartialEq;

#[derive(PartialEq, Clone)]
pub struct PokemonNumber(u16);
```

最后，让 `execute` 函数调用 insert 方法

```rs
fn execute(repo: &mut dyn Repository, req: Request) -> Response {
    match (
        PokemonNumber::try_from(req.number),
        PokemonName::try_from(req.name),
        PokemonTypes::try_from(req.types),
    ) {
        (Ok(number), Ok(name), Ok(types)) => match repo.insert(number, name, types) {
            Insert::Ok(number) => Response::Ok(u16::from(number)),
            Insert::Conflict => Response::Conflict,
        },
        _ => Response::BadRequest,
    }
}
```

再次运行测试

```
cargo test
running 3 tests
test it_should_return_a_bad_request_error_when_request_is_invalid ... ok
test it_should_return_the_pokemon_number_otherwise ... ok
test it_should_return_a_conflict_error_when_pokemon_number_already_exists ... ok
```

太棒了，冲突测试也通过了！

## 以为已经结束了吗？

没有。在存储库中还有一种问题会发生。假设由于某些意外，存储库无法正常工作。如果是数据库，那就是连接错误，如果是 API，那就是网络错误。我们也应该处理这种情况。

你知道我现在要做什么：写一个测试！

```rs
#[test]
fn it_should_return_an_error_when_an_unexpected_error_happens() {
    let mut repo = InMemoryRepository::new().with_error();
    let number = 25;
    let req = Request {
        number,
        name: String::from("Pikachu"),
        types: vec![String::from("Electric")],
    };

    let res = execute(&mut repo, req);

    match res {
        Response::Error => {}
        _ => unreachable!(),
    };
}
```

这个测试有两个不同点。第一，我们添加了 `with_error` 方法表示存储库连接异常。第二，我们需要检查 Respnse 是否发生异常。

首先为 Response 添加一个新的类型

```rs
enum Response {
    ...
    Error,
}
```

现在我们要实现 `with_error` 方法，我的想法是在 `InMemoryRepository` 中增加一个 `error`
字段，表示是否会在连接存储库时进行检查。如果 `error` 为 `true` 我们就返回一个错误，否则返回正常结果：

```rs
pub enum Insert {
    ...
    Error,
}

pub struct InMemoryRepository {
    error: bool,
    pokemons: Vec<Pokemon>,
}

impl InMemoryRepository {
    pub fn new() -> Self {
        let pokemons: Vec<Pokemon> = vec![];
        Self {
            error: false,
            pokemons,
        }
    }

    pub fn with_error(self) -> Self {
        Self {
            error: true,
            ..self
        }
    }
}

impl Repository for InMemoryRepository {
    fn insert(&mut self, number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> Insert {
        if self.error {
            return Insert::Error;
        }

        if self.pokemons.iter().any(|pokemon| pokemon.number == number) {
            return Insert::Conflict;
        }

        let number_clone = number.clone();
        self.pokemons.push(Pokemon::new(number_clone, name, types));
        Insert::Ok(number)
    }
}
```

同样的，在 `execute` 中处理这种情况

```rs
fn execute(repo: &mut dyn Repository, req: Request) -> Response {
    match (
        PokemonNumber::try_from(req.number),
        PokemonName::try_from(req.name),
        PokemonTypes::try_from(req.types),
    ) {
        (Ok(number), Ok(name), Ok(types)) => match repo.insert(number, name, types) {
            Insert::Ok(number) => Response::Ok(u16::from(number)),
            Insert::Conflict => Response::Conflict,
            Insert::Error => Response::Error,
        },
        _ => Response::BadRequest,
    }
}
```

让我们运行测试

```
cargo test
running 4 tests
test it_should_return_a_bad_request_error_when_request_is_invalid ... ok
test it_should_return_a_conflict_error_when_pokemon_number_already_exists ... ok
test it_should_return_an_error_when_an_unexpected_error_happens ... ok
test it_should_return_the_pokemon_number_otherwise ... ok
```

\o/

## 下一步

这篇文章的长度已经足够了，让我们暂时停到这里。下一次，我将为前端部分先实现 HTTP
API。之后我会处理其他的用例。我们之后还会实现更多的存储库和前端接口，这些功能会通过添加不同的命令行参数进行开启。

和以前一样，我会在 [github](https://github.com/alexislozano/pokedex/tree/article-2)
上创建一个包含所有更改的分支。
