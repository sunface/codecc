# Rust 嵌入式开发

Rust 的高性能、可靠性和生产效率使其非常适合嵌入式系统。

[rust_programming_crab_sea.png](https://raw.githubusercontent.com/rustt-org/rustt-assets/main/20220326%20Use%20Rust%20for%20embedded%20development/rust_programming_crab_sea.png)

近几年，很多开发者成为了 Rust 的狂热拥趸。技术潮流变幻莫测，我们很难分清这种对新技术的激情到底是出于新鲜感还是出于对技术本身的价值的认可，[RT-Thread](https://opensource.com/article/21/7/rt-thread-smart) 社区开发者 Liu Kang 认为 Rust 是一种设计良好的语言。Kang 说 Rust 的目标是帮助开发者构建可靠和高效的软件，而 Rust 自始至终都是为了这个目标设计的。你将会了解到一些 Rust 的关键特性，在本文中，Kang 展示了一些恰好适用于嵌入式的 Rust 的特性。下面是示例：

- 高性能：速度快，内存利用率高
- 可靠性：编译过程可以消除内存错误
- 生产效率：完善的文档，用户友好的编译器、报错信息非常详细，一流的工具。有集成的包管理和构建工具，智能多编辑器支持，如自动补全、键入检测、自动格式化等等。



## 为什么在嵌入式开发中使用 Rust?

Rust 能兼顾安全性和高性能。嵌入式软件中的问题大部分跟内存有关。某种程度上，Rust 是一种面向编译器的语言，因此在编译过程中可以确保内存安全。使用 Rust 开发嵌入式设备有下面几个好处：

- 强大的静态分析
- 灵活的内存
- 无惧并发
- 可移植
- 社区驱动

本文中，我用开源的 [RT-Thread 操作系统](https://github.com/RT-Thread/rt-thread)来展示如何使用 Rust 进行嵌入式开发。

## 在 C 中怎么调用 Rust

想要在 C 代码中调用 Rust 代码，你需要把 Rust 源码打包成一个静态库文件。在编译 C 代码的过程中，把静态库链接进去。

### 用 Rust 源码创建一个静态库

这个过程分两步。

1 在 Clion 中执行 `cargo init --lib rust_to_c` 构建一个 lib 库。把下面代码添加到 `lib.rs` 文件中。下面函数计算两个 **i32** 类型的值的和然后返回计算结果：

```rust
#![no_std]
use core::panic::PanicInfo;

#[no_mangle]
pub extern "C" fn sum(a: i32, b: i32) -> i32 {
    a + b
}

#[panic_handler]
fn panic(_info:&PanicInfo) -> !{
    loop{}
}
```

2 把下面的代码添加到 `Cargo.toml` 文件中，告诉 Rustc 要生成什么类型的库：

```toml
[lib]
name = "sum"
crate-type = ["staticlib"]
path = "src/lib.rs"
```

### 交叉编译

你可以根据你的 target 进行交叉编译。如果你的嵌入式系统是基于 Arm 的，那么步骤很简单：

```bash
rustup target add armv7a-none-eabi
```

2 生成静态库文件：

```bash
$ cargo build --target=armv7a-none-eabi --release --verbose
Fresh rust_to_c v0.1.0
Finished release **[**optimized**]** target**(**s**)** **in** 0.01s
```

### 生成头文件

你还需要头文件。

1 安装 [cbindgen](https://github.com/eqrion/cbindgen)。`cbindgen` 工具可以从 Rust 库生成 C 或 C++11 头文件。

```bash
$ cargo install --force cbindgen
```

2 在你的项目文件夹下创建一个 `cbindgen.toml` 文件。

3 生成一个头文件

```bash
$ cbindgen --config cbindgen.toml --crate rust_to_c --output sum.h
```

### 调用 Rust 库文件

现在你可以调用你的 Rust 库了。

1 把生成的 `sum.h` 和 `sum.a` 文件放到 `rt-thread/bsp/qemu-vexpress-a9/applications` 目录

2 修改 `SConscript` 文件并添加一个静态库：

```makefile
   from building import *
   
   cwd     = GetCurrentDir()
   src     = Glob('*.c') + Glob('*.cpp')
   CPPPATH = [cwd]
   
   LIBS = ["libsum.a"]
   LIBPATH = [GetCurrentDir()]
   
   group = DefineGroup('Applications', src, depend = [''], CPPPATH = CPPPATH, LIBS = LIBS, LIBPATH = LIBPATH)
   
   Return('group')
```

3 在 main 函数中调用 **sum** 函数，用 `printf` 打印出 `sum` 函数的返回值。

```c
   #include <stdint.h>
   #include <stdio.h>
   #include <stdlib.h>
   #include <rtthread.h>
   #include "sum.h"
   
   int main(void)
   {
       int32_t tmp;
   
       tmp = sum(1, 2);
       printf("call rust sum(1, 2) = %d\n", tmp);
   
       return 0;
   }
```

4 在 RT-Thread [Env](https://www.rt-thread.io/download.html?download=Env) 环境中使用 `scons` 编译项目并运行：

```bash
$ scons -j6
scons: Reading SConscript files ...
scons: done reading SConscript files.
scons: Building targets ...
[...]
scons: done building targets.

$ qemu.sh
 \ | /
- RT -     Thread Operating System
 / | \     4.0.4 build Jul 28 2021
2006 - 2021 Copyright by rt-thread team
lwIP-2.1.2 initialized!
[...]
call rust sum(1, 2) = 3
```

## 加减乘除

你可以用 Rust 实现一些复杂的数学计算。在 `lib.rs` 文件中，用 Rust 语言实现加减乘除运算。

```rust
#![no_std]
use core::panic::PanicInfo;

#[no_mangle]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[no_mangle]
pub extern "C" fn subtract(a: i32, b: i32) -> i32 {
    a - b
}

#[no_mangle]
pub extern "C" fn multiply(a: i32, b: i32) -> i32 {
    a * b
}

#[no_mangle]
pub extern "C" fn divide(a: i32, b: i32) -> i32 {
    a / b
}

#[panic_handler]
fn panic(_info:&PanicInfo) -> !{
    loop{}
}
```

把你的库文件和头文件编译好后放到应用程序的目录下。使用 `scons` 来编译。如果链接过程中有任何错误，你可以在官方的 [GitHub page](https://github.com/rust-lang/compiler-builtins/issues/353) 上找到解决方案。

修改 `rtconfig.py` 文件，添加链接参数 `--allow-multiple-definition`：

```bash
       DEVICE = ' -march=armv7-a -marm -msoft-float'
       CFLAGS = DEVICE + ' -Wall'
       AFLAGS = ' -c' + DEVICE + ' -x assembler-with-cpp -D__ASSEMBLY__ -I.'
       LINK_SCRIPT = 'link.lds'
       LFLAGS = DEVICE + ' -nostartfiles -Wl,--gc-sections,-Map=rtthread.map,-cref,-u,system_vectors,--allow-multiple-definition'+\
                         ' -T %s' % LINK_SCRIPT
   
       CPATH = ''
       LPATH = ''
```

编译并执行 QEMU 来看一下效果。

## 在 Rust 中调用 C

可以在 C 代码中调用 Rust，那么可以在 Rust 代码中调用 C 吗？下面是一个在 Rust 代码中调用 C 函数 `rt_kprintf` 的例子：

首先，修改 `lib.rs` 文件：

```rust
    // The imported rt-thread functions list
    extern "C" {
        pub fn rt_kprintf(format: *const u8, ...);
    }
   
    #[no_mangle]
    pub extern "C" fn add(a: i32, b: i32) -> i32 {
        unsafe {
            rt_kprintf(b"this is from rust\n" as *const u8);
        }
        a + b
    }
```

其次，生成库文件：

```bash
$ cargo build --target=armv7a-none-eabi --release --verbose
Compiling rust_to_c v0.1.0
Running `rustc --crate-name sum --edition=2018 src/lib.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type staticlib --emit=dep-info,link -C opt-level=3 -C embed-bitcode=no -C metadata=a
Finished release [optimized] target(s) in 0.11s
```

为了能运行起来，你需要把从 Rust 生成的库文件复制到应用程序的目录并重新构建：

```bash
$ scons -j6 scons: Reading SConscript files ... scons: done reading SConscript files. [...]
scons: Building targets ... scons: done building targets.
```

再运行 QEMU 查看你的嵌入式镜像中的结果。

## 你可以尝试更多

在嵌入式开发开发中，你可以使用 Rust 的所有特性而无需牺牲灵活性和稳定性。现在就在你的嵌入式系统中尝试下 Rust 吧。你可以参考 RT-Thread 项目的 [YouTube 频道](https://www.youtube.com/channel/UCdDHtIfSYPq4002r27ffqP)来了解更多关于嵌入式 Rust（以及 RT-Thread）的信息。请记住，嵌入式也可以是开放的。

---

via: https://opensource.com/article/21/10/rust-embedded-development

作者：[Alan Smithee](https://opensource.com/users/alansmithee)
译者：[lxbwolf](https://github.com/lxbwolf)

本文由 [Rustt](https://github.com/studyrs/rustt) 翻译，[StudyRust](https://github.com/studyrs) 荣誉推出
