 # 使用pyo3，从python调用rust代码

 在 Rust 中，完全理解枚举 Option 和枚举 Result，对于理解可选数值和错误处理非常重要。在这篇文章中，我们将深入学习这两方面的内容。

## 概述

想理解  <font color="green">Option</font> 和 Result，理解如下内容是很重要的：

* Rust 中的 enum
* 匹配 enum 变量
* Rust 中的 prelude

### Rust 中的 enum

有很好的原因去使用 enum ，Among others, they are good for safe input handling and adding context to types by giving a collection of variants a name. In Rust, the Option as well as the Result are enumerations, also referred to as enums. Rust 中的 enum 是非常灵活的，它可以包括很多数据类型，例如 tuple ，struct 等等。另外，你还可以在 enums 上实现方法。

Option 和 Result 理解起来非常简单。我们首先来看一个 enum 的例子：

```rust
enum Example {
    This,
    That,
}

let this = Example::This;
let that = Example::That;
```

在上面，我们定义了一个 enum 命名为 <font color="green">Example</font>，这个 enum 包括两个变量 <font color="green">This</font> 和 <font color="green">That</font>。然后，我们创建了2个 enum 的实例变量  <font color="green">This</font> 和 <font color="green">That</font>。他们使用其自己的字段被创建。重要的是一个 enum 实例经常使用多个字段中的一个。当你使用 struct 定义字段的时候，你可以使用其定义的全部可能的字段。一个 enum 是不一样的，因为你只可以使用其中的一个变量字段。

### 显示 enum 变量

默认的，enum 变量是不能打印到屏幕的。 在我们定义的 enum 上，通过使用 <font color="green">strum_macros</font> 可以很容易的派生 ‘Display’， 在 enum 上我们使用 <font color="green">#[derive(Display)]</font> ：

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

使用 <font color="green">match</font> 关键字，我们可以在 enum 上实现模式匹配。下面函数使用Example 枚举做为参数：

```rust
fn matcher(x: Example) {
    match x {
        Example::This => println!("We got This."),
        Example::That => println!("We got That."),
    }
}
```

我们可以给这个 matcher 函数传递一个Example的数值，函数内部的 <font color="green">match</font> 将决定什么内容会被打印到屏幕：

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

The Rust prelude, described here, is something that is a part of every program. It is a list of things that are automatically imported into every Rust program. Most of the items in the prelude are traits that are used very often. But in addition to this, we also find the following 2 items:

* std::option::Option::{self, Some, None}
* std::result::Result::{self, Ok, Err}

The first one is the <font color="green">Option</font> enum, described as ‘a type which expresses the presence or absence of a value’. The later is the <font color="green">Result</font> enum, described as ‘a type for functions that may succeed or fail.’

Because these types are so commonly used, their variants are also exported. Let’s go over both types in more detail.

