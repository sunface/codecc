> 原文链接: https://alexis-lozano.com/hexagonal-architecture-in-rust-5/
>
> 翻译：[trdthg](https://github.com/trdthg)
>
> 选题：[trdthg](https://github.com/trdthg)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

# 2021-09-12 - Rust 六边形架构 #5 - 其他用例

上次，我们做了一些重构。不知何故，我们的客户对我们很生气……我的意思是，他应该高兴，代码现在比以前更干净了。

在吃完一块蛋糕之后 (显然，这是我们应得的)，让我们继续实现剩下的用例。这篇文章可能会有点长，但是，嘿，如果你不想阅读整个过程，代码在 github 上 : )
我们将实现的用例是：

- 获取所有宝可梦
- 查询一只宝可梦
- 删除一只宝可梦

## 获取所有宝可梦

像往常一样，我们将从测试开始。让我们首先创建一个新的用例 文件：_domain/fetch\_all\_pokemons.rs_：

```rs
// domain/mod.rs
pub mod fetch_all_pokemons;
```

让我们考虑一下这个用例有哪些可能的结果？

1. 成功了，我们得到了宝可梦
2. 存储库中发生了未知错误，我们无法得到我们的宝可梦

让我们先从错误案例开始：

```rs
#[cfg(test)]
mod tests {
    use super::*;
    use crate::repositories::pokemon::InMemoryRepository;

    #[test]
    fn it_should_return_an_unknown_error_when_an_unexpected_error_happens() {
        let repo = Arc::new(InMemoryRepository::new().with_error());

        let res = execute(repo);

        match res {
            Err(Error::Unknown) => {}
            _ => unreachable!(),
        };
    }
}
```

如您所见，我仍然没有写一行代码，只写了测试。当然，现在还不能通过编译。这里有两件事很有趣：

- 这里的测试和创建宝可梦那里几乎一样
- 这里不需要 repo 之外的其他参数

接下来让我们添加一些必要的代码让测试通过：

```rs
use crate::repositories::pokemon::Repository;
use std::sync::Arc;

pub enum Error {
    Unknown,
}

pub fn execute(repo: Arc<dyn Repository>) -> Result<(), Error> {
    Err(Error::Unknown)
}
```

接着继续添加下一个测试：

```rs
#[test]
fn it_should_return_all_the_pokemons_ordered_by_increasing_number_otherwise() {
    let repo = Arc::new(InMemoryRepository::new());
    repo.insert(
        PokemonNumber::pikachu(),
        PokemonName::pikachu(),
        PokemonTypes::pikachu(),
    )
    .ok();
    repo.insert(
        PokemonNumber::charmander(),
        PokemonName::charmander(),
        PokemonTypes::charmander(),
    )
    .ok();

    let res = execute(repo);

    match res {
        Ok(res) => {
            assert_eq!(res[0].number, u16::from(PokemonNumber::charmander()));
            assert_eq!(res[0].name, String::from(PokemonName::charmander()));
            assert_eq!(res[0].types, Vec::<String>::from(PokemonTypes::charmander()));
            assert_eq!(res[1].number, u16::from(PokemonNumber::pikachu()));
            assert_eq!(res[1].name, String::from(PokemonName::pikachu()));
            assert_eq!(res[1].types, Vec::<String>::from(PokemonTypes::pikachu()));
        }
        _ => unreachable!(),
    };
}
```

在测试里，我按照 `number` 递减的顺序插入了宝可梦，所以存储库中宝可梦的排序应该是确定的。当存储库没有输出错误时，我们应该能从用例中得到一个
`Vec<Response>`。现在我们要为用例加入 `Response`，并编写一部分用例函数的内容：

```rs
pub struct Response {
    pub number: u16,
    pub name: String,
    pub types: Vec<String>,
}

pub fn execute(repo: Arc<dyn Repository>) -> Result<Vec<Response>, Error> {
    match repo.fetch_all() {
        Ok(pokemons) => Ok(pokemons
            .into_iter()
            .map(|p| Response {
                number: u16::from(p.number),
                name: String::from(p.name),
                types: Vec::<String>::from(p.types),
            })
            .collect::<Vec<Response>>()),
        Err(FetchAllError::Unknown) => Err(Error::Unknown),
    }
}
```

然后再为存储库添加并实现 `fetch_all` 方法：

```rs
pub enum FetchAllError {
    Unknown,
}

pub trait Repository: Send + Sync {
    fn fetch_all(&self) -> Result<Vec<Pokemon>, FetchAllError>;
}

impl Repository for InMemoryRepository {
    fn fetch_all(&self) -> Result<Vec<Pokemon>, FetchAllError> {
        if self.error {
            return Err(FetchAllError::Unknown);
        }

        let lock = match self.pokemons.lock() {
            Ok(lock) => lock,
            _ => return Err(FetchAllError::Unknown),
        };

        let mut pokemons = lock.to_vec();
        pokemons.sort_by(|a, b| a.number.cmp(&b.number));
        Ok(pokemons)
    }
}
```

为了实现宝可梦按照 `number` 排序，`PokemonNumber` 需要实现 `PartialEq`、 `Eq`、`PartialOrd`、`Ord`
这 4 个 Trait

```rs
#[derive(Clone, PartialEq, Eq, PartialOrd, Ord)]
pub struct PokemonNumber(u16);
```

运行 c`argo test`：

```
cargo test
running 6 tests
...
it_should_return_an_unknown_error_when_an_unexpected_error_happens ... ok
it_should_return_all_the_pokemons_ordered_by_increasing_number_otherwise ... ok
```

我们已经完成了这个用例 :) 现在让我们快速实现 api。在 _api/mod.rs_ 中，添加以下内容：

