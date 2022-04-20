# 使用 pyo3 从 Python 调用 Rust -> 使用 Rust 加速你的 Python

通过使用[PyO3](https://github.com/PyO3/pyo3)，从 Python 调用 Rust 代码是很容易的事情。 依赖 `PyO3` 和 `maturin` 的组合你可以写一个 Rust 的类库，`maturin` 是 `PyO3` 生态系统的一个支持工具，它可以编译 Rust 类库并使其直接作为 Python 的模块被安装。除此之外，`PyO3` 可以在 Python 和 Rust 之间转换类型，还可以通过一组宏轻松地将 Rust 函数导出到 Python 。

在这篇博文中，我将对 `PyO3` 做一个简短的介绍。之后，我将讨论几个用 Rust 编写并从 Python 调用的示例函数。这些例子包括：

* 计算 Python 和 Rust 中的n阶斐波那契数列
* 让 Python 在 Rust 函数中使用多种类型 ？？？
* 在 Python 代码里使用 Rust 的结构体
* Python 发送 `JSON` 给 Rust 并且作为结构体序列化这个 `JSON`
* 允许 Rust 从 Python 运行时中使用 `logger`
* 在 Rust 里产生一个 `Error` 并且在 Python 里作为异常被捕获

## `PyO3` 介绍 

`PyO3` 为希望将 Rust 和 Python 代码粘合在一起的人提供了一些人体工程学。它可以帮助您从 Rust 调用 Python 代码，以及从 Python 调用 Rust 代码。因为我一直在使用它来从 Python 调用 Rust 代码，所以这是我在这里要写的唯一一件事。

那么，`PyO3` 给了你什么？

首先，`maturin` 该工具将为你编译 Rust 代码，并将编译后的代码作为 Python 模块安装到你的虚拟环境中。之后，你可以在 Python 代码中导入该模块并使用它。在 `pip install maturin` 之后，只需运行1个命令 `maturin develop` 即可使用 Python 中的 Rust 代码。

除了 `maturin` 之外，就是 PyO3 本身了。 PyO3 提供了 Rust 绑定到 Python 的解析器。 这使你根本不需要为 Python 和 Rust 的交互而烦恼。例如，您不必担心如何将 Python 字符串转换为 C 语言中的某些内容，然后再转换为 Rust 中的其他内容。对于整数、浮点数、列表以及字典等等也是同样的。 为了方便起见，PyO3 附带了很多宏，可以避免编写太多样板代码。要向 Python 公开 Rust 函数，可以使用宏对其进行注释。在这之后，PyO3 将负责剩下的部分。如果要导出结构或方法，同样适用。

除了 `maturin`，当然还有`PyO3`本身。`PyO3` 为 Python 解释器提供了 Rust 绑定。这使得你不必为 Python 和 Rust 之间的交互操心太多。例如，你不必担心如何将 Python 字符串转换为 C 语言中的某些内容，然后再转换为 Rust 中的其他内容。整数、浮点数、列表、字典等也是如此。为了方便起见，`PyO3` 附带了很多宏，可以避免编写太多样板代码。要向 Python 公开 Rust 函数，可以使用宏对其进行注释。在这之后，`PyO3` 将负责剩下的部分。如果要导出结构或方法，也同样适用。

## 从 Python 调用一个 Rust 函数

在第一个示例中，我们将从 Python 调用一个 Rust 乘法函数。通常，我们可以这样写：

```rust
fn multiply(a: isize, b: isize) -> isize {
    a * b
}
```

在不添加太多内容的情况下，我们可以从 Python 调用这个函数。首先，我们需要：

* 引入 `pyo3` `prelude`
* 使用 `#[pyfunction]` 标记函数，将其转换为 `PyCFunction`
* 把结果包裹进一个 `PyResult`
* 为 `#[pymodule]` 增加函数

以下代码按相同顺序说明了上述步骤：

```rust
use pyo3::prelude::*;

#[pyfunction]
fn multiply(a: isize, b: isize) -> PyResult<isize> {
    Ok(a * b)
}

#[pymodule]
fn rust(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(multiply, m)?)?;
    Ok(())
}
```

以上内容将在名为 rust 的 Python 模块中公开 multiply 函数（在最后一个函数的名称之后）。我们还需要把正确的 [Cargo.toml](https://github.com/saidvandeklundert/pyo3/blob/main/multiply/Cargo.toml) 文件配置好。

为了使事情变得简单，请确保在 *Cargo.toml* 中输入库的名称和用 `#[pymodule]` 注释的函数名相匹配。在我的例子中，我在我的货物中 Cargo.toml 配置了以下内容：

```text
[lib]
name = "rust"
```

当这两个名称匹配时，`maturin` 构建工具将使用该名称将 Rust 库安装为Python模块。

因此，在本例中，选择 `rust` 作为包的名称后，我们可以编写以下 Python 来调用 multiply 函数：

```python
import rust

result = rust.multiply(2, 3)
print(result)
```

为了能够运行这段代码，我们需要编译 Rust 代码，并将其安装为 Python 库。这就是 `maturin` 的用武之地：

```
root@rust:/# git clone https://github.com/saidvandeklundert/pyo3.git
Cloning into 'pyo3'...
remote: Enumerating objects: 36, done.
    ...
Resolving deltas: 100% (9/9), done.
root@rust:/# cd pyo3/multiply/
root@rust:/pyo3/multiply# python3 -m venv .env
root@rust:/pyo3/multiply# source .env/bin/activate
(.env) root@rust:/pyo3/multiply# pip install maturin
Collecting maturin
    ...
Installing collected packages: toml, maturin
Successfully installed maturin-0.11.5 toml-0.10.2
(.env) root@rust:/pyo3/multiply# maturin develop
🔗 Found pyo3 bindings
🐍 Found CPython 3.9 at python
   Compiling proc-macro2 v1.0.32
    ...
   Compiling multiply v0.1.0 (/pyo3/multiply)
    Finished dev [unoptimized + debuginfo] target(s) in 23.48s
(.env) root@rust:/pyo3/multiply# python3 multiply.py
6
```

就这样！

现在我可以运行[这些](https://github.com/saidvandeklundert/pyo3/blob/main/multiply/multiply.py) 脚本：

```shell
(.env) root@rust: multiply# python3 multiply.py
6
```

## 在 Python 和 Rustn 中计算 n 阶斐波那契数列

为了计算 n 阶斐波那契数列，我将使用一个 Python 和 Rust 彼此都很类似的函数。以下是 Python 函数：

``` python
def get_fibonacci(number: int) -> int:
    """Get the nth Fibonacci number."""
    if number == 1:
        return 1
    elif number == 2:
        return 2

    total = 0
    last = 0
    current = 1
    for _ in range(1, number):
        total = last + current
        last = current
        current = total
    return total
```

为了加入同样的 Rust 代码，我们需要做如下工作：

* 在我们的 *lib.rs* 中增加函数
* 给函数标记 `#[pyfunction]`
* 在模块中增加函数

代码片段演示如下：

```rust
#[pyfunction]
fn get_fibonacci(number: isize) -> PyResult<u128> {
    if number == 1 {
        return Ok(1);
    } else if number == 2 {
        return Ok(2);
    }

    let mut sum = 0;
    let mut last = 0;
    let mut curr = 1;
    for _ in 1..number {
        sum = last + curr;
        last = curr;
        curr = sum;
    }
    Ok(sum)
}

#[pymodule]
fn rust(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(get_fibonacci, m)?)?;
    Ok(())
}
```

After pulling in the new code, we can use maturin to build and install the module and do a comparison:
引入新代码后，我们可以使用 `maturin` 构建和安装模块，并进行比较：
```
(.env) root@rust:/pyo3/pyo3# maturin develop
(.env) root@rust:/pyo3/pyo3# python -i
Python 3.9.2 (default, Feb 28 2021, 17:03:44) 
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.    
>>> import timeit
>>>
>>> timeit.timeit("get_fibonacci(5)", setup="from fib import get_fibonacci")
0.49461510000401177
>>>
>>> timeit.timeit("get_fibonacci(5)", setup="from rust import get_fibonacci")
1.1281064000068
>>>
>>>
>>> timeit.timeit("get_fibonacci(150)", setup="from fib import get_fibonacci")
9.057604000001447
>>> timeit.timeit("get_fibonacci(150)", setup="from rust import get_fibonacci")
3.5204217999998946
```

上面告诉我们，当我们调用函数来计算第5个斐波那契数时，Python的速度更快。但当我们寻找第150个斐波那契数时，生锈的速度几乎是原来的三倍。

但情况会好转。

我们还可以通过将 `--release` 作为参数添加到 `maturin` 来进行发布构建：

```
(.env) root@rust:/pyo3/pyo3# python -i
Python 3.9.2 (default, Feb 28 2021, 17:03:44) 
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.    
>>> import timeit
>>> timeit.timeit("get_fibonacci(5)", setup="from fib import get_fibonacci")
0.4583319000012125
>>> timeit.timeit("get_fibonacci(5)", setup="from rust import get_fibonacci")
0.11867309999797726
>>> timeit.timeit("get_fibonacci(150)", setup="from fib import get_fibonacci")
8.990601400000742
>>> timeit.timeit("get_fibonacci(150)", setup="from rust import get_fibonacci")
0.15236040000309004
>>>
```
 
在发布版本中，这两种情况下的 Rust 函数都要快得多。

我还比较了许多其他函数。扫描文本中的子字符串、操作文本、汇总数字以及其他各种事情。我发现这种类型的性能提升并不是那么典型。让事情加速2倍或3倍通常并不难。

## 与不同的数据类型一起工作 

调用 Rust 函数时，PyO3 会将作为函数参数传递的 Python 类型转换为 Rust 类型。它还将 Rust 函数返回的 Rust 类型转换为 Python 中可用的类型。PyO3 用户指南描述了 Python 类型和 Rust 类型之间的映射方式。让 PyO3 自动为你执行此操作，可以轻松处理不同类型的函数参数。下面用几个例子来说明这一点。

### 返回列表中数字的总和：

以下函数将对数组中的数字求和并返回结果：

```rust
#[pyfunction]
fn list_sum(a: Vec<isize>) -> PyResult<isize> {
    let mut sum: isize = 0;
    for i in a {
        sum += i;
    }
    Ok(sum)
}

#[pymodule]
fn rust(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(list_sum, m)?)?;
    Ok(())
}  
```

在 Python 中调用此函数时，我们可以传入一个整数列表，并得到一个 Python 整数作为返回值：

```
>>> rust.list_sum([10, 10, 10, 10, 10])
50
```

让我们做最后一个性能比较。在这种情况下，我们可以看到，只有当函数必须执行大量计算时，才有必要尝试加快速度。以下示例显示了对函数进行少量输入时的性能差异：

```python
>>> timeit.timeit("rust.list_sum(a_list)", setup="""
... import rust
... a_list = [x for x in range(1,10)]
... """)
0.42956949999643257
>>>
>>> timeit.timeit("sum_list(a_list)", setup="""
... from __main__ import sum_list
... a_list = [x for x in range(1,10)]
... """)
0.4579178999993019
```

几乎没有什么区别。现在，当我们将函数输入增加到 3.000 个数字时，我们可以开始看到使用 Rust 函数有一些真正的优势：

```python
>>> timeit.timeit("sum_list(a_list)", setup="""
... from __main__ import sum_list
... a_list = [x for x in range(1,3000)]
... """)
168.12326449999819
>>> timeit.timeit("rust.list_sum(a_list)", setup="""
... import rust
... a_list = [x for x in range(1,3000)]
... """)
95.2027356000035
>>>
```

关键的一点是，你能在多大程度上提高速度，实际上取决于你从 Python 外包给 Rust 的代码的哪一部分。我将跳过 Rust 和 Python 之间的任何进一步比较，并将重点放在我认为值得的几个场景上。

### 打印一个字典中的数值

下一个函数将打印 HashMap 的键和值：

```rust
use std::collections::HashMap;

#[pyfunction]
fn dict_printer(hm: HashMap<String, String>) {
    for (key, value) in hm {
        println!("{} {}", key, value)
    }
}

#[pymodule]
fn rust(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(dict_printer, m)?)?;

    Ok(())
}
```

我们现在可以从 Python 中调用这个函数，传入一个使用相同类型的字典：

```python
>>> a_dict = {
...     "key 1": "value 1",
...     "key 2": "value 2",
...     "key 3": "value 3",
...     "key 4": "value 4",
... }
>>>
>>> rust.dict_printer(a_dict)
key 2 value 2
key 1 value 1
key 3 value 3
key 4 value 4
```

因为我们定义了 Rust HashMap，将字符串用于键和值，所以我们必须在 Python 代码中做同样的事情。如果我们使用任何其他类型作为键或值，函数调用将失败。Rust 会尝试将一种类型强制转换为另一种类型：

```python
>>> try:
...     rust.dict_printer({"a": 1, "b": 2})
... except TypeError as e:
...     print(f"Caught a type error: {e}")
...
Caught a type error: argument 'hm': 'int' object cannot be converted to 'PyString'
>>>
```

### 打印一个单词 n 次

下面的函数将打印一个单词几次。此外，它还可以以反转和/或大写形式打印单词：

```rust
#[pyfunction]
fn word_printer(mut word: String, n: isize, reverse: bool, uppercase: bool) {
    if reverse {
        let mut reversed_word = String::new();
        for c in word.chars().rev() {
            reversed_word.push(c);
        }
        word = reversed_word;
    }
    if uppercase {
        word = word.to_uppercase();
    }
    for _ in 0..n {
        println!("{}", word);
    }
}
```

这个例子有点蹩脚，但它表明，编写将各种类型作为参数的函数并不太困难：

```python
>>> rust.word_printer("hello", 3, False, True)
HELLO
HELLO
HELLO
>>> rust.word_printer("eyb", 2, True, False)        
bye
bye
```

## 在 Python 中使用 Rust 结构体

令我惊讶的是，PyO3 还使在 Python 中使用 Rust 结构变得越来越容易。虽然我没有穷尽或测试所有的可能性和极端情况，但使用一个包含多种方法的结构也是非常简单的。我举了以下的例子：

```rust 
#[pyclass]
pub struct RustStruct {
    #[pyo3(get, set)]
    pub data: String,
    #[pyo3(get, set)]
    pub vector: Vec<u8>,
}
#[pymethods]
impl RustStruct {
    #[new]
    pub fn new(data: String, vector: Vec<u8>) -> RustStruct {
        RustStruct { data, vector }
    }
    pub fn printer(&self) {
        println!("{}", self.data);
        for i in &self.vector {
            println!("{}", i);
        }
    }
    pub fn extend_vector(&mut self, extension: Vec<u8>) {
        println!("{}", self.data);
        for i in extension {
            self.vector.push(i);
        }
    }
}
#[pymodule]
fn rust(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_class::<RustStruct>()?;

    Ok(())
}
```

注意这里使用了4个新的标记：

* #[pyclass]: 在结构定义之上，用于在Python中公开类。
* #[pymethods]: 在impl块上方，用于将 struct 方法公开为 Python 中的类方法。 
* #[pyo3(get, set)]: 如果希望能够在 Python 中获取或设置结构字段，请使用这些宏。
* #[new]: 在构造函数之上，这是为了能够在 Python 中将该结构作为类进行构造。

此外，我们以稍微不同的方式将结构体添加到模块中。如果使用以下选项：

```rust
m.add_function(wrap_pyfunction!(xxx, m)?)?;
```

我们现在使用如下例子：

```rust
m.add_class::<RustStruct>()?; // inserted the name of the struct that is to be exported here
```

再一次执行 `maturin develop` 后，我们已经在 Python 中准备好使用结构体了。我们可以使用结构体，就好像它是 Python 的一个类一样： 

```python
>>> rust_struct = rust.RustStruct(data="some data", vector=[255, 255, 255])
>>> rust_struct.extend_vector([1, 1, 1, 1])
Extending the vector.
>>> rust_struct.printer()
some data
255
255
255
1
1
1
1
>>> type(rust_struct)
<class 'builtins.RustStruct'>
>>> rust_struct.data
'some data'
>>> rust_struct.data = "some other data"
>>> rust_struct.data
'some other data'
```

## 把 Json 送到 Rust

发送复杂数据结构的一种方法是使用 JSON 。以下示例将 JSON 字符串封送到 Rust 结构中：

```rust
extern crate serde;
extern crate serde_json;
use serde::{Deserialize, Serialize};

#[pyfunction]
fn human_says_hi(human_data: String) {
    println!("{}", human_data);
    let human: Human = serde_json::from_str(&human_data).unwrap();

    println!(
        "Now we can work with the struct:\n {:#?}.\n {} is {} years old.",
        human, human.name, human.age,
    )
}

#[derive(Debug, Serialize, Deserialize)]
struct Human {
    name: String,
    age: u8,
}
```

在 Python 端，我将使用 Pydantic 发送一些 Json ：

```python
>>> from pydantic import BaseModel
>>> 
>>> class Human(BaseModel):
...     name: str
...     age: int
...
>>> 
>>> jan = Human(name="Jan", age=6)
>>> print(jan.json())
{"name": "Jan", "age": 6}
>>> rust.human_says_hi(jan.json())
{"name": "Jan", "age": 6}
Now we can work with the struct:
 Human {
    name: "Jan",
    age: 6,
}.
 Jan is 6 years old.
 ```

这可能是 abn 方法，可以将大型结构体映射到类似Python端的数据结构。我使用 pydantic 是因为它是我最喜欢的 Python 软件包之一。这个我在这里使用的 `.json()` 方法将类转换为json字符串。

## 让 Rust 使用来自 Python 运行时的 logger

Rust 可以使用我们在 Python 中定义的 logger 来记录日志。 可以使用 `pyo3-log` 类库。 我们首先在 `Cargo.toml` 中添加以下内容:

```
[dependencies]
pyo3-log = "0.5.0"
log = "0.4.14"
```

下面，我们创建了2个例子，用来记录一个日志信息：

```rust
use log::{debug, error, info, warn};
use pyo3_log;

#[pyfunction]
fn log_different_levels() {
    error!("logging an error");
    warn!("logging a warning");
    info!("logging an info message");
    debug!("logging a debug message");
}

#[pyfunction]
fn log_example() {
    info!("A log message from {}!", "Rust");
}
#[pymodule]
fn rust(_py: Python, m: &PyModule) -> PyResult<()> {
    pyo3_log::init();

    m.add_wrapped(wrap_pyfunction!(log_example))?;
    m.add_wrapped(wrap_pyfunction!(log_different_levels))?;

    Ok(())
}
```

然后，我们在 Python 中定义了一个 logger，我们可以在 Rust 里面使用它：

```python
>>> import logging
>>> FORMAT = "%(levelname)s %(name)s %(asctime)-15s %(filename)s:%(lineno)d %(message)s"
>>> logging.basicConfig(format=FORMAT)
>>> logging.getLogger().setLevel(logging.DEBUG)
>>> logging.info("Logging from the Python code")
INFO root 2021-11-15 20:24:46,979 <stdin>:1 Logging from the Python code>>> rust.log_example()
INFO rust 2021-11-15 20:24:51,336 lib.rs:118 A log message from Rust!
>>> rust.log_different_levels()
ERROR rust 2021-11-15 20:24:55,212 lib.rs:110 logging an error
WARNING rust 2021-11-15 20:24:55,212 lib.rs:111 logging a warning       
INFO rust 2021-11-15 20:24:55,213 lib.rs:112 logging an info message    
DEBUG rust 2021-11-15 20:24:55,213 lib.rs:113 logging a debug message   
>>>
```

## 捕获异常

在最后一个示例中，我们将在 Rust 中触发一个错误，并将其作为 Python 中的一个异常捕获。这一部分相当复杂，我采取的步骤显示在代码片段之后：

```rust
use std::fmt;

// 1
#[derive(Debug)]
struct MyError {
    pub msg: &'static str,
}

// 2
impl std::error::Error for MyError {}

// 3
impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Error from Rust: {}", self.msg)
    }
}

// 4
impl std::convert::From<MyError> for PyErr {
    fn from(err: MyError) -> PyErr {
        PyOSError::new_err(err.to_string())
    }
}

#[pyfunction]
// 5
fn greater_than_2(number: isize) -> Result<isize, MyError> {
    if number <= 2 {
        return Err(MyError {
            msg: "number is less than or equal to 2",
        });
    } else {
        return Ok(number);
    }
}

#[pymodule]
fn rust(_py: Python, m: &PyModule) -> PyResult<()> {
    // 6
    m.add_function(wrap_pyfunction!(greater_than_2, m)?)?;    

    Ok(())
}
```

1: 我们定义 `MyError` 作为自定义错误，该结构体拥有一个可以用来发送一个自定义消息的字段。 

2: 为 `MyError` 实现 `Error` 特性 。 

3: 我们为 `MyError` 实现了 `Display` 特性，并且在 `MyError` 结构体的 `msg` 字段显示消息。  

4: 为 `MyError` 实现了 `From` 特性。此特性用于在使用输入值时进行值到值的转换。 

5: 我们创建了一个命名为 `greater_than_2` 的函数。当输入值等于2或者小于2的时候，这个函数将触发错误或者异常。 

6: 我们通过正常的方式把函数加入到 Python 模块中。

现在我们转移到 Python 端，并且运行函数，触发异常：

```python
>>> rust.greater_than_2(1)
Traceback (most recent call last):
  ...
OSError: Error from Rust: number is less than or equal to 2
>>> rust.greater_than_2(3)
3
>>> rust.greater_than_2(11)
11
>>> rust.greater_than_2(-11)
Traceback (most recent call last):
  ...
OSError: Error from Rust: number is less than or equal to 2
```
 
虽然这需要相当多的代码才能实现！但在我看来还是不难。

## 结束语

使用PyO3并使用它从Python调用Rust是一种非常愉快的体验。我发现，与依赖 ffi 或 libc 相比，已经实施的人体工程学使一切变得更加容易。宏提供了很多便利，让 PyO3 在 Rust 和 Python 之间转换类型给开发过程带来了很多便利。还有一个成熟的构建工具 `maturin`。这绝对是一种工作的乐趣。

这个资源库得到了积极维护，文件记录也很好。在了解 PyO3 之后，我对在 Python 中使用 Rust 感到兴奋。

可以找到这个博客中的[代码](https://github.com/saidvandeklundert/pyo3)示例，如果你想自己运行代码，还有一个 Docker 构建文件。

关于PyO3的其他一些有价值的链接：

* [PyO3 repo](https://github.com/PyO3/pyo3)
* [PyO3 user guide](https://pyo3.rs/main/)
* [PyO3 architecture guide](https://github.com/PyO3/pyo3/blob/main/Architecture.md)
* [Maturin repo](https://github.com/PyO3/maturin)

---

> 原文链接: [http://saidvandeklundert.net/learn/2021-11-18-calling-rust-from-python-using-pyo3/](http://saidvandeklundert.net/learn/2021-11-18-calling-rust-from-python-using-pyo3/)
> 
> 翻译：[minikiller](https://github.com/minikiller)
> 选题：[minikiller](https://github.com/minikiller)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[RustCn](https://hirust.cn) 荣誉推出

