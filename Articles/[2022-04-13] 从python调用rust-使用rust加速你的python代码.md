> 原文链接: http://saidvandeklundert.net/learn/2021-11-06-calling-rust-from-python/
>
> **翻译：[ChenYe](https://github.com/Ch3nYe)**
> 选题：[minikiller](https://github.com/minikiller)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

# 从python调用rust-使用rust加速你的python代码

  

Python 是一门很棒的语言！至少，在我看来是这样。它有着丰富的第三方库，可以非常好的完成很多工作。但作为解释型语言，Python 并不是最快的。如果要加速代码运行速度，你可以使用 C/C++ 。这种编程语言上的扩展使得你可以在 Python 中调用这些其他语言写的函数。这种方式提供了更快的执行速度，同时你也不会被全局解释器锁困扰。

  

除了使用 C 或 C++ 之外，我们还可以使用 Rust 。Rust 提供了 FFI（Foreign Function Interface 外部函数接口），它允许你将 Rust 函数导出到 C 语言。通过让 Python 使用这些导出的 C 函数，你可以将 Python 和 Rust 缝合在一起。如此一来就可以在需要的地方使用外部的 Rust 程序。

  

在本文中，我将介绍一些关于如何从 Python 调用多个 Rust 函数的基本示例。在 Rust 一侧，我将使用 std 中的 `ffi`，在 Python 一侧，我将仍然使用 `ctypes`：

![Calling Rust from Python](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220413-calling-rust-from-python/calling_rust_from_python_std_ffi_and_ctypes.webp)

  

## 在 Python 中调用 Rust 函数打印一个字符串.

首先，我们将编写一个打印字符串的 Rust 函数。下图说明具体发生了什么：

![Python string to Rust](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220413-calling-rust-from-python/python_string_to_rust_via_c.webp)

  

在 Python 侧我们做了以下几件事：

- Rust 导出为 C 编译生成的库文件被导入
- 将 string 转换为 bytes
- 调用从 Rust 中导入的函数
- Python 侧的 Rust 函数的参数是 UTF-8 编码的 bytes

  

在 Rust 侧我们做了以下几件事：

- 创建一个新的库
- 通过 **Cargo.toml** 指明我们要构建一个 Rust 动态库
- 用 Rust 编写与 C 兼容的外部函数接口
- 读取经由 C 传递来的 Python 侧的输入，类型是 `Char *` 
- 编译 Rust 库

  

完成之后，我们就可以运行代码了！

  

### Python 侧.

```python
import ctypes

rust = ctypes.CDLL("target/release/librust_lib.so")


if __name__ == "__main__":
    SOME_BYTES = "Python says hi inside Rust!".encode("utf-8")
    rust.print_string(SOME_BYTES)
```

  

首先，我们导入 **ctypes** ，然后，我们指定 **.so** 文件的问题。在我这里，命名为 **print_string.py** 的 Python 脚本被放在构建 rust 库的目录下。在我运行完 `cargo build --release` 后，会产生以下目录结构：

```
rust_lib/
├── target/
|   ├── lib.rs
├── target/
│   ├── release/
│       ├── librust_lib.so
├── print_string.py
```

  

在 Python 代码的 `__main__` 部分，我先通过调用 `.encode("utf-8")` 将字符串转换为 bytes 。这一步将得到 UTF-8 编码的 bytes 。然后，我通过 `rust.print_string(SOME_BYTES)` 调用 Rust 函数，`rust` 是对我们之前加载的库的引用。`print_string` 是导出的 Rust 函数名。`SOME_BYTES` 作为参数传递给目标函数。



我们还可以以另一种方式为 Rust 函数提供参数。试试将以下内容添加到 **print_string_script.py** ：

```python
    rust.print_string(
        ctypes.c_char_p("Another way of sending strings to Rust via C.".encode("utf-8"))
    )
```

在这种情况下，我们使用 `ctypes.c_char_p` 将值传递给 rust 函数。 `c_char_p` 是一个指向字符串的指针。

  

### Rust 侧.

在 Rust 侧，我们先使用 `cargo new --lib` 创建一个库。然后我们编辑 **Cargo.toml** 文件：

```yaml
[package]
name = "rust_from_python"
version = "0.1.0"
authors = ["Said van de Klundert"]


[lib]
name = "rust_lib"
crate-type = ["dylib"]
```

这表明我们正在构建一个 Rust 动态库，并且我们将使用 **libc** 。

然后，我们编写 **lib.rs** 文件：

```rust
use std::ffi::CStr;
use std::os::raw::c_char;
use std::str;

/// Turn a C-string into a string slice and print to console:
#[no_mangle]
pub extern "C" fn print_string(c_string_ptr: *const c_char) {
    let bytes = unsafe { CStr::from_ptr(c_string_ptr).to_bytes() };
    let str_slice = str::from_utf8(bytes).unwrap();
    println!("{}", str_slice);
}
```

第一行将 [CStr](https://doc.rust-lang.org/std/ffi/struct.CStr.html) 引入，这是一个表示 C 字符串的借用的结构体，它的文档是这样说的：

```
This type represents a borrowed reference to a nul-terminated array of bytes. It can be constructed safely from a &[u8] slice, or unsafely from a raw *const c_char. It can then be converted to a Rust &str by performing UTF-8 validation, or into an owned CString.
```

我们将用它，将 **CStr** 转换成 Rust 中的 **&str** 。

  

第二行 `use std::os::raw::c_char;` ，将 [c_char](https://doc.rust-lang.org/std/os/raw/type.c_char.html) 类型引入。

  

第三行 `use std::os::raw::c_char;` ，使得我们能访问 **from_utf8** ，使用这个方法将 **bytes** 转换为 Rust 的 **&str** 。

  

现在，我们看看这个函数：

```rust
#[no_mangle]
pub extern "C" fn print_string(c_string_ptr: *const c_char) {
    let bytes = unsafe { CStr::from_ptr(c_string_ptr).to_bytes() };
    let str_slice = str::from_utf8(bytes).unwrap();
    println!("{}", str_slice);
}
```

`extern` 关键字用于创建 FFI（Foreign Function Interface）。它可用于调用其他语言的函数或创建允许其他语言调用 Rust 函数接口。

  

以下引用来自 book of Rust：

```
We also need to add a #[no_mangle] annotation to tell the Rust compiler not to mangle the name of this function. Mangling is when a compiler changes the name we’ve given a function to a different name that contains more information for other parts of the compilation process to consume but is less human readable. Every programming language compiler mangles names slightly differently, so for a Rust function to be nameable by other languages, we must disable the Rust compiler’s name mangling.
```

函数参数类型指定为 `c_string_ptr: *const c_char` 。其中，`*const` 表示原始指针，`c_char` 是 C 语言 char 类型。送一组合起来，就是一个指向 `c_char` 的原始指针。

  

为了解引用这个原始指针，我们使用 `unsafe` 代码块。在这个代码块中，我们使用 `CStr::from_ptr` ，它会使用 CStr 包装器包装由参数传进来的指针。之后，我们调用 `to_bytes()` 将 C 字符串转换为 byte 切片。这些字符都将存储在 `bytes` 变量中。

  

将字符切片传给 `str::from_utf8()` 并解包返回值，就会得到一个 `&str` 类型的值，最后将其打印出来。

  

当上述一切都写好之后，使用命令 `cargo build --release` 就可以得到以下目录结构：

```
rust_lib/
├── target/
|   ├── lib.rs
├── target/
│   ├── release/
│       ├── librust_lib.so
├── print_string.py
```

  

从 rust_lib 目录，我们运行 Python 脚本：

```
root@rust:/python/rust/rust_lib# ./print_string.py  
Python says hi inside Rust!
```

此时，我们就已经从 Python 调用了一个 Rust 函数并将一个字符串打印到控制台。

  

## 在 Python 中调用一个 Rust 函数打印一个整数.

当从 Python 向 Rust 传递参数时，我们需要考虑 Python、C 和 Rust 中使用的类型。现在我们试试在屏幕上打印一个整数。首先，我们创建 Python 脚本：

```
import ctypes

rust_lib = ctypes.CDLL("target/release/librust_lib.so")

if __name__ == "__main__":
    SOME_BYTES = (2).to_bytes(4, byteorder="little")
    rust_lib.print_int(SOME_BYTES)
```

  

接下来，我们在 Rust 端并将以下内容添加到我们的 lib.rs 中：

```rust
use std::os::raw::c_int;

// Turn a C-int into a &i32 and print to console:
#[no_mangle]
pub extern "C" fn print_int(c_int_ptr: *const c_int) {
    let int_ptr = unsafe { c_int_ptr.as_ref().unwrap() };
    println!("Python gave us number {}", int_ptr);
}
```

我们将另一种数据类型 `c_int` 引入 。该函数将指向 C 整型的指针作为参数。在我们执行命令 `cargo build --release` 之后，我们可以运行 Python 脚本：

```
root@rust:/python/rust/rust_lib# python3 print_number.py  
Python gave us number 2
```

Rust、Python 和 C 中有许多类型。这些类型之间的对应可能会把人搞的很头疼！

  

## 从 Python 调用多类型的 Rust 函数

现在，我们使用 Python 调用一个 `start_procedure` 。为了专注研究跨语言调用，该函数仅仅获取一个结构并返回另一个结构。在 Python 侧，我们使用 Pydantic basemodel 来创建 Rust 函数所需的输入。Pydantic basemodel 将具有与 Rust 结构相同的字段。我们对 Rust 的返回值做同样的事情。我们创建了一个 Pydantic basemodel ，它是 Rust struct 在 Python 侧的镜像。 Rust struct 和 Pydantic basemodel 将包含多种不同类型的字段。这是我们将以最简单的方式（至少在我看来）处理的这件事：使用 C 语言中的 `Char *`。

![Pydantic BaseModel to Rust Struct](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220413-calling-rust-from-python/model_to_struct.webp)



上图说明了该过程会发生什么。在 Rust 和 Python 之间，JSON 字符串用于传递值。在 Rust 侧，我们将 JSON 解析到相应的结构中。在 Python 侧，我们将 JSON 加载到相应的 Pydantic basemodel 中。这样做的优点是我们只需要在 C 中使用 `Char *`，而不必使用 C 结构体或 C 中的其他类型。

  

### Python 侧.

我们将使用的 Python 脚本如下：

```python
import ctypes
from pydantic import BaseModel
from typing import List

rust = ctypes.CDLL("target/release/librust_lib2.so")


class ProcedureInput(BaseModel):
    timeout: int
    retries: int
    host_list: List[str]
    action: str
    job_id: int


class ProcedureOutput(BaseModel):
    job_id: int
    result: str
    message: str
    failed_hosts: List[str]


if __name__ == "__main__":
    procedure_input = ProcedureInput(
        timeout=10,
        retries=3,
        action="reboot",
        host_list=["server1", "server2"],
        job_id=1,
    )

    ptr = rust.start_procedure(procedure_input.json().encode("utf-8"))

    returned_bytes = ctypes.c_char_p(ptr).value

    procedure_output = ProcedureOutput.parse_raw(returned_bytes)
    print(procedure_output.json(indent=2))
```

我们先加载了目标库，然后定义了两个类。这脸各类是 Pydantic basemodels ，它在运行时强制执行类型提示。**ProcedureInput** 类是 Rust 函数的参数，**ProcedureOutput** 类是我们希望从 Rust 函数中返回的东西。

  

完成定义后，我们实例化一个 **ProcedureInput** 对象。我们使用 `rust.start_procedure` 调用 Rust 函数。当我们进行 Rust 函数调用时，`procedure_input.json().encode("utf-8")` 这个表达式将会以 JSON 字符串的形式输出类实例的字段，并将字符串转成 bytes 。

  

我们会收到从 Rust 中返回的 **ptr** 。接下来，将其转换为 bytes，并且使用 parse_raw 方法将使我们能够从这些 bytes 构建 **ProcedureOutput** 实例。

  

当我们运行 Python 代码时，将得到以下输出：

```
root@rust:/python/rust/rust_lib# python3 call_rust_function.py 
{
  "job_id": 1,
  "result": "success",
  "message": "1 host failed",
  "failed_hosts": [
    "server1"
  ]
}
```

  

### Rust 侧.

以下是 Rust 侧的代码：

```rust
extern crate serde;
extern crate serde_json;

use serde::{Deserialize, Serialize};
use std::ffi::CStr;
use std::ffi::CString;
use std::os::raw::c_char;
use std::str;

#[derive(Debug, Serialize, Deserialize)]
struct ProcedureInput {
    timeout: u8,
    retries: u8,
    host_list: Vec<String>,
    action: String,
    job_id: i32,
}

#[derive(Serialize, Deserialize)]
struct ProcedureOutput {
    result: String,
    message: String,
    failed_hosts: Vec<String>,
    job_id: i32,
}

#[no_mangle]
pub extern "C" fn start_procedure(c_string_ptr: *const c_char) -> *mut c_char {
    let bytes = unsafe { CStr::from_ptr(c_string_ptr).to_bytes() };
    let string = str::from_utf8(bytes).unwrap();
    let model: ProcedureInput = serde_json::from_str(string).unwrap();
    let result = long_running_task(model);
    let result_json = serde_json::to_string(&result).unwrap();
    let c_string = CString::new(result_json).unwrap();
    c_string.into_raw()
}

fn long_running_task(model: ProcedureInput) -> ProcedureOutput {
    let result = ProcedureOutput {
        result: "success".to_string(),
        message: "1 host failed".to_string(),
        failed_hosts: vec!["server1".to_string()],
        job_id: model.job_id,
    };
    return result;
}
```

在这段代码中，我们先定义了两个结构体作为 Pydantic basemodels 在 Rust 侧的镜像：

```rust
#[derive(Debug, Serialize, Deserialize)]
struct ProcedureInput {
    timeout: u8,
    retries: u8,
    host_list: Vec<String>,
    action: String,
    job_id: i32,
}

#[derive(Serialize, Deserialize)]
struct ProcedureOutput {
    result: String,
    message: String,
    failed_hosts: Vec<String>,
    job_id: i32,
}
```

然后，定义了 **start_procedure** 函数：

```rust
#[no_mangle]
pub extern "C" fn start_procedure(c_string_ptr: *const c_char) -> *mut c_char {
    let bytes = unsafe { CStr::from_ptr(c_string_ptr).to_bytes() };
    let string = str::from_utf8(bytes).unwrap();
    let model: ProcedureInput = serde_json::from_str(string).unwrap();
    let result = long_running_task(model);
    let result_json = serde_json::to_string(&result).unwrap();
    let c_string = CString::new(result_json).unwrap();
    c_string.into_raw()
}
```

与之前相似，我们先从原始指针构建一个字符串。然后我们得到了从 Python 传过来的 JSON 字符串，我们将其解析为 **ProcedureInput** 结构体。我们将该结构传递给 `long_running_task` 方法，它会为我们返回一个 **ProcedureOutput** 对象。使用 `serde_json::to_string` 可以将结构体转为 JSON 字符串。我们使用 `CString::new` 从字节容器创建新的兼容 C 的字符串。使用 `into_raw()` 方法将 `c_string` 的所有权转移给 C 中的调用者。`into_raw()` 方法返回一个指针，我们之前在 Python 侧用它来读取返回值。

  

## 从 Python 调用带有内存泄漏的 Rust 函数

如果我们把 `start_procedure` 放在一个 while 循环里，让它运行一段时间，我们可以看到进程会逐渐开始消耗越来越多的内存。是因为从 Rust 返回的值没有被清理。

  

通过对 Python 脚本进行以下更改，我们将持续调用 `start_procedure` ：

```Python
while True:
    ptr = rust.start_procedure(procedure_input.json().encode("utf-8"))
    returned_bytes = ctypes.c_char_p(ptr).value
    procedure_output = ProcedureOutput.parse_raw(returned_bytes)
    print(procedure_output.json(indent=2))
```

调用脚本，我们可以看到 Python 脚本的内存使用量在慢慢增加。

  

### 修复内存泄露问题

为了解决这个内存泄漏问题，我们首先需要在 Rust 中创建一个清理内存的函数：

```rust
#[no_mangle]
pub extern "C" fn free_mem(c_string_ptr: *mut c_char) {
    unsafe { CString::from_raw(c_string_ptr) };
}
```

此函数将使用 `from_raw` 获取已转移到 C 的 `CString` 的所有权。当函数结束时，变量将不被任何所有者拥有，值被删除，释放内存。

  

我们需要在 Python 侧调用这个函数。 `free_mem` 函数的输入应该是 Rust 返回给 Python 的值。

  

以下 Python 可以连续运行而不会泄漏内存：

```python
while True:
    ptr = rust.start_procedure(procedure_input.json().encode("utf-8"))
    returned_bytes = ctypes.c_char_p(ptr).value
    procedure_output = ProcedureOutput.parse_raw(returned_bytes)
    print(procedure_output)
    rust.free_mem(ptr)
```

`start_procedure` 返回的值就是我们传递给 free_mem 的值。另一件需要注意的事情是我们在处理完值后调用 `free_mem`。

  

在下面的示例中，我们在 Python 代码中使用它之前释放该值：

```Python
ptr = rust.start_procedure(procedure_input.json().encode("utf-8"))
returned_bytes = ctypes.c_char_p(ptr).value
rust.free_mem(ptr)
procedure_output = ProcedureOutput.parse_raw(returned_bytes)
```

  

在让 Rust 释放内存之后，我们尝试在 Python 侧读取相同的字节。当我们运行这段代码时，我们完成了一次 double free ：

```Python
root@rust:/python/rust/rust_lib# python3 call_rust_continuously_free_mem.py
job_id=1 result='success' message='1 host failed' failed_hosts=['server1']
free(): double free detected in tcache 2
Aborted
```

## 总结

从 Python 中使用 Rust 是我这段时间一直想尝试的事情。即使我已经研究 Rust 一段时间了，完全转向 Rust 对我现在参与的任何项目都没有任何意义。在某些情况下，我使用的库在 Rust 中不可用，而在其他情况下，我正在工作的项目太大而无法在 Rust 中重写。另外，其实我对于 Rust 还处于学习阶段。

像这样从 Python 调用 Rust 为我铺平了道路：

- 将 Rust 合并到现有的 Python 项目中并从小处着手
- 对现有的 Python 项目更有信心，知道如果速度真的成为问题，我可以用 Rust 来加速
- 同时使用 Python 和 Rust！

  

Rust 社区的人们在使用 Rust 来加速 Python 方面做出了很多努力。在很多情况下，你会看到这些项目使用 pyo3。这个库提供了 “Python 的 Rust 绑定，包括用于创建原生 Python 扩展库的工具” 。使用 pyo3 的一个例子是 [polars](https://www.pola.rs/) 。

  

这篇文章中的例子可以在[这里](https://github.com/saidvandeklundert/rust_from_python)找到。

  

注意：写这篇文章时，我使用了 CPython 3.9.2、Pydantic 版本 1.8.2 和 Rust 1.56.0。

  