```rs
mod fetch_all_pokemons;

// in the router macro
(GET) (/) => {
    fetch_all_pokemons::serve(repo.clone())
}
```

接着实现具体的handler api/fetch_all_pokemons.rs：

```rs
use crate::api::Status;
use crate::domain::fetch_all_pokemons;
use crate::repositories::pokemon::Repository;
use rouille;
use serde::Serialize;
use std::sync::Arc;

#[derive(Serialize)]
struct Response {
    number: u16,
    name: String,
    types: Vec<String>,
}

pub fn serve(repo: Arc<dyn Repository>) -> rouille::Response {
    match fetch_all_pokemons::execute(repo) {
        Ok(res) => rouille::Response::json(
            &res.into_iter()
                .map(|p| Response {
                    number: p.number,
                    name: p.name,
                    types: p.types,
                })
                .collect::<Vec<Response>>(),
        ),
        Err(fetch_all_pokemons::Error::Unknown) => {
            rouille::Response::from(Status::InternalServerError)
        }
    }
}
```

现在您可以运行 `cargo run`，并使用你最喜欢的 HTTP 客户端 (curl、postman、...)。通过在 `/` 上发送 POST
请求来创建一些宝可梦。然后您可以使用浏览器访问 http://localhost:8000。这是我得到的：

```json
[
  {
    "number": 4,
    "name": "Charmander",
    "types": [
      "Fire"
    ]
  },
  {
    "number": 25,
    "name": "Pikachu",
    "types": [
      "Electric"
    ]
  }
]
```

## 查询一只宝可梦

第二个用例！现在我们想通过给系统一个宝可梦的编号来获取一个宝可梦。让我们创建一个新的用例文件 _domain/fetch\_pokemon.rs_ ：

```rs
// domain/mod.rs
pub mod fetch_pokemon;
```

让我们思考一下用例运行时可能遇到什么情况。

1. 存储库可能会引发未知错误。
2. 请求参数可能是不合法的。
3. 用户给定的编号可能对应没有宝可梦。
4. 获得宝可梦成功了

让我们从未知错误的测试开始。新建 _domain/fetch\_pokemon.rs_ 并添加以下内容：

```rs
use crate::domain::entities::PokemonNumber;
use std::sync::Arc;

#[cfg(test)]
mod tests {
    use super::*;
    use crate::repositories::pokemon::InMemoryRepository;

    #[test]
    fn it_should_return_an_unknown_error_when_an_unexpected_error_happens() {
        let repo = Arc::new(InMemoryRepository::new().with_error());
        let req = Request::new(PokemonNumber::pikachu());

        let res = execute(repo, req);

        match res {
            Err(Error::Unknown) => {}
            _ => unreachable!(),
        };
    }
}
```

接着，为测试实现必要的类型和函数：

```rs
pub struct Request {
    pub number: u16,
}

pub enum Error {
    Unknown
}
```

为了让测试更清晰，我们还为 `Request` 实现了一个 `new` 方法，当然只在测试时使用：

```rs
#[cfg(test)]
mod tests {
    ...

    impl Request {
        fn new(number: PokemonNumber) -> Self {
            Self {
                number: u16::from(number),
            }
        }
    }
}
```

最后，再实现一个只用满足这项测试的 `execute` 函数即可：

```rs
use crate::repositories::pokemon::Repository;

pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Result<(), Error> {
    Err(Error::Unknown)
}
```

