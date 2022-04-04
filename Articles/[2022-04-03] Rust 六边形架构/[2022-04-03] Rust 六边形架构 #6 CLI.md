> 原文链接: https://alexis-lozano.com/hexagonal-architecture-in-rust-6/
>
> 翻译：[trdthg](https://github.com/trdthg)
>
> 选题：[trdthg](https://github.com/trdthg)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

# 2021-10-09 - Rust 六边形架构 #6 - CLI

嘿，好久不见！上次，我们实现了剩余的用例，并将它们连接到我们的 API。现在，我想添加另一种方式来使用我们的程序。我们将使用 CLI。 CLI 是
Command Line Interface (命令行接口) 的意思，它只是一个缩写词，意思是：“让我们通过终端使用这个程序”。

## 搭建脚手架

构建 CLI 意味着我们需要在项目中添加新的依赖和一个新文件夹。让我们从添加依赖开始。 我们需要一种方法，再运行之前需要提示用户是运行 CLI 还是 HTTP
API。我们将使用 `clap` 添加命令行开关让用户选择启动方式，同时还会使用 `dialoguer` 去创建提示信息。

打开 Cargo.toml，并添加：

```toml
[dependencies]
...
clap = "2.33.3"
dialoguer = "0.8.0"
```

当你再读这篇文章时可以将依赖换为最新的。现在让我们添加一些命令行开关，打开 _main.rs_:

```rs
#[macro_use]
extern crate clap;

use clap::{App, Arg};
And then we can use it:

fn main() {
    let repo = Arc::new(InMemoryRepository::new());

    let matches = App::new(crate_name!())
        .version(crate_version!())
        .author(crate_authors!())
        .arg(Arg::with_name("cli").long("cli").help("Runs in CLI mode"))
        .get_matches();

    match matches.occurrences_of("cli") {
        0 => api::serve("localhost:8000", repo),
        _ => unreachable!(),
    }
}
```

所以，首先我们创建了存储库。然后我们创建一个处理 CLI 的 clap 应用程序。如果在运行程序时添加 `--cli` 标志，程序现在将
panic。如果没有添加，就会运行 API。 正如我之前所说，`clap` 能让我们快速创建一个 CLI。您可以通过运行来尝试：

```rs
cargo run -- --help

pokedex 0.1.0
Alexis Lozano <alexis.pascal.lozano@gmail.com>

USAGE:
    pokedex [FLAGS]

FLAGS:
        --cli        Runs in CLI mode
    -h, --help       Prints help information
    -V, --version    Prints version information
```

非常不错是吧 :)

现在我们要把 `unreachable!()` 替换为 `cli::run(repo)`。我们现在要创建一个 `cli` 模块，所有和 cli
相关的代码都会在该模块里。在 _main.rs_ 里引入模块：

```rs
mod cli;
```

接着让我们创建 _src/cli_ 文件夹，并在其中添加一个 _mod.rs_ 文件。在 _cli/mod.rs_ 中添加以下代码：

```rs
use crate::repositories::pokemon::Repository;
use std::sync::Arc;

pub fn run(repo: Arc<dyn Repository>) {}
```

现在运行 `cargo run -- --cli` 应该不会 panic 了。

接着让我们创建一个循环：

```rs
use dialoguer::{theme::ColorfulTheme, Select};

pub fn run(repo: Arc<dyn Repository>) {
    loop {
        let choices = [
            "Fetch all Pokemons",
            "Fetch a Pokemon",
            "Create a Pokemon",
            "Delete a Pokemon",
            "Exit",
        ];
        let index = match Select::with_theme(&ColorfulTheme::default())
            .with_prompt("Make your choice")
            .items(&choices)
            .default(0)
            .interact()
        {
            Ok(index) => index,
            _ => continue,
        };

        match index {
            4 => break,
            _ => continue,
        };
    }
}
```

我们列出了所有用户能够运行的命令。如果用户选择 `Exit`，程序就会退出。否则，程序暂时什么都不会做。别担心，我们马上就会实现其他的命令。

## 创建一个宝可梦

让我们从创建开始。如果我们能够有方法向存储库中添加宝可梦，后面的测试就更容易做了：

```rs
match index {
    2 => create_pokemon::run(repo.clone()),
    ...
};
```

现在我们需要创建一个新的模块：

```rs
mod create_pokemon;
```

好了，在 _cli/create_pokemon.rs_ 中填加上对应的函数签名：

```rs
use crate::repositories::pokemon::Repository;
use std::sync::Arc;

pub fn run(repo: Arc<dyn Repository>) {}
```

为了创建一个宝可梦，CLI 需要向用户询问宝可梦的 编号、名称和类型。为了方便这个过程，并且提高提示信息的复用性，我们会为这些信息分别实现各自的提示函数：

```rs
use crate::cli::{prompt_name, prompt_number, prompt_types};

pub fn run(repo: Arc<dyn Repository>) {
    let number = prompt_number();
    let name = prompt_name();
    let types = prompt_types();
}
```

接着在 _cli/mod.rs_ 中实现:

```rs
use dialoguer::{..., Input, MultiSelect};

pub fn prompt_number() -> Result<u16, ()> {
    match Input::new().with_prompt("Pokemon number").interact_text() {
        Ok(number) => Ok(number),
        _ => Err(()),
    }
}

pub fn prompt_name() -> Result<String, ()> {
    match Input::new().with_prompt("Pokemon name").interact_text() {
        Ok(name) => Ok(name),
        _ => Err(()),
    }
}

pub fn prompt_types() -> Result<Vec<String>, ()> {
    let types = ["Electric", "Fire"];
    match MultiSelect::new()
        .with_prompt("Pokemon types")
        .items(&types)
        .interact()
    {
        Ok(indexes) => Ok(indexes
            .into_iter()
            .map(|index| String::from(types[index]))
            .collect::<Vec<String>>()),
        _ => Err(()),
    }
}
```

> 提示：多选框按空格选中

如你所见，所有的提示都可能失败，让我们回到 _cli/create\_pokemon.rs_，有了用户的输入信息，我们可以将它封装为用例中需要的
`Request` 结构体：

```rs
use crate::domain::create_pokemon;

pub fn run(repo: Arc<dyn Repository>) {
    ...
    let req = match (number, name, types) {
        (Ok(number), Ok(name), Ok(types)) => create_pokemon::Request {
            number,
            name,
            types,
        },
        _ => {
            println!("An error occurred during the prompt");
            return;
        }
    };
```

当发生输入错误时，我们会退回到主菜单。现在有了 `Request` 结构体，我们就能调用创建宝可梦的用例了：

```rs
pub fn run(repo: Arc<dyn Repository>) {
    ...
    match create_pokemon::execute(repo, req) {
        Ok(res) => {},
        Err(create_pokemon::Error::BadRequest) => println!("The request is invalid"),
        Err(create_pokemon::Error::Conflict) => println!("The Pokemon already exists"),
        Err(create_pokemon::Error::Unknown) => println!("An unknown error occurred"),
    };
}
```

我们处理了所有的错误类型，每种错误都会反馈到用户的终端上。当用例执行成功时，我们会返回一个 `Response`：

```rs
#[derive(Debug)]
struct Response {
    number: u16,
    name: String,
    types: Vec<String>,
}

pub fn run(repo: Arc<dyn Repository>) {
    ...
    match create_pokemon::execute(repo, req) {
        Ok(res) => println!(
            "{:?}",
            Response {
                number: res.number,
                name: res.name,
                types: res.types,
            }
        ),
        ...
    };
}
```

好了，让我们测试一下！打开终端，以命令行模式运行：

```rs
✔ Make your choice · Create a Pokemon
Pokemon number: 25
Pokemon name: Pikachu
Pokemon types: Electric
Response { number: 25, name: "Pikachu", types: ["Electric"] }
Great! First command implemented, let's do the next one!
```

## 获取所有宝可梦

现在我们去实现获取所有宝可梦！首先我们在对应的 `index` 添加相应的处理函数：

```rs
match index {
    0 => fetch_all_pokemons::run(repo.clone()),
    ...
};
```

创建模块

```rs
mod fetch_all_pokemons;
```

现在实现具体的功能：

```rs
use crate::domain::fetch_all_pokemons;
use crate::repositories::pokemon::Repository;
use std::sync::Arc;

#[derive(Debug)]
struct Response {
    number: u16,
    name: String,
    types: Vec<String>,
}

pub fn run(repo: Arc<dyn Repository>) {
    match fetch_all_pokemons::execute(repo) {
        Ok(res) => res.into_iter().for_each(|p| {
            println!(
                "{:?}",
                Response {
                    number: p.number,
                    name: p.name,
                    types: p.types,
                }
            );
        }),
        Err(fetch_all_pokemons::Error::Unknown) => println!("An unknown error occurred"),
    };
}
```

我已经详细的解释过第一个例子，这里就没必要一步一步说明了。这个命令不需要任何参赛，所以也不需要任何提示信息，我们只需要从存储库拿到结果，依次打印即可。

## 查询一个宝可梦

和之前一样，创建一个模块：

```rs
mod fetch_pokemon;

...

match index {
    ...
    1 => fetch_pokemon::run(repo.clone()),
    ...
};
```

创建对应的处理函数

```rs
use crate::cli::prompt_number;
use crate::domain::fetch_pokemon;
use crate::repositories::pokemon::Repository;
use std::sync::Arc;

#[derive(Debug)]
struct Response {
    number: u16,
    name: String,
    types: Vec<String>,
}

pub fn run(repo: Arc<dyn Repository>) {
    let number = prompt_number();

    let req = match number {
        Ok(number) => fetch_pokemon::Request { number },
        _ => {
            println!("An error occurred during the prompt");
            return;
        }
    };
    match fetch_pokemon::execute(repo, req) {
        Ok(res) => println!(
            "{:?}",
            Response {
                number: res.number,
                name: res.name,
                types: res.types,
            }
        ),
        Err(fetch_pokemon::Error::BadRequest) => println!("The request is invalid"),
        Err(fetch_pokemon::Error::NotFound) => println!("The Pokemon does not exist"),
        Err(fetch_pokemon::Error::Unknown) => println!("An unknown error occurred"),
    }
}
```

这里，我们先向用户询问了编号，如果获取失败了就输出错误信息。接着构建 Request 结构体，调用用例，成功就打印结果，否则打印错误信息。

# 删除一个宝可梦

最后一个命令！

```rs
mod delete_pokemon;

...

match index {
    ...
    3 => delete_pokemon::run(repo.clone()),
    ...
};
```

创建新模块：

```rs
use crate::cli::prompt_number;
use crate::domain::delete_pokemon;
use crate::repositories::pokemon::Repository;
use std::sync::Arc;

pub fn run(repo: Arc<dyn Repository>) {
    let number = prompt_number();

    let req = match number {
        Ok(number) => delete_pokemon::Request { number },
        _ => {
            println!("An error occurred during the prompt");
            return;
        }
    };
    match delete_pokemon::execute(repo, req) {
        Ok(()) => println!("The Pokemon has been deleted"),
        Err(delete_pokemon::Error::BadRequest) => println!("The request is invalid"),
        Err(delete_pokemon::Error::NotFound) => println!("The Pokemon does not exist"),
        Err(delete_pokemon::Error::Unknown) => println!("An unknown error occurred"),
    }
}
```

和查询一个宝可梦基本一样，只不过不需要返回信息。

## 总结

现在您无需直接发送 HTTP 请求即可在终端中享受使用程序 : D
但是你知道我们缺少什么吗？一个真正存储我们的宝可梦的地方。目前，它们只存在于内存中，因此每次我们运行程序时存储库都是空的。从 CLI 创建我们的宝可梦然后从
API 获取它们会很酷 : ) 下一篇将是本系列的最后一篇文章：实现一个长期保存的存储库。

代码可以在 [Github](https://github.com/alexislozano/pokedex/tree/article-6) 上查看
