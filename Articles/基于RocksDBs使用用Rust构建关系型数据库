> 原文链接: https://blog.petitviolet.net/post/2021-05-25/building-database-on-top-of-rocksdb-in-rust
>
> **翻译：[Cerdore](https://github.com/cerdore)**
> 选题：[Cerdore](https://github.com/cerdore)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

# 基于RocksDB用rust构建关系型数据库

最近，我在RocksDB 之上使用 Rust 开发了一个叫做 rrrdb 的迷你数据库，RocksDB是一个著名的 Key-Value 存储引擎。尽管我不认为 rrrdb 是一个关系数据库，因为它根本不支持在对象之间建立关系，但 “rrrdb” 这个名字确实来自于 Relational-database in Rust with RocksDB。Rrrdb 提供 SQL 接口与它的底层存储层 RocksDB 进行交互。Rrrdb 只是提供了 SQL 接口来与它的底层存储层(RocksDB)进行交互。本文描述了 rrdb 的样式和架构。

GitHub repository: [petitviolet/rrrdb](https://github.com/petitviolet/rrrdb)

# Overview

从README中复制而来。请忽略`.unwrap()` 。

```c
let rrrdb = RrrDB::new(path)
rrrdb.execute("test_db", "CREATE TABLE users (id integer, name varchar)").unwrap();
rrrdb.execute("test_db", "INSERT INTO users VALUES (1, 'Alice')").unwrap();
rrrdb.execute("test_db", "INSERT INTO users VALUES (2, 'Bob')").unwrap();
let result = rrrdb.execute("test_db", "SELECT name FROM users WHERE id = 2").unwrap();
result == 
    OkDBResult::SelectResult(ResultSet::new(
        vec![
            Record::new(vec![
                FieldValue::Text("Bob".to_string()),
            ]),
        ],
        ResultMetadata::new(vec![
            FieldMetadata::new("name", "varchar")
        ])
    ))
```

我相信它是有一定作用的，因为我们可以看到 `RrrDB` 实例执行的 SQL语句。

# 架构

为了能够处理 SQL语句与 RocksDB 的交互，Rrrdb 有三个组件: SQL、 Planner 和 Executor。

## SQL

首先，它必须能够处理客户机发送的 SQL 语句字符串。为此，SQL 层由 分词器（tokenizer）和解析器（parser）组成。分词器将给定的 SQL 字符串标记为一组标记，然后解析器根据这组标记构建抽象语法树（AST）。结果就是，我们获得了抽象语法树，所以我们可以为如何从底层存储层获取实际数据创建一个计划。

## Planner 计划生成器

第二层是 Planner。Planner 希望通过 SQL 层生成的抽象语法树来构建一个从 KV存储的 RocksDB 获取实际数据的计划。例如，当我们得到一个SQL`select * from users where id = 1` 时，Planner 会生成一个计划来获取 `id = 1`的单个记录。再举一个例子，当我们得到一个 SQL `select * from users` ，用于获取用户表中的所有记录时，Planner 将生成一个迭代所有 `user` 表的键值对的计划。很明显，Planner 是有很大空间优化这个数据库系统性能的地方，但是，到目前为止，我没有做任何优化:)

## Executor 计划执行器

Executor 是最后一个，但或许也是最重要的一个。

Executor 反馈计划并访问底层存储层，以获取客户想要的数据。存储层以数组字节的形式处理数据，即 Rust 中的 `[u8]` ，这样 RocksDB 就可以在不转换数据的情况下存储数据，因为 Rust-RocksDB 需要关键字和值的`[u8]`。Executor 在性能方面有许多需要改进的空间，例如数据格式、缓存、批量操作等。除了性能之外，我们还有很多实现从 SQL 语句到键值对操作转换的方法，尽管这取决于存储层的架构。如果我们希望使用面向列的数据库而不是面向行的数据库，那么计划应该逐一访问每个列的值，而不是一次获取所有列。

# 实现

## ColumnFamily

RocksDB 提供 ColumnFamily 特性以支持逻辑分区数据库。ColumnFamily 有一组键值对。此外，我们可以在单个 ColumnFamily 上使用 Iterator，这样我们就可以在特定 ColumnFamily 中获取所有的记录，而无需访问不相关的数据。在 rrrdb 中，它使用 ColumnFamilies 来存储表的数据。这意味着通过在相应的 ColumnFamily 上运行迭代来获取特定表的所有记录。Rrdb 的 `Storage`模块 实现了[`iterator`](https://github.com/petitviolet/rrrdb/blob/a120db0e3346b76d561153bf00d0078148fc7df3/src/rrrdb/storage.rs#L111-L115)方法，以获得基础 ColumnFamily 上的迭代器。

```c
fn get_column_family(&self, namespace: &Namespace) -> DBResult<&ColumnFamily> {
    let cf_name = namespace.cf_name();
    self.rocksdb
        .cf_handle(&cf_name)
        .ok_or(DBError::namespace_not_found(namespace))
}

pub fn iterator<'a>(&'a self, namespace: &Namespace) -> DBResult<RecordIterator<'a>> {
    self.get_column_family(namespace).map(|cf| RecordIterator {
        db_iterator: self.rocksdb.iterator_cf(cf, rocksdb::IteratorMode::Start),
    })
}
```

如上面的代码片段所示，它调用 `RocksDB::iterator_cf` 来获得 ColumnFamily 上的迭代器，然后将其包装为名为 `RecordIterator` 的原始迭代器类型，这样它就可以在每个键值对上应用所需的内容。

```c
impl<'a> Iterator for RecordIterator<'a> {
    type Item = (String, Box<[u8]>);

    fn next(&mut self) -> Option<Self::Item> {
        let n = self.db_iterator.next();
        n.map(|(key, value)| {
            let string_key = String::from_utf8(key.to_vec())
                .expect(&format!("Invalid key was found. key: {:?}", key));
            (string_key, value)
        })
    }
}
```

在本例中，`RecordIterator::next` 基于所有键的表示都是字符串的假设，返回一对字符串化的键和一个原始值。

## 使用宏来定义支持关键字

我使用宏来定义 SQL 中支持的关键字列表，如下面的代码片段所示

```c
macro_rules! define_keywords {
  ($($keyword:ident), *) => {
      #[derive(Debug, Clone, PartialEq, Eq, Hash)]
      pub(crate) enum Keyword {
        $($keyword), *
      }
      impl Keyword {
        pub fn find(s: &str) -> Option<Keyword> {
          match s {
            $(s if s.to_lowercase() == stringify!($keyword).to_lowercase() => { Some(Keyword::$keyword) },)
            *
            _ => None,
          }
        }
      }
      impl std::fmt::Display for Keyword {
        fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
          match self {
            $(Keyword::$keyword => { write!(f, "{}", stringify!($keyword)) }),
            *
          }
        }
      }
      pub(crate) const ALL: &[Keyword] = &[
        $(Keyword::$keyword), *
      ];
  };
}

define_keywords!(Create, Database, Table, Select, From, Where, Insert, Into, Values);
```

用`Keyword::find`，很容易判断给定的字符串是否是关键字。

```c
match Keyword::find(s) {
    Some(keyword) => return_ok(Token::Keyword(keyword)),
    None => return_ok(Token::Word(s)),
}
```

## 基于Vec的模式匹配

下面的代码片段是我可以在 `Vec<A>` 上分享的一个关于模式匹配的提示，它期望在给定的 `Vec<Token>` 中有2个 token，并将它们绑定到临时值中。

```c
let mut v = vec![];

// v.push(token);

if let &[Token::Word(column_name), Token::Word(column_type)] = &&v[..] {
    results.push(ColumnDefinition::new(
        column_name.to_owned(), // Is this to_owned() avoidable...?
        column_type.to_owned(),
    ));
} else {
    return Self::unexpected_token("create table column definitions", &v, self.pos);
}
```

## 序列化/反序列化

Rrdb 在 RocksDb 中以 `Vec<u8>` 的形式存储数据，但是这些数据基本上是 Struct 的实例。因此，最好能够直接获取/放置 struct 实例，而不需要通过 rrrdb 用户端将它们转换为 `Vec<u8>` 。为此，rrrdb 依赖于 [serde_json](https://docs.serde.rs/serde_json/) crate，它提供了在 Rust 中处理 JSON 的能力，因此它可以在给定的结构上自动应用序列化和反序列化。用于从 RocksDB 获取对象并进行序列化的代码片段;

```c
pub fn get_serialized<T: DeserializeOwned>(
    &self,
    namespace: &Namespace,
    key: &str,
) -> Result<Option<T>, DBError> {
    self.get(namespace, key).and_then(|opt| match opt {
        Some(found) => match String::from_utf8(found) {
            Ok(s) => match serde_json::from_str::<T>(&s) {
                Ok(t) => Ok(Some(t)),
                Err(err) => Err(DBError::from(format!("Failed to deserialize: {:?}", err))),
            },
            Err(err) => Err(DBError::from(format!("Failed to deserialize: {:?}", err))),
        },
        None => Ok(None),
    })
}
```

# 一些思考

构建一个关系型数据库之类的系统对我来说真的很有趣。尽管这个迷你数据库功能有限并且性能比较糟糕，但这是学习 Rust 和数据库的好方法。如果您想更深入地了解基于 Key-Value 数据库的 RDBMS，TiDB 可能是最佳选择。