让我们运行 `cargo test fetch_pokemon`:

```
test it_should_return_an_unknown_error_when_an_unexpected_error_happens ... ok
```

好的。我们可以开始下一个测试了。让我们来看一下请求格式错误的情况：

```rs
#[test]
fn it_should_return_a_bad_request_error_when_request_is_invalid() {
    let repo = Arc::new(InMemoryRepository::new());
    let req = Request::new(PokemonNumber::bad());

    let res = execute(repo, req);

    match res {
        Err(Error::BadRequest) => {}
        _ => unreachable!(),
    };
}
```

首先，让我们在 `PokemonNumber` 中创建 `bad` 函数。在 _domain/entities.rs_ 中添加以下内容：

```rs
#[cfg(test)]
impl PokemonNumber {
    ...
    pub fn bad() -> Self {
        Self(0)
    }
}
```

接着，在 `Error` 中添加 `Badrequest` 类型与之对应：

```rs
pub enum Error {
    BadRequest,
    ...
}
```

现在已经能够通过编译，但是测试尚且不能通过，实际上，我们的 `execute` 函数暂时只能返回 `Unknown` 类型：

```rs
use std::convert::TryFrom;

pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Result<(), Error> {
    match PokemonNumber::try_from(req.number) {
        Ok(number) => Err(Error::Unknown),
        _ => Err(Error::BadRequest),
    }
}
```

现在两个测试都通过了！我们现在可以进行第三个测试：在存储库中找不到宝可梦时，应该返回 `NotFound`:

```rs
#[test]
fn it_should_return_a_not_found_error_when_the_repo_does_not_contain_the_pokemon() {
    let repo = Arc::new(InMemoryRepository::new());
    let req = Request::new(PokemonNumber::pikachu());

    let res = execute(repo, req);

    match res {
        Err(Error::NotFound) => {}
        _ => unreachable!(),
    };
}
```

同样，在 `Error` 中补充 `NotFound` 错误类型：

```rs
pub enum Error {
    ...
    NotFound,
    ...
}
```

像之前一样，测试没有通过。我们需要在 `execute` 函数中处理对应的错误类型：

```rs
use crate::repositories::pokemon::{FetchOneError, ...};

Ok(number) => match repo.fetch_one(number) {
    Ok(_) => Ok(()),
    Err(FetchOneError::NotFound) => Err(Error::NotFound),
    Err(FetchOneError::Unknown) => Err(Error::Unknown),
}
```

接着在 _repositories/pokemon.rs_ 中补充对应的方法和错误类型：

```rs
pub enum FetchOneError {
    NotFound,
    Unknown,
}

pub trait Repository: Send + Sync {
    ...
    fn fetch_one(&self, number: PokemonNumber) -> Result<(), FetchOneError>;
}

impl Repository for InMemoryRepository {
    ...
    fn fetch_one(&self, number: PokemonNumber) -> Result<(), FetchOneError> {
        if self.error {
            return Err(FetchOneError::Unknown);
        }

        Err(FetchOneError::NotFound)
    }
}
```

Et voilà，测试通过了！最后去处理获取成功的测试吧：

```rs
#[cfg(test)]
mod tests {
    use crate::domain::entities::{PokemonName, PokemonTypes};

    #[test]
    fn it_should_return_the_pokemon_otherwise() {
        let repo = Arc::new(InMemoryRepository::new());
        repo.insert(
            PokemonNumber::pikachu(),
            PokemonName::pikachu(),
            PokemonTypes::pikachu(),
        )
        .ok();
        let req = Request::new(PokemonNumber::pikachu());

        let res = execute(repo, req);

        match res {
            Ok(res) => {
                assert_eq!(res.number, u16::from(PokemonNumber::pikachu()));
                assert_eq!(res.name, String::from(PokemonName::pikachu()));
                assert_eq!(res.types, Vec::<String>::from(PokemonTypes::pikachu()));
            }
            _ => unreachable!(),
        };
    }
}
```

首先创建 `Response` 类型，并改变 `execute` 的返回值类型：

```rs
pub struct Response {
    pub number: u16,
    pub name: String,
    pub types: Vec<String>,
}

pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Result<Response, Error> {
```

现在，我们去处理 `Ok(_)` 的部分：

```rs
use crate::domain::entities::{Pokemon, ...};

Ok(Pokemon {
    number,
    name,
    types,
}) => Ok(Response {
    number: u16::from(number),
    name: String::from(name),
    types: Vec::<String>::from(types),
})
```

