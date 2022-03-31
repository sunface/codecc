 # rust 的 ``Option`` 和 ``Result``
 
 在 Rust 中，完全理解枚举 ``Option`` 和枚举 ``Result`` ，对于理解可选数值和错误处理非常重要。在这篇文章中，我们将深入学习这两方面的内容。

## 概述

想理解 ``Option`` 和 ``Result``，理解如下内容是很重要的：

* Rust 中的 enum
* 匹配 enum 变量
* Rust 中的 prelude

### Rust 中的 enum

有很好的原因去使用 enum ，除此之外，它们还有助于安全的输入处理，并通过为变量集合命名，为类型添加上下文。 在 Rust 中，``Option`` 和 ``Result`` 都是 enum 类型。 Rust 中的 enum 是非常灵活的，它可以包括很多数据类型，例如 tuple ，struct 等等。另外，你还可以在 enums 上实现方法。

``Option`` 和 ``Result`` 理解起来非常简单。我们首先来看一个 enum 的例子：

```rust
enum Example {
    This,
    That,
}

let this = Example::This;
let that = Example::That;
```

在上面，我们定义了一个 enum 命名为 ``Example``，这个 enum 包括两个变量  ``This`` 和 ``That`` 。然后，我们创建了2个 enum 的实例变量  ``This`` 和 ``That`` 。他们使用其自己的字段被创建。重要的是一个 enum 实例经常使用多个字段中的一个。当你使用 struct 定义字段的时候，你可以使用其定义的全部可能的字段。一个 enum 是不一样的，因为你只可以使用其中的一个变量字段。

### 显示 enum 变量

默认的，enum 变量是不能打印到屏幕的。 在我们定义的 enum 上，通过使用 ``strum_macros`` 可以很容易的派生 ‘Display’， 在 enum 上我们使用 ``#[derive(Display)] `` ：

```rust
use strum_macros::Display;

#[derive(Display)]
enum Example {
    This,
    That,
}

let this = Example::This;
let that = Example::That;

println!("Example::This contains: {}", this);
println!("Example::That contains: {}", that);
```

现在，我们可以在屏幕上使用 println！显示 enum 变量的数值：

```text
Example::This contains: This
Example::That contains: That
```

### 匹配 enum 变量

使用 match 关键字，我们可以在 enum 上实现模式匹配。下面函数使用Example 枚举做为参数：

```rust
fn matcher(x: Example) {
    match x {
        Example::This => println!("We got This."),
        Example::That => println!("We got That."),
    }
}
```

我们可以给这个 matcher 函数传递一个 Example 的数值，函数内部的 ``match`` 将决定什么内容会被打印到屏幕：

```rust
matcher(Example::This);
matcher(Example::That);
```

执行上面代码将会打印如下内容：

```text
We got This.
We got That
```

### Rust 中的 prelude

这里描述的 Rust prelude 是每个程序的一部分。它是自动导入到每个Rust程序中的内容列表。prelude 中的大多数内容都是经常使用的 <ruby>特征<rt>trait</rt></ruby> 。除此之外，我们还发现以下两项：

* std::``Option``::``Option``::{self, Some, None}
* std::``Result``::``Result``::{self, Ok, Err}

第一个就是 ``Option`` enum，表示值的存在或不存在的类型。第二个是 ``Result`` enum，被描述为一种可能成功或失败的函数返回类型。

因为这些类型非常常用，所以也会预先加载它们。让我们更详细地讨论这两种类型。

## ``Option``

