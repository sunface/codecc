> 原文链接: https://www.philipdaniels.com/blog/2019/rust-equality-and-ordering/
> 
> **翻译：[朕与将军解战袍](https://github.com/a1393323447)**
>
> 选题：[trdthg](https://github.com/trdthg)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

# 在 Rust 中手动实现 `PartialEq` , `Eq` , `Hash` , `PartialOrd` 和 `Ord`

## 介绍

这篇是一篇简短的教程，教你如何手动实现 Rust 中关于相等、哈希和排序的 trait 。通常你可以通过 `derive` 自动实现：

```rust
#[derive(PartialEq, Eq, Hash, PartialOrd, Ord)]
pub MyStruct {
    ...
}
```

但有时候你想自己来，可能是因为你的版本比自动生成的性能更好，可能是你想要更明确地说明"相等"意味着什么，可能你想表达 `MyStruct` 和 `SomeOtherStruct` 的实例之间的关系，而自动生成的版本表达不了。

在这篇文章中，我会以下面这个简单的结构体为例：

```rust
#[derive(Debug, Default, Clone)]
pub struct FileInfo {
    pub path: PathBuf,
    pub contents: String,
    pub is_valid_utf8: bool,
}
```

它表示位于 `path` 的一些文本文件的内容，内容作为字符串加载，并带有一个标志 `flag` ，用于指示内容是否为有效的 UTF-8 格式。

在使用该结构的上下文中，`path` 总是唯一的——一个文件只会被加载一次。我喜欢用实体关系建模的术语来看待 Rust 的数据：`path` 就像一个主键，然后结构体的其它字段就是一些确定的属性。因此，identity 的概念也被封装进 `path` 中——如果 `path` 相等，那么两个 `FileInfo` 实例就相等。

## 相等：`PartialEq` 和 `Eq`

如果想要表达自定义类型相等的意义，你必须实现 `PartialEq` 。实现之后，自定义类型就可以使用 `x == y` 和 `x != y` 。

为 `FileInfo` 实现 `PartialEq` 比较简单，只需委托给 `path` 的 `PartialEq` 实现：

```rust
impl PartialEq for FileInfo {
    fn eq(&self, other: &Self) -> bool {
        self.path == other.path
    }
}
```

你也可以（通常应该）实现 `Eq` 。

`Eq` 的定义为空，不包含任何方法：

```rust
trait Eq: PartialEq<Self> {}
```

然而，实现 `Eq` 并不是没有用的—— `Eq` 是一种标记，表示该类型可以被用作 `hashmap` 中的键。

可以用一个空 `impl` 块手动实现它：

```rust
impl Eq for FileInfo {}
```

更简单的方法是使用 `#[derive(Eq)]` 。

那什么情况下，你 **不** 应该实现 `Eq` 呢？这种情况很少见。`Eq` 表示[等价关系](https://en.wikipedia.org/wiki/Equivalence_relation)，因此所有的实例 `x` 都需要满足下面三种性质：

- [传递性](https://en.wikipedia.org/wiki/Transitive_relation)：若 `x == y` 且 `y == z` ，则 `x == z`
- [对称性](https://en.wikipedia.org/wiki/Symmetric_relation)：若 `x == y`， 则 `y == x`
- [自反性](https://en.wikipedia.org/wiki/Reflexive_relation)：`x == x` 为真

大多数的数据都满足这三个性质。主要的例外——也是 Rust 标准库中位唯一的例外——是浮点数，`NaN` 不满足自反性（ `IEEE` 浮点数编码标准中规定 `NaN` 不等于任何数，包括自己）。

> **总结**  在为自定义类型实现 `PartitalEq` 的同时，也使用 `#[derive(Eq)]` ，除非该类型不满足等价关系。

## 哈希

对一个值进行哈希和相等的概念密切相关，因此，如果你实现了自己的 `PartialEq`，那么你也应该实现 `Hash` 。

> 这个关系必须成立：若 `x == y` ，则 `hash(x) == hash(y)`

如果你违背了上述规则，那么你的自定义类型的值作为 [HashMap](https://doc.rust-lang.org/std/collections/struct.HashMap.html) 和  [HashSets](https://doc.rust-lang.org/std/collections/struct.HashSet.html) 的键时，不能正常工作。

也就是说，如果两个值根据 `PartialEq` 判断是相等的，那么它们应该有相同的哈希值。反之**不成立**——两个值具有相同的哈希值并**不**意味着它们相等。这种情况被称为“哈希冲突”。在某些域（domain）中，这是不可避免的，因为可能的值比不同的 `u64` 值要多得多（哈希值的类型是 `u64` ）。举一个小例子，一个有两个类型为 `u64` 的成员的结构体，它的实例有 `u64::MAX * u64::MAX` 种可能，这个数量远大于 `u64::MAX`。因此，不可能将这个结构体的每一个实例都映射到一个独特的哈希值。

我们可以使用和实现 `PartialEq` 差不多的方式（将计算任务委托给 `path` ） 实现 `Hash`：

```rust
impl Hash for FileInfo {
    fn hash<H: Hasher>(&self, hasher: &mut H) {
        self.path.hash(hasher);
    }
}
```

这种委托技术适用于所有类型，因为标准库中的所有基本类型都实现了 `PartialEq` 和 `Hash` 。

## 排序：`PartialOrd` 和 `Ord`

值的相对顺序是使用运算符 `<` ，`<=` ，`>=` 和 `>` 计算的。为了使用这些运算符，你需要为自定义类型实现 [PartialOrd](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html) 。

> 在实现 `PartialOrd` 之前，必须实现 `PartialEq` 。

例如：

```rust
impl PartialOrd for FileInfo {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        self.path.partial_cmp(&other.path)
    }
}
```

[Ordering](https://doc.rust-lang.org/std/cmp/enum.Ordering.html) 是一个包含三个成员的简单枚举：

```rust
pub enum Ordering {
    Less,
    Equal,
    Greater,
}
```

你可能好奇为什么 `partial_cmp` 返回一个 `Option` 而不是 `Ordering` 。再次以浮点数为例，因为 [NaN](https://en.wikipedia.org/wiki/NaN) 不是一个数字，一些表达式如 `3.0 < NaN` 是没有意义的。这种情况下，`partial_cmp` 返回 `None` 。这种情况，在标准库中只发生在浮点数中。

事实上，`partial_cmp` 返回 `Option<Ordering>` 是有价值的：可能两个值 `x` 和 `y` 没有固定的顺序。在实践中，这意味着实现 `PartialOrd` 不足以让值可排序。你还需要实现  [Ord](https://doc.rust-lang.org/std/cmp/trait.Ord.html) 。

> 为了让值可以被排序，必须实现 **Ord** 。

> 在实现 **Ord** 之前，必须实现 **PartialOrd**，**Eq** 和 **PartialEq** 。

对于 `FileInfo` 来说，我们再次委托给成员：

```rust
impl Ord for FileInfo {
    fn cmp(&self, other: &Self) -> Ordering {
        self.path.cmp(&other.path)
    }
}
```

现在，我们可以对 `Vec<FileInfo>` 进行排序了。

## 扩展到一个以上的成员

你可能会好奇，在需要比较多于一个成员（用实体-关系术语表述，有一个复合主键）的时候，怎么样实现上述的 trait 。下面的模式是可行的：

```rust
impl PartialEq for ExtendedFileInfo {
    fn eq(&self, other: &Self) -> bool {
        // Equal if all key members are equal
        self.path == other.path &&
        self.attributes == other.attributes
    }
}

impl Hash for FileInfo {
    fn hash<H: Hasher>(&self, hasher: &mut H) {
        // Ensure we hash all key members.
        self.path.hash(hasher);
        self.attributes.hash(hasher);
    }
}
```

排序有点棘手——你需要先比较第一个字段，如果结果不是 `Equal`，那么就完成，但如果结果是`Equal`，则需要继续比较下一个字段，依此类推。这部分留给读者作为练习!

## 拓展到不同类型的比较

在上面的讨论中，我忽略了一些东西。在上面的每一个例子中，都是在比较 `FileInfo` 。事实上，我们并不一定要这样比较。这些 trait （ `PartialEq` 、`PartialOrd` ）接受一个类型参数 `Rhs` （"right hand side" 的简写）。可以让你将 `FileInfo` 和（比如说） `Path` 比较，或者（更通常的情况）将复数和 `f64` 作比较。（ `Ord` 和上面提到的情况相对：`self` 和 `Rhs` 必须是相同的类型，译者建议读者去看一下 `Ord` 的定义）。

`Rhs` 之所以没有出现在上面的示例代码里，是因为它默认和 `Self` 是同一类型。下面是从标准库中 `PartialEq` 的完整定义：

```rust
pub trait PartialEq<Rhs: ?Sized = Self> {
    fn eq(&self, other: &Rhs) -> bool;
    fn ne(&self, other: &Rhs) -> bool { !self.eq(other) }
}
```

为了能够进行跨类型的相等比较，例如比较 `FileInfo` 和 `&str` ，你可以这样做：

```rust
impl PartialEq<&str> for FileInfo {
    fn eq(&self, other: &&str) -> bool {
        match self.path.to_str() {
            Some(s) => s == *other,
            None => false
        }
    }
}
```

注意到，函数 `eq` 的参数 `other` 总是一个类型的共享引用，导致在为 `&str` 实现的时候，最终得到的是一个二级引用 `&&str` 。所以我们不得不解引用，通过 `s == *other` 作比较。

## 关于性能的说明

你可能会想，手动实现的这些 trait 和使用自动派生的哪个更高效？嗯...很难说。如果你知道一个大结构体中只有几个字段和比较有关，那么手动实现的版本可能会更高效。另一方面，尽管自动派生的代码可能会检查结构中的每个字段，但它使用了短路布尔表达式，所以可能会在查看第一个字段后停止（比如 `FileInfo` 里的 `path`）。因此，在实践中可能和手动实现的一样快。而 `Hash` 是例外——如果你可以只对一、两个简单成员而不是一长串成员作哈希，那你可以轻易打败内置实现。

自动派生代码还有两个妙处。第一，它为 trait 所有方法生成自定义实现，包括那些有默认实现的方法。例如，自动实现 `PartialOrd` 不仅仅生成了 `partial_cmp` ，还生成了 `lt`， `le`， `ge` 和`gt` 。第二，它为每一个方法都加上了 `#[inline]` 。

你可以使用  [cargo expand](https://github.com/dtolnay/cargo-expand) 将 `#[derive()]` 生成的代码输出到 `stdout` 。你可以尝试在下面的程序中删除一些自定义实现，并用 `#[derive]` 代替。

## 完整的程序

```rust
use std::path::PathBuf;
use std::hash::{Hash, Hasher};
use std::collections::HashMap;
use std::cmp::Ordering;

#[derive(Debug, Default, Clone, Eq)]
pub struct FileInfo {
    pub path: PathBuf,
    pub contents: String,
    pub is_valid_utf8: bool,
}

impl FileInfo {
    fn new<P: Into<PathBuf>>(path: P) -> Self {
        Self {
            path: path.into(),
            ..Default::default()
        }
    }
}

impl PartialEq for FileInfo {
    #[inline]
    fn eq(&self, other: &Self) -> bool {
        self.path == other.path
    }
}

impl Hash for FileInfo {
    #[inline]
    fn hash<H: Hasher>(&self, hasher: &mut H) {
        self.path.hash(hasher);
    }
}

impl PartialOrd for FileInfo {
    #[inline]
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        self.path.partial_cmp(&other.path)
    }
}

impl Ord for FileInfo {
    #[inline]
    fn cmp(&self, other: &Self) -> Ordering {
        self.path.cmp(&other.path)
    }
}

impl PartialEq<&str> for FileInfo {
    #[inline]
    fn eq(&self, other: &&str) -> bool {
        match self.path.to_str() {
            Some(s) => s == *other,
            None => false
        }
    }
}

fn main() {
    // 用于演示不同的 trait 的功能
    // 试着注释掉 'impl' 块，看看它们没有实现时的各种编译错误。 

    let f1 = FileInfo::new("/temp/foo");
    let f2 = FileInfo::new("/temp/bar");

    // ------------------------------------------------------------------------------
    // 用于演示 PartialEq。它让我们可以使用 `==` 和 `!=` 。
    if f1 == f2 {
        println!("f1 and f2 are equal");
    } else {
        println!("f1 and f2 are NOT equal");
    }

    if f1 != f2 {
        println!("f1 and f2 are NOT equal");
    } else {
        println!("f1 and f2 are equal");
    }

    // ------------------------------------------------------------------------------
    // 用于演示 Hash。注意，HashMap 会获取它的键的所有权——键会被移动到 HashMap 里
    let mut hm = HashMap::new();
    hm.insert(f1, 200);
    hm.insert(f2, 500);
    // 为了避免使讨论复杂化，创建一个新的 FileInfo 用于查找
    // 在实践中，需要实现 Borrow 来避免临时变量。
    let f_lookup = FileInfo::new("/temp/foo");
    let file_size = hm[&f_lookup];
    println!("f1 has a size of {} bytes", file_size);

    // ------------------------------------------------------------------------------
    // 用于演示 PartialEq。它让我们可以使用 `<`, `<=`, `>=` 和 `>`。

    // 创建一些新的 FileInfo，因为之前创建的都被移动到了 HashMap 里
    let f1 = FileInfo::new("/temp/foo");
    let f2 = FileInfo::new("/temp/bar");

    if f1 < f2 {
        println!("f1 is less than f2");
    } else {
        println!("f1 is not less than f2");
    }

    if f1 > f2 {
        println!("f1 is greater than f2");
    } else {
        println!("f1 is not greater than f2");
    }

    // ------------------------------------------------------------------------------
    // 用于演示 Ord 。它让解锁了和排序有关的功能
    let mut v = vec![f1, f2];
    v.sort();
    println!("v after sorting = {:#?}", v);

    // ------------------------------------------------------------------------------
    // 用于演示跨类型相等比较测试
    let f1 = FileInfo::new("/temp/foo");
    if f1 == "/temp/foo" {
        println!("The path in f1 is equal to the str value \"/temp/foo\"");
    } else {
        println!("Nope, comparisons to strings are not working as they should be.");
    }
}
```