最后修改存储库的 `fetch_one` 函数：

```rs
pub trait Repository: Send + Sync {
    fn fetch_one(&self, number: PokemonNumber) -> Result<Pokemon, FetchOneError>;
}

impl Repository for InMemoryRepository {
    fn fetch_one(&self, number: PokemonNumber) -> Result<Pokemon, FetchOneError> {
        if self.error {
            return Err(FetchOneError::Unknown);
        }

        let lock = match self.pokemons.lock() {
            Ok(lock) => lock,
            _ => return Err(FetchOneError::Unknown),
        };

        match lock.iter().find(|p| p.number == number) {
            Some(pokemon) => Ok(pokemon.clone()),
            None => Err(FetchOneError::NotFound),
        }
    }
}
```

所有的测试都通过了现在：

```
test it_should_return_a_not_found_error_when_the_repo_does_not_contain_the_pokemon ... ok
test it_should_return_the_pokemon_otherwise ... ok
test it_should_return_a_bad_request_error_when_request_is_invalid ... ok
test it_should_return_an_unknown_error_when_an_unexpected_error_happens ... ok
```

让我们现在向 API 中前加一个新路由：

```rs
mod fetch_pokemon;

// in the router macro
(GET) (/{number: u16}) => {
    fetch_pokemon::serve(repo.clone(), number)
}
```

接着再 _api/fetch\_pokemon.rs_ 创建对应的 `serve` 函数：

```rs
use crate::api::Status;
use crate::domain::fetch_pokemon;
use crate::repositories::pokemon::Repository;
use rouille;
use serde::Serialize;
use std::sync::Arc;

#[derive(Serialize)]
struct Response {
    number: u16,
    name: String,
    types: Vec<String>,
}

pub fn serve(repo: Arc<dyn Repository>, number: u16) -> rouille::Response {
    let req = fetch_pokemon::Request { number };
    match fetch_pokemon::execute(repo, req) {
        Ok(fetch_pokemon::Response {
            number,
            name,
            types,
        }) => rouille::Response::json(&Response {
            number,
            name,
            types,
        }),
        Err(fetch_pokemon::Error::BadRequest) => rouille::Response::from(Status::BadRequest),
        Err(fetch_pokemon::Error::NotFound) => rouille::Response::from(Status::NotFound),
        Err(fetch_pokemon::Error::Unknown) => rouille::Response::from(Status::InternalServerError),
    }
}
```

现在您可以运行应用程序，打开您的 HTTP 客户端并将一些宝可梦添加到存储库中。然后，您应该能够通过在网址末尾添加宝可梦的数量来一一获取它们。
例如，在我的计算机上，http://localhost:8000/25 上的 GET 请求返回了：

```json
{
  "number": 25,
  "name": "Pikachu",
  "types": [
    "Electric"
  ]
}
```

## 删除一只宝可梦

我保证，我们很快就会完成。删除宝可梦是我们的最后一个用例。这个用例的结果有四种可能：

- 成功
- 请求格式错误
- 没有找到对应的宝可梦
- 未知错误

您应该已经注意到，它们与获取 Pokemon 用例中的完全相同。我会解释更快一些。现在就开始了。让我们直接编写所有测试：

```rs
// domain/mod.rs
pub mod delete_pokemon;

// domain/delete_pokemon.rs
#[cfg(test)]
mod tests {
    use super::*;
    use crate::domain::entities::{PokemonName, PokemonTypes};
    use crate::repositories::pokemon::InMemoryRepository;

    #[test]
    fn it_should_return_an_unknown_error_when_an_unexpected_error_happens() {
        let repo = Arc::new(InMemoryRepository::new().with_error());
        let req = Request::new(PokemonNumber::pikachu());

        let res = execute(repo, req);

        match res {
            Err(Error::Unknown) => {}
            _ => unreachable!(),
        };
    }

    #[test]
    fn it_should_return_a_bad_request_error_when_request_is_invalid() {
        let repo = Arc::new(InMemoryRepository::new());
        let req = Request::new(PokemonNumber::bad());

        let res = execute(repo, req);

        match res {
            Err(Error::BadRequest) => {}
            _ => unreachable!(),
        };
    }

    #[test]
    fn it_should_return_a_not_found_error_when_the_repo_does_not_contain_the_pokemon() {
        let repo = Arc::new(InMemoryRepository::new());
        let req = Request::new(PokemonNumber::pikachu());

        let res = execute(repo, req);

        match res {
            Err(Error::NotFound) => {}
            _ => unreachable!(),
        };
    }

    #[test]
    fn it_should_return_ok_otherwise() {
        let repo = Arc::new(InMemoryRepository::new());
        repo.insert(
            PokemonNumber::pikachu(),
            PokemonName::pikachu(),
            PokemonTypes::pikachu(),
        )
        .ok();
        let req = Request::new(PokemonNumber::pikachu());

        let res = execute(repo, req);

        match res {
            Ok(()) => {},
            _ => unreachable!(),
        };
    }

    impl Request {
        fn new(number: PokemonNumber) -> Self {
            Self {
                number: u16::from(number),
            }
        }
    }
}
```

