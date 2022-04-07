> 原文链接: https://alexis-lozano.com/hexagonal-architecture-in-rust-4/
>
> 翻译：[trdthg](https://github.com/trdthg)
>
> 选题：[trdthg](https://github.com/trdthg)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

# 2021-09-02 - Rust 六边形架构 #4 - 重构

这篇文章是下面系列的一部分

- [Hexagonal architecture in Rust #1 - Domain](https://alexis-lozano.com/hexagonal-architecture-in-rust-1/)
- [Hexagonal architecture in Rust #2 - In-memory repository](https://alexis-lozano.com/hexagonal-architecture-in-rust-2/)
- [Hexagonal architecture in Rust #3 - HTTP API](https://alexis-lozano.com/hexagonal-architecture-in-rust-3/)
- [Hexagonal architecture in Rust #4 - Refactoring](https://alexis-lozano.com/hexagonal-architecture-in-rust-4/)
- [Hexagonal architecture in Rust #5 - Remaining use-cases](https://alexis-lozano.com/hexagonal-architecture-in-rust-5/)
- [Hexagonal architecture in Rust #6 - CLI](https://alexis-lozano.com/hexagonal-architecture-in-rust-6/)
- [Hexagonal architecture in Rust #7 - Long-lived repositories](https://alexis-lozano.com/hexagonal-architecture-in-rust-7/)

嗨，又是我！起初，我想实现我们仍然需要处理的剩余的用例。但这将会是下一次的内容。今天我们将做一些重构 :)

## 开始之前

我做了两个我在第一篇文章中没有看到的更改。在 `domain/entities.rs` 中，我用 Self 替换了 u16：

```rs
impl From<PokemonNumber> for u16 {
    fn from(n: PokemonNumber) -> Self {
        n.0
    }
}
```

在 `repositories/pokemons.rs` 中，我在 `with_error` 上添加了一个测试注释：

```rs
impl InMemoryRepository {
    #[cfg(test)]
    pub fn with_error(self) -> Self {
        Self {
            error: true,
            ..self
        }
    }
}
```

让我们现在进行重构 :)

## 使用 `Result` 替换自定义枚举

之前我们使用自定义枚举作为用例和 存储库 的返回值，现在把他们重构为 Result。

### 更改用例的返回值类型

首先，我们将用例的返回值暂时设置为 500 以方便测试、

```rs
pub fn serve(repo: Arc<dyn Repository>, req: &rouille::Request) -> rouille::Response {
    let req = ...
    rouille::Response::from(Status::InternalServerError)
    // match create_pokemon::execute(repo, req) {
    //     create_pokemon::Response::Ok(number) => rouille::Response::json(&Response { number }),
    //     create_pokemon::Response::BadRequest => rouille::Response::from(Status::BadRequest),
    //     create_pokemon::Response::Conflict => rouille::Response::from(Status::Conflict),
    //     create_pokemon::Response::Error => rouille::Response::from(Status::InternalServerError),
    // }
}
```

现在我们将测试的返回值修改为 Result 类型：

```rs
    #[test]
    fn it_should_return_a_bad_request_error_when_request_is_invalid() {
        ...
        match res {
            Err(Error::BadRequest) => {}
            _ => unreachable!(),
        };
    }

    #[test]
    fn it_should_return_a_conflict_error_when_pokemon_number_already_exists() {
        ...
        match res {
            Err(Error::Conflict) => {}
            _ => unreachable!(),
        }
    }

    #[test]
    fn it_should_return_an_unknown_error_when_an_unexpected_error_happens() {
        ...
        match res {
            Err(Error::Unknown) => {}
            _ => unreachable!(),
        };
    }

    #[test]
    fn it_should_return_the_pokemon_number_otherwise() {
        ...
        match res {
            Ok(res_number) => assert_eq!(res_number, number),
            _ => unreachable!(),
        };
    }
}
```

接着再修改用例，把它的返回值修改为 Result ：

```rs
pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Result<u16, Error> {
    ...
        (Ok(number), Ok(name), Ok(types)) => match repo.insert(number, name, types) {
            Insert::Ok(number) => Ok(u16::from(number)),
            Insert::Conflict => Err(Error::Conflict),
            Insert::Error => Err(Error::Unknown),
        },
        _ => Err(Error::BadRequest),
    }
}
```

测试现在应该通过了！

```
cargo test
running 4 tests
test it_should_return_a_conflict_error_when_pokemon_number_already_exists ... ok
test it_should_return_a_bad_request_error_when_request_is_invalid ... ok
test it_should_return_an_unknown_error_when_an_unexpected_error_happens ... ok
test it_should_return_the_pokemon_number_otherwise ... ok
```

最后再去修改我们的 API：

```rs
pub fn serve(repo: Arc<dyn Repository>, req: &rouille::Request) -> rouille::Response {
    let req = ...
    match create_pokemon::execute(repo, req) {
        Ok(number) => rouille::Response::json(&Response { number }),
        Err(create_pokemon::Error::BadRequest) => rouille::Response::from(Status::BadRequest),
        Err(create_pokemon::Error::Conflict) => rouille::Response::from(Status::Conflict),
        Err(create_pokemon::Error::Unknown) => rouille::Response::from(Status::InternalServerError),
    }
}
```

Usecase 修改完成，接下来我们去处理 Reposity

### 更改 Repository 的返回类型

Repository 没有测试，所以我们从修改用例调用 repo 的返回值开始：

```rs
use crate::repositories::pokemon::{InsertError, ...};

...
pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Result<u16, Error> {
    ...
        (Ok(number), Ok(name), Ok(types)) => match repo.insert(number, name, types) {
            Ok(number) => Ok(u16::from(number)),
            Err(InsertError::Conflict) => Err(Error::Conflict),
            Err(InsertError::Unknown) => Err(Error::Unknown),
        },
        _ => Err(Error::BadRequest),
    }
}
```

在宝可梦编号冲突的测试时，您应该将 `.ok()` 添加到存储库 insert操作之后 。 现在让我们在 _repositories/pokemon.rs_
中删除 `Insert` 并创建 `InsertError`：

```rs
pub enum InsertError {
    Conflict,
    Unknown,
}
```

最后在更改 `Repository` Trait 和 `InMemoryRepository` 的返回值类型即可：

```rs
pub trait Repository: Send + Sync {
    fn insert(&self,
        number: PokemonNumber,
        name: PokemonName,
        types: PokemonTypes,
    ) -> Result<PokemonNumber, InsertError>;
}

impl Repository for InMemoryRepository {
    fn insert(
        &self,
        number: PokemonNumber,
        name: PokemonName,
        types: PokemonTypes,
    ) -> Result<PokemonNumber, InsertError> {
        if self.error {
            return Err(InsertError::Unknown);
        }

        let mut lock = match self.pokemons.lock() {
            Ok(lock) => lock,
            _ => return Err(InsertError::Unknown),
        };

        if lock.iter().any(|pokemon| pokemon.number == number) {
            return Err(InsertError::Conflict);
        }

        let number_clone = number.clone();
        lock.push(Pokemon::new(number_clone, name, types));
        Ok(number)
    }
}
```

## 填加一个新的用例

在常规的 HTTP API 中，每次创建一个新的对象，我通常会把这个对象在返回回去。特别是当返回的对象中包含一切前端没有传来的字段。比如 `create_at`
等由存储库添加的字段。

首先，我需要你像我们一开始那样暂时注释掉 `api/create_pokemon.rs`。 以便于我们专注于测试。

在 _domain/create\_pokemon.rs_ 中添加一个新的测试：

```rs
#[test]
fn it_should_return_the_pokemon_number_otherwise() {
    let repo = Arc::new(InMemoryRepository::new());
    let req = Request {
        number: 25,
        name: String::from("Pikachu"),
        types: vec![String::from("Electric")],
    };

    let res = execute(repo, req);

    match res {
        Ok(Response {
            number,
            name,
            types,
        }) => {
            assert_eq!(number, 25);
            assert_eq!(name, String::from("Pikachu"));
            assert_eq!(types, vec![String::from("Electric")]);
        }
        _ => unreachable!(),
    };
}
```

同时创建一个 `Response` 结构体

```rs
pub struct Response {
    pub number: u16,
    pub name: String,
    pub types: Vec<String>,
}
```

接下来，我们将修改 `execute` 函数，在插入成功时应该返回 `Pokemon` 的所有字段：

```rs
use crate::domain::entities::{Pokemon, ...};

pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Result<Response, Error> {
    ...
        (Ok(number), Ok(name), Ok(types)) => match repo.insert(number, name, types) {
            Ok(Pokemon {
                number,
                name,
                types,
            }) => Ok(Response {
                number: u16::from(number),
                name: String::from(name),
                types: Vec::<String>::from(types),
            }),
            Err(InsertError::Conflict) => Err(Error::Conflict),
            Err(InsertError::Unknown) => Err(Error::Unknown),
        },
        _ => Err(Error::BadRequest),
    }
}
```

在 `insert` 执行成功后，直接返回一个 `Pokemon` 结构体：

```rs
pub trait Repository: Send + Sync {
    fn insert(
        &self,
        number: PokemonNumber,
        name: PokemonName,
        types: PokemonTypes,
    ) -> Result<Pokemon, InsertError>;
}

impl Repository for InMemoryRepository {
    fn insert(
        &self,
        number: PokemonNumber,
        name: PokemonName,
        types: PokemonTypes,
    ) -> Result<Pokemon, InsertError> {
        ...
        let pokemon = Pokemon::new(number, name, types);
        lock.push(pokemon.clone());
        Ok(pokemon)
    }
}
```

为了使 `pokemon.clone()` 能够正常工作，我们需要为 `Pokemon` 实现 `Clone` Trait：

```rs
#[derive(Clone)]
pub struct PokemonName(String);

#[derive(Clone)]
pub struct PokemonTypes(Vec<PokemonType>);

#[derive(Clone)]
enum PokemonType {

#[derive(Clone)]
pub struct Pokemon {
```

现在存储库的插入逻辑已经完成，用例希望能够直接拿到 `Pokemon` 的 `name` 和 `types` 字段，我们需要把这两个字端也转为公开的：

```rs
pub struct Pokemon {
    pub number: PokemonNumber,
    pub name: PokemonName,
    pub types: PokemonTypes,
}
```

接着，我们需要为 `Response` 实现类型转换， 从 `PokemonNumber` 转换为 `u16`、从 `PokemonName` 转换为
`String`、从 `PokemonTypes` 转换为 `Vec<String>`:

```rs
impl From<PokemonName> for String {
    fn from(n: PokemonName) -> Self {
        n.0
    }
}

impl From<PokemonTypes> for Vec<String> {
    fn from(pts: PokemonTypes) -> Self {
        let mut ts = vec![];
        for pt in pts.0.into_iter() {
            ts.push(String::from(pt));
        }
        ts
    }
}

impl From<PokemonType> for String {
    fn from(t: PokemonType) -> Self {
        String::from(match t {
            PokemonType::Electric => "Electric",
            PokemonType::Fire => "Fire",
        })
    }
}
```

现在测试应该能够通过了：

```
cargo test
running 4 tests
test it_should_return_a_conflict_error_when_pokemon_number_already_exists ... ok
test it_should_return_a_bad_request_error_when_request_is_invalid ... ok
test it_should_return_an_unknown_error_when_an_unexpected_error_happens ... ok
test it_should_return_the_pokemon_number_otherwise ... ok
```

最后，我们去更新 api 的内容：

```rs
#[derive(Serialize)]
struct Response {
    number: u16,
    name: String,
    types: Vec<String>,
}

pub fn serve(repo: Arc<dyn Repository>, req: &rouille::Request) -> rouille::Response {
    let req = ...
    match create_pokemon::execute(repo, req) {
        Ok(create_pokemon::Response {
            number,
            name,
            types,
        }) => rouille::Response::json(&Response {
            number,
            name,
            types,
        }),
        Err(create_pokemon::Error::BadRequest) => rouille::Response::from(Status::BadRequest),
        Err(create_pokemon::Error::Conflict) => rouille::Response::from(Status::Conflict),
        Err(create_pokemon::Error::Unknown) => rouille::Response::from(Status::InternalServerError),
    }
}
```

`cargo run` 之后，再次向 server 发送数据：

```json
{
  "number": 17,
  "name": "Charmander",
  "types": [
    "Fire"
  ]
}
```

## 创建一些测试值

你喜欢在测试过程中使用 `PokemonName::try_from(String::from("Pikachu")).unwrap()`
之类的东西吗？让我们在 _domain/entities.rs_ 中创建一些函数:

### 宝可梦编号

```rs
#[cfg(test)]
impl PokemonNumber {
    pub fn pikachu() -> Self {
        Self(25)
    }

    pub fn charmander() -> Self {
        Self(4)
    }
}
```

### 宝可梦名字

```rs
#[cfg(test)]
impl PokemonName {
    pub fn pikachu() -> Self {
        Self(String::from("Pikachu"))
    }

    pub fn charmander() -> Self {
        Self(String::from("Charmander"))
    }

    pub fn bad() -> Self {
        Self(String::from(""))
    }
}
```

### 宝可梦类型

```rs
#[cfg(test)]
impl PokemonTypes {
    pub fn pikachu() -> Self {
        Self(vec![PokemonType::Electric])
    }

    pub fn charmander() -> Self {
        Self(vec![PokemonType::Fire])
    }
}
```

接下来让我们在用例测试中使用这些测试值。首先为 `Request` 添加一个测试用的 `new` 方法，以便我们更轻松的模拟一个请求：

```rs
#[cfg(test)]
mod tests {
    ...

    impl Request {
        fn new(number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> Self {
            Self {
                number: u16::from(number),
                name: String::from(name),
                types: Vec::<String>::from(types),
            }
        }
    }
}
```

接下来就是各种的测试：

```rs
#[test]
fn it_should_return_a_bad_request_error_when_request_is_invalid() {
    ...
    let req = Request::new(
        PokemonNumber::pikachu(),
        PokemonName::bad(),
        PokemonTypes::pikachu(),
    );
    ...
}

#[test]
fn it_should_return_a_conflict_error_when_pokemon_number_already_exists() {
    ...
    repo.insert(
        PokemonNumber::pikachu(),
        PokemonName::pikachu(),
        PokemonTypes::pikachu(),
    )
    .ok();
    let req = Request::new(
        PokemonNumber::pikachu(),
        PokemonName::charmander(),
        PokemonTypes::charmander(),
    );
    ...
}

#[test]
fn it_should_return_an_unknown_error_when_an_unexpected_error_happens() {
    ...
    let req = Request::new(
        PokemonNumber::pikachu(),
        PokemonName::pikachu(),
        PokemonTypes::pikachu(),
    );
    ...
}

#[test]
fn it_should_return_the_pokemon_otherwise() {
    ...
    let req = Request::new(
        PokemonNumber::pikachu(),
        PokemonName::pikachu(),
        PokemonTypes::pikachu(),
    );
    ...
    match res {
        Ok(res) => {
            assert_eq!(res.number, u16::from(PokemonNumber::pikachu()));
            assert_eq!(res.name, String::from(PokemonName::pikachu()));
            assert_eq!(res.types, Vec::<String>::from(PokemonTypes::pikachu()));
        }
        _ => unreachable!(),
    };
}
```

## 总结

我们终于完成了这个漫长的重构，希望一切顺利 :) 我保证，下次我们将去实现一些新的用例！

代码可以在 [Github](https://github.com/alexislozano/pokedex/tree/article-4) 上查看