## Option
Brought into scope by the prelude, the [Option](https://doc.rust-lang.org/std/option/index.html) is available to us without having to lift a finger. option 定义如下：

```rust
pub enum Option<T> {
    None,
    Some(T),
}
```

The above tells us <font color="green">Option<T></font> is an enum with 2 variants: <font color="green">None</font> and <font color="green">Some(T)</font>. In terms of how it is used, the <font color="green">None</font> can be thought of as ‘nothing’ and the <font color="green">Some(T)</font> can be thought of as ‘something’. A key thing that is not immediately obvious to those starting out with Rust is the <font color="green"><T></font>-thing. The <font color="green"><T></font> tells us the Option Enum is a <font color="green">generic</font> Enum.

### Option 是 type T 泛型

The enum is ‘generic over type T’. The ‘T’ could have been any letter, just that ‘T’ is used as a convention where 1 generic is involved.

So what does it mean when we say ‘the enum is generic over type T’? It means that we can use it for any type. As soon as we start working with the Enum, we can (and must actually) replace ‘T’ with a concrete type. This can be any type, as illustrated by the following:

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

The code shows us the enum can be generic over standard as well as over custom types. Additionally, when we define an enum as being of type x, it can still contain the variant ‘None’. So the Option is a way of saying the following:

```text
This can be of a type T value, which can be anything really, or it can be nothing. 
```

### Option 上的匹配
Since Rust does not use exceptions or null values, you will see the Option (and as we will learn later on the Result) used all over the place.

Since the Option is an enum, we can use pattern matching to handle each variant in it’s own way:

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

### Unwrapping the Option
Oftentimes, you’ll see unwrap being used. This looked a bit mysterious at first. Hoovering over it in the IDE offers some clues:

{[ID unwrap]}

In case you are using VScode, something that is nice to know is that simultaneously pressing Ctrl + left mouse button can take you to the source code. In this case, it takes us to the place in option.rs where unwrap is defined:

```rust
pub const fn unwrap(self) -> T {
    match self {
        Some(val) => val,
        None => panic!("called `Option::unwrap()` on a `None` value"),
    }
}
```

In option.rs, we can see unwrap is defined in the impl<T> Option<T> block. When we call it on a value, it will try to ‘unwrap’ the value that is tucked into the Some variant. It matches on ‘self’ and if the Some variant is present, ‘val’ is ‘unwrapped’ and returned. If the ‘None’ variant is present, the panic macro is called:

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
thread 'main' panicked at 'called `Option::unwrap()` on a `None` value', src\main.rs:86:17
```

Calling unwrap on an Option is quick and easy but letting your program panic and crash is not a very elegant or safe approach.

### Option 实例
Let’s look at some examples where you could use an Option.

#### 给一个函数传递一个可选的数值
```rust
fn might_print(option: Option<&str>) {
    match option {
        Some(text) => println!("The argument contains the following value: '{}'", text),
        None => println!("The argument contains None."),
    }
}
let something: Option<&str> = Some("some str");
let nothing: Option<&str> = None;
might_print(something);
might_print(nothing);
```

输入如下：

```text
The argument contains the following value: 'some str'
The argument contains None.
```

#### 定义一个函数返回一个 Option 数值

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
We can safely assign the return of this function to a variable and, later on, use match to determine how to handle a ‘None’ return. The previous code prints the following:
```text
Some("Rust in action")
None
```
Let’s examine three different ways to work with the Optional return.

The first one, which is the least safe, would be simply calling unwrap:
```rust
let a = contains_char("Rust in action", 'a');
let a_unwrapped = a.unwrap();   
println!("{:?}", a_unwrapped);
```
The second, safer option, is to use a match statement:
```rust
let a = contains_char("Rust in action", 'a');
match a {
    Some(a) => println!("contains_char returned something: {:?}!", a),
    None => println!("contains_char did not return something, so branch off here"),
}
```
The third option is to capture the return of the function in a variable and use if let:
```rust
let a = contains_char("Rust in action", 'a');
if let Some(a) = contains_char("Rust in action", 'a') {
    println!("contains_char returned something: {:?}!", a);
} else {
    println!("contains_char did not return something, so branch off here")
}
```

#### struct 中的 Option 数值
We can also use the Option inside a struct. This might be useful in case a field may or may not have any value:

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
An example where the Option is used inside Rust is the pop method for vectors. This method returns an Option<T>. The pop-method returns the last element. But it can be that a vector is empty. In that case, it should return None. An additional problem is that a vector can contain any type. In that case, it is convenient for it to return Some(T). So for that reason, pop() returns Option<T>.

The pop method for the vec from Rust 1.53:
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
A trivial example where we output the result of popping a vector beyond the point where it is still containing items:

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

## Result

Another important construct in Rust is the Result enum. Same as with the Option, the Result is an enum. The definition of the Result can be found in result.rs:

```rust
pub enum Result<T, E> {
    /// Contains the success value
    Ok(T),
    /// Contains the error value    
    Err(E),
}
```

The Result enum is generic over 2 types, given the name T and E. The T is used for the OK variant, which is used to express a successful result. The E is used for the Err variant, used to express an error value. The fact that Result is generic over E makes it possible to communicate different errors. If Result would not have been generic over E, there would just be 1 type of error. Same as there is 1 type of ‘None’ in Option. This would not leave a lot of room when using the error value in our flow control or reporting.

As indicated before, the Prelude brings the Result enum as well as the Ok and Err variants into scope in the Prelude like so:
```
std::result::Result::{self, Ok, Err}
```
这意味着在我们的代码里面，我们可以在任何位置，直接访问 Result ，Ok 和 Err 。

### Result 上的匹配
Let’s start off creating an example function that returns a Result. In the example function, we check whether a string contains a minimum number of characters. The function is the following:

```rust
fn check_length(s: &str, min: usize) -> Result<&str, String> {
    if s.chars().count() >= min {
        return Ok(s)
    } else {
        return Err(format!("'{}' is not long enough!", s))
    }
}
```

It is not a very useful function, but simple enough to illustrate returning a Result. The function takes in a string literal and checks the number of characters it contains. If the number of characters is equal to, or more then ‘min’, the string is returned. If this is not the case, an error is returned. The return is annotated with the Result enum. We specify the types that the Result will contain when the function returns. If the string is long enough, we return a string literal. If there is an error, we will return a message that is a String. This explains the Result<&str, String>.

The if s.chars().count() >= min does the check for us. In case it evaluates to true, it will return the string wrapped in the Ok variant of the Result enum. The reason we can simply write Ok(s) is because the variants that make up Result are brought into scope as well. We can see that the else statement will return an Err variant. In this case, it is a String that contains a message.
让我们运行函数并且使用 dbg！ 输出 Result ：

```rust
let a = check_length("some str", 5);
let b = check_length("another str", 300);
dbg!(a); // Ok("some str",)
dbg!(b); // Err("'another str' is not long enough!",)
``` 

我们可以使用 match 表达式处理我们函数的返回值 Result ：

```rust
let func_return = check_length("some string literal", 100);
let a_str = match func_return {
    Ok(a_str) => a_str,
    Err(error) => panic!("Problem running 'check_length':\n {:?}", error),
};
// thread 'main' panicked at 'Problem running 'check_length':
// "'some string literal' is not long enough!"'
```
### Unwrapping the Result
Instead of using a match expression, there is also a shortcut that you’ll come across very often. This shortcut is the unwrap method that is defined for the Result enum. The method is defined as follows:
```rust
impl<T, E: fmt::Debug> Result<T, E> {
    ...
    pub fn unwrap(self) -> T {
        match self {
            Ok(t) => t,
            Err(e) => unwrap_failed("called `Result::unwrap()` on an `Err` value", &e),
        }
    }
    ...
}
```

Calling unwrap returned the contained ‘Ok’ value. If there is no ‘Ok’ value, unwrap will panic. In the following example, the from_str method returns an ‘Ok’ value:

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
We can see that ‘json_serialized’ contains the value that was wrapped in the ‘Ok’ variant.

The following demonstrates what happened when we call unwrap on a function that does not return an ‘Ok’ variant. Here, we call ‘serde_json::from_str’ on invalide JSON:

```rust
use serde_json::json;
let invalid_json = r#"
{
    "key": "v
}"#;

let json_serialized: serde_json::Value = serde_json::from_str(&invalid_json).unwrap();
/*
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Error("control character (\\u0000-\\u001F) found while parsing a string", line: 4, column: 0)',
*/
```

There is a panic and the program comes to a halt. Instead of unwrap, we could also choose to use expect.

```rust
use serde_json::json;

let invalid_json = r#"
{
    "key": "v
}"#;
let b: serde_json::Value =
    serde_json::from_str(&invalid_json).expect("unable to deserialize JSON");
```

This time, when we run the code, we can also see the message that we added to it:

```text
thread 'main' panicked at 'unable to deserialize JSON: Error("control character (\\u0000-\\u001F) found while parsing a string", line: 4, column: 0)'
```

Since unwrap and expect result in a panic, it ends the program, period. Most oftentimes, you’ll see unwrap being used in the example section, where the focus is on the example and lack of context really prohibits proper error handling for a specific scenario. The example section, code comments and documentation examples is where you will most oftentimes encounter unwrap. See for instance the example to have serde serialize fields as camelCase:

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
Example taken from the serde documentation, right here.

### 使用 ? 并且处理不同的错误
Different projects oftentimes define their own errors. Searching a repo for something like pub struct Error or pub enum Error can sometimes reveal the errors defined for a project. But the thing is, different crates and projects might return their own error type. If you have a function that uses methods from a variety of projects, and you want to propagate that error, things can get a bit trickier. There are several ways to deal with this. Let’s look at an example where we deal with this by ‘Boxing’ the error.

In the next example, we define a function that reads the entire contents of target file into a string and then serializes it into JSON, while mapping it to a struct. The function returns a Result. The Ok variant is the ‘Person’ struct and the Error that will be propagated can be an error coming serde or std::fs. To be able to return errors from both these packages, we return Result<Person, Box<dyn Error>>. The ‘Person’ is the Ok variant of the Result. The Err variant is defined as Box<dyn Error>, which represents ‘any type of error’.

Another thing worth mentioning about the following example is the use of ?. We will use fs::read_to_string(s) to read a file as a string and we will use erde_json::from_str(&text) to serialize the text to a struct. In order to avoid having to write match arms for the Results returned by those methods, we place the ? behind the call to those methods. This syntactic sugar will perform an unwrap in case the preceding Result contains an Ok. If the preceding Result contains an Err variant, it ensures that this Err is returned just as if the return keyword would have been used to propagate the error. And when the error is returned, our ‘Box’ will catch them.

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

The first time we call the function, it succeeds and dbg!(x); outputs the following:
```text
[src\main.rs:20] x = Ok(
    Person {
        name: "Marie",
        age: 2,
    },
)
```
The second calls encounters an error. The file contains the following:
```text
{
	"name": "Marie",
	"a
}
```
This file can be opened and read to a String, but Serde cannot parse it as JSON. This function call outputs the following:
```text
[src\main.rs:21] y = Err(
    Error("control character (\\u0000-\\u001F) found while parsing a string", line: 3, column: 4),     
)
```
We can see the Serde error was properly propagated.

The last function call tried to open a file that does not exist:
```text
[src\main.rs:22] z = Err(
    Os {
        code: 2,
        kind: NotFound,
        message: "The system cannot find the file specified.",
    },
)
```
That error, coming from std::fs, was also properly propagated.

### 使用其他的 crates：anyhow

有许多的 crates 可以帮助我们处理错误，一些可以帮助我们管理样板代码，另一些则添加了额外报告等功能。一个易于使用的 crates 的代表就是 anyhow ：

![anyhow.png](https://github.com/rustt-org/rustt-assets/blob/main/20220326-rust-option-and-result/anyhow.png?raw=true)

这个 crate 将帮助我们实现一个简化的 Result 类型，同时通过增加 context 我们可以很轻松的标记我们的错误。下面的代码片段演示了三个 anyhow 的基础能力：

```rust
use anyhow::{anyhow, Context, Result};
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

在以上的例子中，需要在 Cargo.toml 增加 anyhow = "1.0.43" 。在头部，anyhow ，Context 和 Result 被引入。让我们一个接一个的讨论：

#### anyhow::Result
这是一个处理错误的一个更方便的类型。你可以在 main() 中使用它。在 get_secrets 函数中我们可以看到 Result 在使用，就是这个被实现的 enum 使事情变得简单。这些 trait 中的一个称作 Context ，我们将在下面进行讨论。

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

正如错误是可以被传播的，Context trait 允许你将原始错误包装起来，并且包含了一个更多语境意识的信息。以前，我们会这样打开一个文件：

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

了解 Option 以及 Result 如何用于 Rust 非常重要。上面对 Rust 中的 Option ，Result 和错误处理的解释，是我如何了解它们的书面描述。我希望这篇文章对其他人有益。

这篇文章的源码可以在(这里)找到[https://github.com/saidvandeklundert/LearningRust/blob/master/blog/option-and-result/src/main.rs]。 除此之外，这里还有一些额外的链接值得查看，以更好地理解 Rust 中的 Option，Result 以及错误处理。

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