除了成功情况下的测试，其他的测试基本上是对 _domain/fetch\_pokemon.rs_ 的复制粘贴。接下来是类型：

```rs
pub struct Request {
    pub number: u16,
}

pub enum Error {
    BadRequest,
    NotFound,
    Unknown,
}
```

我们不需要定义 `Response` 类型，因为在 `Ok` 情况下我们不会返回任何内容。让我们定义 `execute` 函数：

```rs
use crate::domain::entities::PokemonNumber;
use crate::repositories::pokemon::{DeleteError, Repository};
use std::convert::TryFrom;
use std::sync::Arc;

pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Result<(), Error> {
    match PokemonNumber::try_from(req.number) {
        Ok(number) => match repo.delete(number) {
            Ok(()) => Ok(()),
            Err(DeleteError::NotFound) => Err(Error::NotFound),
            Err(DeleteError::Unknown) => Err(Error::Unknown),
        },
        _ => Err(Error::BadRequest),
    }
}
```

太好了! 现在我们只剩下实现存储库了，我们很快就会完成：

```rs
pub enum DeleteError {
    NotFound,
    Unknown,
}

pub trait Repository: Send + Sync {
    fn delete(&self, number: PokemonNumber) -> Result<(), DeleteError>;
}

impl Repository for InMemoryRepository {
    fn delete(&self, number: PokemonNumber) -> Result<(), DeleteError> {
        if self.error {
            return Err(DeleteError::Unknown);
        }

        let mut lock = match self.pokemons.lock() {
            Ok(lock) => lock,
            _ => return Err(DeleteError::Unknown),
        };

        let index = match lock.iter().position(|p| p.number == number) {
            Some(index) => index,
            None => return Err(DeleteError::NotFound),
        };

        lock.remove(index);
        Ok(())
    }
}
```

运行测试：

```
test it_should_return_a_bad_request_error_when_request_is_invalid ... ok
test it_should_return_a_not_found_error_when_the_repo_does_not_contain_the_pokemon ... ok
test it_should_return_an_unknown_error_when_an_unexpected_error_happens ... ok
test it_should_return_ok_otherwise ... ok
We can now add the new route in our API. Let's add the following to api/mod.rs:
```

现在我们可以在 API 中添加新路由：

```rs
mod delete_pokemon;

// in the router macro
(DELETE) (/{number: u16}) => {
    delete_pokemon::serve(repo.clone(), number)
}

enum Status {
    Ok,
    ...
}
impl From<Status> for rouille::Response {
    fn from(status: Status) -> Self {
        let status_code = match status {
            Status::Ok => 200,
            ...
```

我在 `Status` 中补充一个 OK 类型用来表示没有相应体的 200 状态码，现在添加对应的 `serve` 函数：

```rs
use crate::api::Status;
use crate::domain::delete_pokemon;
use crate::repositories::pokemon::Repository;
use rouille;
use std::sync::Arc;

pub fn serve(repo: Arc<dyn Repository>, number: u16) -> rouille::Response {
    let req = delete_pokemon::Request { number };
    match delete_pokemon::execute(repo, req) {
        Ok(()) => rouille::Response::from(Status::Ok),
        Err(delete_pokemon::Error::BadRequest) => rouille::Response::from(Status::BadRequest),
        Err(delete_pokemon::Error::NotFound) => rouille::Response::from(Status::NotFound),
        Err(delete_pokemon::Error::Unknown) => rouille::Response::from(Status::InternalServerError),
    }
}
```

通过使用 HTTP 客户端，现在你应该能够删除宝可梦了 :)

## 总结

我希望它不会太长......但现在我们的客户很高兴！耶！下一次，我们将为我们的用例实现另一个前端。现在我们只有一个 HTTP API，如果还能有一个 CLI
来管理我们的宝可梦就好了 :)

代码可以在 [Github](https://github.com/alexislozano/pokedex/tree/article-5) 上查看