由 prelude 引入的 [Option](https://doc.rust-lang.org/std/``Option``/index.html) 我们不需要动一根手指就可以使用。 ``Option`` 定义如下：

```rust
pub enum Option<T> {
    None,
    Some(T),
}
```

以上我们可以看到 ``Option<T>`` 是一个<ruby>枚举<rt>enum</rt></ruby>并包含两个变量：``None`` 和 ``Some(T) ``。就其使用方式而言，``None ``可以被认为是 ``无`` ，``Some(T) ``可以被认为是“某物”。对于那些刚开始使用 Rust 的人来说，一件关键的事情不是很明显，那就是 T 这件符号。这个 T 告诉我们这是一个泛型。

### ``Option`` 是 "T" 型上的泛型

<ruby>枚举<rt>enum</rt></ruby>是 "T" 型上的泛型。字符 "T" 可以是任何的字符串, 只是 “T” 被用作涉及1个泛型的约定。

那么，当我们说 “枚举是T型上的泛型” 是什么意思呢？这意味着我们可以将其用于任何类型。一旦我们开始使用<ruby>枚举<rt>enum</rt></ruby>，我们就可以（而且必须）用具体类型替换 “T” 。这可以是任何类型，如下所示：

```rust
let a_str: Option<&str> = Some("a str");
let a_string: Option<String> = Some(String::from("a String"));
let a_float: Option<f64> = Some(1.1);
let a_vec: Option<Vec<i32>> = Some(vec![0, 1, 2, 3]);

#[derive(Debug)]
struct Person {
    name: String,
    age: i32,
}

let marie = Person {
    name: String::from("Marie"),
    age: 2,
};

let a_person: Option<Person> = Some(marie);
let maybe_someone: Option<Person> = None;

println!(
    "{:?}\n{:?}\n{:?}\n{:?}\n{:?}\n{:?}",
    a_str, a_string, a_float, a_vec, a_person, maybe_someone
);
```

以上代码输出如下：

```text
Some("a str")
Some("a String")
Some(1.1)
Some([0, 1, 2, 3])
Some(Person { name: "Marie", age: 2 })
None
```

代码告诉我们，枚举可以是泛型，不但可以是标准类型，也可以是自定义类型。此外，当我们将枚举定义为 x 类型时，它仍然可以包含变量 “None” 。因此，这个 ``Option`` 是这样说的：

```text
This can be of a type T value, which can be anything really, or it can be nothing. 
```

### ``Option`` 上的匹配
既然 Rust 不使用异常或者空数值，你会看到 ``Option``（我们将在后面了解 ``Result`` ）被广泛使用。

由于该 ``Option`` 是一个<ruby>枚举<rt>enum</rt></ruby>，我们可以使用模式匹配以自己的方式处理每个变量：

```rust
let something: Option<&str> = Some("a String"); // Some("a String")
let nothing: Option<&str> = None;   // None

match something {
    Some(text) => println!("We go something: {}", text),
    None => println!("We got nothing."),
}

match nothing {
    Some(something_else) => println!("We go something: {}", something_else),
    None => println!("We got nothing"),
}
```

以上代码输出如下：

```text
We go something: a String
We got nothing
```

### 取出 ``Option`` 中的值

通常，你会看到 unwrap 被使用。一开始看起来有点神秘，在IDE中浏览它可以提供一些线索：

![unwrap.png](https://github.com/rustt-org/rustt-assets/blob/main/20220326-rust-``Option``-and-``Result``/ide_unwrap.png?raw=true)

如果您使用的是 VScode ，那么同时按下 Ctrl + 鼠标左键可以找到源代码。在这种情况下，它将我们带到 ``Option``.rs 文件中 unwrap 定义的位置：

```rust
pub const fn unwrap(self) -> T {
    match self {
        Some(val) => val,
        None => panic!("called ```Option``::unwrap()` on a `None` value"),
    }
}
```

在 ``Option.rs`` 中，我们可以看到 unwrap 被定义在 impl \<T> ``Option<T>`` 代码块中。 当我们调用某个值时，它会尝试 unwrap 隐藏在 Some 变量中的值。它与 self 匹配，如果存在 Some 变量，val 将被 unwrap 并返回。如果存在 None 变量，则调用 panic 宏：

```rust
let something: Option<&str> = Some("Something");
let unwrapped = something.unwrap(); 
println!("{:?}\n{:?}", something, unwrapped);
let nothing: Option<&str> = None;
nothing.unwrap();
```

以上代码输出如下：

```text
Some("Something")
"Something"
thread 'main' panicked at 'called ```Option``::unwrap()` on a `None` value', src\main.rs:86:17
```

在一个 ``Option`` 上调用 unwrap 方法既快捷又简单，但让程序陷入恐慌和崩溃并不是一种非常优雅或安全的方法。

### ``Option`` 实例

让我们来看一些可以使用 ``Option`` 的示例。

#### 给一个函数传递一个可选的数值

```rust
fn might_print(Option: Option<&str>) {
    match Option {
        Some(text) => println!("The argument contains the following value: '{}'", text),
        None => println!("The argument contains None."),
    }
}
let something: Option<&str> = Some("some str");
let nothing: Option<&str> = None;
might_print(something);
might_print(nothing);
```

输出如下：

```text
The argument contains the following value: 'some str'
The argument contains None.
```

#### 定义一个函数返回一个 ``Option`` 数值

```rust
// Returns the text if it contains target character, None otherwise:
fn contains_char(text: &str, target_c: char) -> Option<&str> {
    if text.chars().any(|ch| ch == target_c) {
        return Some(text);
    } else {
        return None;
    }
}
let a = contains_char("Rust in action", 'a');
let q = contains_char("Rust in action", 'q');
println!("{:?}", a);
println!("{:?}", q);
```

我们可以安全地将此函数的返回分配给一个变量，然后使用 match 确定如何处理 None 返回。前面的代码打印以下内容：

```text
Some("Rust in action")
None
```

让我们研究三种不同的方法来处理 ``Option`` 的返回。

第一个，是最不安全的，简单地调用 unwrap 方法：

```rust
let a = contains_char("Rust in action", 'a');
let a_unwrapped = a.unwrap();   
println!("{:?}", a_unwrapped);
```

第二个，更安全的选择是使用 match 语句：

```rust
let a = contains_char("Rust in action", 'a');
match a {
    Some(a) => println!("contains_char returned something: {:?}!", a),
    None => println!("contains_char did not return something, so branch off here"),
}
```

第三个， ``Option`` 是在变量中捕获函数的返回，并在以下情况下使用：

```rust
let a = contains_char("Rust in action", 'a');
if let Some(a) = contains_char("Rust in action", 'a') {
    println!("contains_char returned something: {:?}!", a);
} else {
    println!("contains_char did not return something, so branch off here")
}
```

#### struct 中的 ``Option`` 数值

我们也可以在结构中使用 ``Option`` 。表示字段可能值有或可能没有任何值，一般会很有用：

```rust
#[derive(Debug)]
struct Person {
    name: String,
    age: Option<i32>,
}

let marie = Person {
    name: String::from("Marie"),
    age: Some(2),
};

let jan = Person {
    name: String::from("Jan"),
    age: None,
};
println!("{:?}\n{:?}", marie, jan);
```

以上代码输出如下：

```text
Person { name: "Marie", age: Some(2) }
Person { name: "Jan", age: None }
```

#### 真实世界的实例

在 Rust 中使用 ``Option`` 的一个例子是数组的 pop 方法。此方法返回一个 ``Option<T>`` 。pop 方法返回最后一个元素，有时候元素可能是空的。在这种情况下，它应该返回 None 值。另一个问题是，数组里的元素可以包含任何类型。在这种情况下，它很方便返回 Some(T) 。因此，pop() 返回 ``Option<T>`` 。

从 Rust 1.53 获得数组的 pop 方法：

```rust
impl<T, A: Allocator> Vec<T, A> {
    // .. lots of other code
    pub fn pop(&mut self) -> Option<T> {
        if self.len == 0 {
            None
        } else {
            unsafe {
                self.len -= 1;
                Some(ptr::read(self.as_ptr().add(self.len())))
            }
        }
    }
    // lots of other code
}   
``` 

一个简单的例子，我们不断输出数组里的元素，尽管已经超出其包含的数据项：

```rust
let mut vec = vec![0, 1];
let a = vec.pop();
let b = vec.pop();
let c = vec.pop();
println!("{:?}\n{:?}\n{:?}\n", a, b, c);
```

以上代码输出如下：

```text
Some(1)
Some(0)
None
```

## ``Result``

Rust 另外的一个重要的结构就是 ``Result`` 枚举。和 ``Option`` 一样，``Result`` 也是一个枚举。在 ``Result.rs`` 源码里可以找到 ``Result`` 的定义：

```rust
pub enum Result<T, E> {
    /// Contains the success value
    Ok(T),
    /// Contains the error value    
    Err(E),
}
```
``Result`` 枚举是包含2个类型的泛型，分别是 T 和 E 。T 用于 Ok 变量，表示一个成功的结果。E 用于 Err 变量，表示一个错误的数值。由于 E 是泛型，所以可以使人们可以传递不同的错误。如果 ``Result`` 不是 E 上的泛型，那么只会有1种类型的错误。就会与 ``Option`` 中有1种类型的 None 相同。在报告中使用错误值时，就显得不太灵活。

如前所述，Prelude 将 ``Result`` 枚举以及 Ok 和 Err 变量纳入 prelude 的范围，如下所示：

```
std::``Result``::``Result``::{self, Ok, Err}
```

这意味着在我们的代码里面，我们可以在任何位置，直接访问 ``Result`` ，Ok 和 Err 。

### ``Result`` 上的匹配

让我们开始创建一个可以返回 ``Result`` 的函数例子。在这个函数例子里，我们检查一个字符串是否包括一个最小数量的字符。函数如下：

```rust
fn check_length(s: &str, min: usize) -> Result<&str, String> {
    if s.chars().count() >= min {
        return Ok(s)
    } else {
        return Err(format!("'{}' is not long enough!", s))
    }
}
```

这不是一个非常有用的函数，但足够简单的展示返回一个 ``Result`` 。函数带有两个参数，一个是字符串字面量参数，一个是需要检查包含的字符数量参数。如果字符数量等于或者大于 min , 字符串被返回。这个返回值标记成了 ``Result`` 枚举。我们指定函数返回时 ``Result`` 将包含的类型。如果字符串足够长，我们将返回字符串文本。如果出现错误，我们将返回一条字符串消息。这例子很好解释了 ``Result`` <&str，String>。

if s.chars().count() >= min 语句为我们进行检查。如果计算结果为true，它将返回 ``Result`` 枚举的 Ok 变量中包装的字符串。我们之所以可以简单地写 Ok(s)，是因为组成 ``Result`` 的变量也被纳入了范围。我们可以看到 else 语句将返回一个 Err 变量。在本例中，它是一个包含错误消息的字符串。

让我们运行函数并且使用 dbg！输出 ``Result`` ：

```rust
let a = check_length("some str", 5);
let b = check_length("another str", 300);
dbg!(a); // Ok("some str",)
dbg!(b); // Err("'another str' is not long enough!",)
``` 

我们可以使用 match 表达式处理我们函数的返回值 ``Result`` ：

```rust
let func_return = check_length("some string literal", 100);
let a_str = match func_return {
    Ok(a_str) => a_str,
    Err(error) => panic!("Problem running 'check_length':\n {:?}", error),
};
// thread 'main' panicked at 'Problem running 'check_length':
// "'some string literal' is not long enough!"'
```
### 取出 ``Result`` 的值

除了使用匹配表达式，还有一个你经常会遇到的快捷方式。此快捷方式是为 ``Result`` 枚举定义的 unwrap 方法。该方法定义如下：

```rust
impl<T, E: fmt::Debug> Result<T, E> {
    ...
    pub fn unwrap(self) -> T {
        match self {
            Ok(t) => t,
            Err(e) => unwrap_failed("called ```Result``::unwrap()` on an `Err` value", &e),
        }
    }
    ...
}
```

调用 unwrap 返回一个包含 Ok 的值，如果没有 Ok 值，unwrap 将会触发恐慌。在下面的例子中，from_str 方法返回 Ok 值：

```rust
use serde_json::json;
let json_string = r#"
{
    "key": "value",
    "another key": "another value",
    "key to a list" : [1 ,2]
}"#;
let json_serialized: serde_json::Value = serde_json::from_str(&json_string).unwrap();
println!("{:?}", &json_serialized);
// Object({"another key": String("another value"), "key": String("value"), "key to a list": Array([Number(1), Number(2)])})
```
我们可以看到 json_serialized 包含的数值包裹在 Ok 变量里面了。 

以下演示了当我们在一个函数中调用 unwrap 时，如果不返回 Ok 变量会发生什么。这里，我们调用 serde_json::from_str 用于无效的 JSON ：

```rust
use serde_json::json;
let invalid_json = r#"
{
    "key": "v
}"#;

let json_serialized: serde_json::Value = serde_json::from_str(&invalid_json).unwrap();
/*
thread 'main' panicked at 'called ```Result``::unwrap()` on an `Err` value: Error("control character (\\u0000-\\u001F) found while parsing a string", line: 4, column: 0)',
*/
```

这导致了一个恐慌并且程序终止了。除了 unwrap ，我们还可以选择使用 expect 。

```rust
use serde_json::json;

let invalid_json = r#"
{
    "key": "v
}"#;
let b: serde_json::Value =
    serde_json::from_str(&invalid_json).expect("unable to deserialize JSON");
```

这次，当我们执行代码时，我们也可以看到我们增加的消息：

```text
thread 'main' panicked at 'unable to deserialize JSON: Error("control character (\\u0000-\\u001F) found while parsing a string", line: 4, column: 0)'
```

因为 unwrap 和 expect 会导致恐慌，所以程序结束了。大多数情况下，你会在示例部分看到 unwrap 的使用，其中的重点是示例，缺乏上下文确实会妨碍对特定场景进行正确的错误处理。示例部分、代码注释和文档示例是你经常遇到的 unwrap 部分。例如，请参见将 serde 序列化字段作为 camelCase 的示例：

```rust
use serde::Serialize;

#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
struct Person {
    first_name: String,
    last_name: String,
}

fn main() {
    let person = Person {
        first_name: "Graydon".to_string(),
        last_name: "Hoare".to_string(),
    };

    let json = serde_json::to_string_pretty(&person).unwrap();  // <- unwrap

    // Prints:
    //
    //    {
    //      "firstName": "Graydon",
    //      "lastName": "Hoare"
    //    }
    println!("{}", json);
}
```

例子来自于 serde [文档](https://serde.rs/attr-rename.html) 。

### 使用 ? 并且处理不同的错误

不同的项目通常会定义自己的错误。在 repo 中搜索诸如 pub struct Error 或 pub enum Error 之类的内容，有时可能会发现为项目定义的错误。但问题是，不同的 create 和项目可能会返回自己的错误类型。如果有一个函数使用来自各种项目中的方法，并且想要传播错误，那么事情可能会变得有点棘手。有几种方法可以解决这个问题。让我们来看一个例子，我们通过 Box 错误来处理这个问题。

在下一个示例中，我们定义了一个函数，该函数将目标文件的全部内容读入到一个字符串中，然后将其序列化为 JSON ，同时将其映射到结构体。函数返回一个 ``Result`` 。Ok 变量是Person结构体，将传播的错误可能是来自 serde 或 std::fs 的错误。为了能够从这两个包返回错误，我们返回``Result``<Person，Box<dyn Error>>。 Person 是 ``Result`` 的 Ok 变体。Err 变量定义为 Box<dyn Error> ，表示任何类型的错误 。

关于下面的例子，另一件值得一提的事情是 ？。我们将使用 fs::read_to_string 将文件作为字符串读取，并使用 serde_json::from_str(&text) 将文本序列化为结构。为了避免为这些方法返回的结果编写匹配表达式，我们将 ？放在调用这些方法的背后。如果前面的 ``Result`` 包含 Ok ，则此语法糖将执行 unwrap 。如果前面的结果包含 Err 变量，它将确保返回此错误，就像使用 return 关键字传播错误一样。当错误被返回时，我们的 Box 将捕获它们。

实例代码：

```rust
use serde::{Deserialize, Serialize};
use std::error::Error;
use std::fs;

// Debug allows us to print the struct.
// Deserialize and Serialize adds decoder and encoder capabilities to the struct.
#[derive(Debug, Deserialize, Serialize)]
struct Person {
    name: String,
    age: usize,
}

fn file_to_json(s: &str) -> Result<Person, Box<dyn Error>> {
    let text = fs::read_to_string(s)?;
    let marie: Person = serde_json::from_str(&text)?;
    Ok(marie)
}

let x = file_to_json("json.txt");
let y = file_to_json("invalid_json.txt");
let z = file_to_json("non_existing_file.txt");

dbg!(x);
dbg!(y);
dbg!(z);
```

首次我们调用函数，运行成功并且 dbg!(x) 输出如下内容：

```text
[src\main.rs:20] x = Ok(
    Person {
        name: "Marie",
        age: 2,
    },
)
```

第二次调用遇到错误。文件包括如下内容：

```text
{
	"name": "Marie",
	"a
}
```

这个文件可以打开并且读入数据，但是 Serde 不能解析成 JSON 格式。这个函数调用输出如下内容：

```text
[src\main.rs:21] y = Err(
    Error("control character (\\u0000-\\u001F) found while parsing a string", line: 3, column: 4),     
)
```

我们可以看到 Serde 的错误可以正确的传播

最后一个函数调用试图打开一个并不存在的文件：

```text
[src\main.rs:22] z = Err(
    Os {
        code: 2,
        kind: NotFound,
        message: "The system cannot find the file specified.",
    },
)
```

那个来自 std::fs 的错误可以被正确的传播。

### 使用其他的 crates：anyhow

有许多的 crates 可以帮助我们处理错误，一些可以帮助我们管理样板代码，另一些则添加了额外报告等功能。一个易于使用的 crates 的代表就是 anyhow ：

![anyhow.png](https://github.com/rustt-org/rustt-assets/blob/main/20220326-rust-``Option``-and-``Result``/anyhow.png?raw=true)

这个 crate 将帮助我们实现一个简化的 ``Result`` 类型，同时通过增加 context 我们可以很轻松的标记我们的错误。下面的代码片段演示了三个 anyhow 的基础能力：

```rust
use anyhow::{anyhow, Context, ``Result``};
use serde::{Deserialize, Serialize};
use std::fs;

#[derive(Debug, Deserialize, Serialize)]
struct Secrets {
    username: String,
    password: String,
}

fn get_secrets(s: &str) -> Result<Secrets> {
    let text = fs::read_to_string(s).context("Secrets file is missing.")?;
    let secrets: Secrets =
        serde_json::from_str(&text).context("Problem serializing secrets.")?;
    if secrets.password.chars().count() < 2 {
        return Err(anyhow!("Password in secrets file is too short"));
    }
    Ok(secrets)
}
```

在以上的例子中，需要在 Cargo.toml 增加 anyhow = "1.0.43" 。在头部，anyhow ，Context 和 ``Result`` 被引入。让我们一个接一个的讨论：

#### anyhow::``Result``
这是一个处理错误的一个更方便的类型。你可以在 main() 中使用它。在 get_secrets 函数中我们可以看到 ``Result`` 在使用，就是这个被实现的 enum 使事情变得简单。这些 <ruby>特征<rt>trait</rt></ruby> 中的一个称作 Context ，我们将在下面进行讨论。

如果我们运行 get_secrets  并且一切正常，我们获得如下返回：

```rust
let a = get_secrets("secrets.json");
dbg!(a);
```

以上代码输出如下：

```
/*
[src\main.rs:] a = Ok(
    Secrets {
        username: "username",
        password: "password",
    },
)
*/
```

我们获得了一个正常的 Ok 数值。

#### anyhow::Context

正如错误是可以被传播的，Context <ruby>特征<rt>trait</rt></ruby> 允许你将原始错误包装起来，并且包含了一个更多语境意识的信息。以前，我们会这样打开一个文件：

```rust
let text = fs::read_to_string(s)?;
```

这将打开文件并且 unwrap Ok 变量。或者, 如果read_to_string 返回一个 Err，那么 ? 会传播这个错误。如今，我们可以象如下这样： 

```rust
let text = fs::read_to_string(s).context("Secrets file is missing.")?;
```

我们对于 serde_json::from_str 方法做同样类似的事情，我们可以使用以下代码出发2个错误：

```rust
let b = get_secrets("secrets.jsonnn");
dbg!(b);

let c = get_secrets("invalid_json.txt");
dbg!(c);
```
在第一个用例，由于文件并不存在将会引起一个错误。在第二个用例，Serde 由于无法解析 JSON 将给我们一个错误。上面的函数输出如下内容：

```text
[src\main.rs:358] b = Err(
    Error {
        context: "Secrets file is missing.",
        source: Os {
            code: 2,
            kind: NotFound,
            message: "The system cannot find the file specified.",
        },
    },
)
[src\main.rs:361] c = Err(
    Error {
        context: "Problem serializing secrets.",
        source: Error("control character (\\u0000-\\u001F) found while parsing a string", line: 3, column: 4),
    },
)
```

#### anyhow::anyhow

这是一个宏，你可以使用它来返回一个 anyhow::Error。下面加载一些 JSON ，其中包含太短的密码（我知道这是一个愚蠢的例子）：

```rust
let d = get_secrets("wrong_secrets.json");
dbg!(d);
```

结果应该如下：

```
[src\main.rs:364] d = Err(
    "Password in secrets file is too short",
)
```

Jane Lusby 复制了 anyhow crate。她创建了[eyre](https://crates.io/crates/eyre/0.3.7)，这也是值得一看的。特别是结合优秀的[Rustconf 2020](https://2020.rustconf.com/talks)演讲，所谓的[Error handling Isn’t All About Errors](https://www.youtube.com/watch?v=rAF8mLI0naQ)。

## 收尾

了解 ``Option`` 以及 ``Result`` 如何用于 Rust 非常重要。上面对 Rust 中的 ``Option`` ，``Result`` 和错误处理的解释，是我如何了解它们的书面描述。我希望这篇文章对其他人有益。

这篇文章的源码可以在(这里)找到[https://github.com/saidvandeklundert/LearningRust/blob/master/blog/``Option``-and-``Result``/src/main.rs]。 除此之外，这里还有一些额外的链接值得查看，以更好地理解 Rust 中的 ``Option``，``Result`` 以及错误处理。

* The Rust Programming Language [Chapter 6](https://doc.rust-lang.org/book/ch06-00-enums.html) and [Chapter 9](https://doc.rust-lang.org/book/ch09-00-error-handling.html)
* From [Rustconf 2020](https://2020.rustconf.com/talks) talks, the [Error handling Isn’t All About Errors](https://www.youtube.com/watch?v=rAF8mLI0naQ) talk by by Jane Lusby 
* Error handling and dealing with multiple error types in [Rust by example](https://doc.rust-lang.org/rust-by-example/error/multiple_error_types.html)
* [Next level thoughts and ideas on errors](https://www.lpalmieri.com/posts/error-handling-rust/)
* [Wrapping errors](https://doc.rust-lang.org/rust-by-example/error/multiple_error_types/wrap_error.html)

---

> 原文链接: [http://saidvandeklundert.net/learn/2021-11-18-calling-rust-from-python-using-pyo3/](http://saidvandeklundert.net/learn/2021-11-18-calling-rust-from-python-using-pyo3/)
> 
> 翻译：[minikiller](https://github.com/minikiller)
> 选题：[minikiller](https://github.com/minikiller)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出