---
title: "Rust中有限状态转换器(FST)的实现fst Crate"
date: 2022-09-29T11:16:05+08:00
draft: false
categories: 技术
keywords: Rust Automata fst RocksDB 状态机 有限状态机 倒排索引 Lucene 有序集合 fsa crate
---

fst库是"使用Automata和Rust索引1,600,000,000Keys"作者所建立的实现其博客思想的库。作者文章的后续就是针对其博客思想的构建历程，我们在这里先插入一下作者实现的这个库的一些主要功能，方便理解作者后续的思想。[原文地址](https://docs.rs/fst/latest/fst/index.html)

fst是一个用于有效存储和搜索有序集合或者key是字符串的map的库。这个crate的一个关键设计目标是支持存储和搜索非常大的集合或map（即数十亿）。这意味着已经付出了很多努力来确保所有操作都是内存高效的。

fst is a library for efficiently storing and searching ordered sets or maps where the keys are byte strings. A key design goal of this crate is to support storing and searching very large sets or maps (i.e., billions). This means that much effort has gone in to making sure that all operations are memory efficient.

集合和映射由有限状态机(FSA)表示，它是一种对key中公共前缀和后缀的压缩形式。此外，有限状态机可以有效的对自动机（如正则表达式或模糊查询的Levenshtein距离）或字典范围进行查询。

Sets and maps are represented by a finite state machine, which acts as a form of compression on common prefixes and suffixes in the keys. Additionally, finite state machines can be efficiently queried with automata (like regular expressions or Levenshtein distance for fuzzy queries) or lexicographic ranges.

要阅读更多有关有限状态转换器机制的信息，包括此crate中使用的算法的参考书目，请参阅该 raw::Fst的文档。

To read more about the mechanics of finite state transducers, including a bibliography for algorithms used in this crate, see the docs for the raw::Fst type.

### 安装

只需将fst添加到您的Cargo.toml依赖项列表中：

Simply add a corresponding entry to your Cargo.toml dependency list:

```
[dependencies]
fst = "0.4"
```

本文档中的示例将显示其余部分。

The examples in this documentation will show the rest.

### 类型和模块概述 Overview of types and modules

这个 crate 在顶层模块中提供了高级抽象的集合和map。

This crate provides the high level abstractions—namely sets and maps—in the top-level module.

set和map子模块包含特定于集合和map的类型，例如范围查询和流。

The set and map sub-modules contain types specific to sets and maps, such as range queries and streams.

raw模块允许与有限状态转换器直接交互。即，可以使用raw模块直接访问转换器的状态和转换。

The raw module permits direct interaction with finite state transducers. Namely, the states and transitions of a transducer can be directly accessed with the raw module.


### 示例：模糊查询 Example: fuzzy query

这个例子展示了如何在内存中创建一组字符串，然后执行一个模糊查询。即，查找与foo编辑距离为1的所有key。（编辑距离是从一个字符串到另一个字符串所需的字符插入、删除或替换的数量。在这种情况下，一个字符是一个 Unicode 代码点。注：如果对这块有疑问，可直接百度Levenshtein距离）

This example shows how to create a set of strings in memory, and then execute a fuzzy query. Namely, the query looks for all keys within an edit distance of 1 of foo. (Edit distance is the number of character insertions, deletions or substitutions required to get from one string to another. In this case, a character is a Unicode codepoint.)

这需要在crate中启用levenshtein功能。默认情况下未启用。

This requires the levenshtein feature in this crate to be enabled. It is not enabled by default.

```
use fst::{IntoStreamer, Streamer, Set};
use fst::automaton::Levenshtein;

fn example() -> Result<(), Box<dyn std::error::Error>> {
    // A convenient way to create sets in memory.
    let keys = vec!["fa", "fo", "fob", "focus", "foo", "food", "foul"];
    let set = Set::from_iter(keys)?;

    // Build our fuzzy query.
    let lev = Levenshtein::new("foo", 1)?;

    // Apply our fuzzy query to the set we built.
    let mut stream = set.search(lev).into_stream();

    let keys = stream.into_strs()?;
    assert_eq!(keys, vec!["fo", "fob", "foo", "food"]);
    Ok(())
}

```

警告：Levenshtein自动机使用大量内存

Warning: Levenshtein automatons use a lot of memory

Levenshtein自动机的构造应考虑“概念证明”量。即他们做得恰到好处,同时他们没有任何内存浪费。

The construction of Levenshtein automatons should be consider “proof of concept” quality. Namely, they do just enough to be correct. But they haven’t had any effort put into them to be memory conscious.

请注意，如果 Levenshtein自动机变得太大将返回错误（堆使用量为数十 MB）。

Note that an error will be returned if a Levenshtein automaton gets too big (tens of MB in heap usage).

### 示例：使用文件流和内存映射以进行搜索 Example: stream to a file and memory map it for searching

这显示了如何创建一个MapBuilder将将文件构成成一个流式map。值得注意的是，这永远不会将整个转换器存储在内存中。相反，在构造过程中只需要固定大小的内存。

This shows how to create a MapBuilder that will stream construction of the map to a file. Notably, this will never store the entire transducer in memory. Instead, only constant memory is required during construction.

在搜索阶段，我们使用 memmap crate 使文件可以&[u8]形式使用,而不必将其全部读入内存（操作系统会自动为您处理）。注：内存映射这块可参考memmp crate

For the search phase, we use the memmap crate to make the file available as a &[u8] without necessarily reading it all into memory (the operating system will automatically handle that for you).

```
use std::fs::File;
use std::io;

use fst::{IntoStreamer, Streamer, Map, MapBuilder};
use memmap::Mmap;

// This is where we'll write our map to.
let mut wtr = io::BufWriter::new(File::create("map.fst")?);

// Create a builder that can be used to insert new key-value pairs.
let mut build = MapBuilder::new(wtr)?;
build.insert("bruce", 1).unwrap();
build.insert("clarence", 2).unwrap();
build.insert("stevie", 3).unwrap();

// Finish construction of the map and flush its contents to disk.
build.finish()?;

// At this point, the map has been constructed. Now we'd like to search it.
// This creates a memory map, which enables searching the map without loading
// all of it into memory.
let mmap = unsafe { Mmap::map(&File::open("map.fst")?)? };
let map = Map::new(mmap)?;

// Query for keys that are greater than or equal to clarence.
let mut stream = map.range().ge("clarence").into_stream();

let kvs = stream.into_str_vec()?;
assert_eq!(kvs, vec![
    ("clarence".to_owned(), 2),
    ("stevie".to_owned(), 3),
]);
```

### 示例：不区分大小写的搜索 Example: case insensitive search

我们可以使用正则表达式对集合执行不区分大小写的搜索。我们可以使用regex-automata crate将正则表达式编译成自动机：

We can perform case insensitive search on a set using a regular expression. We can use the regex-automata crate to compile a regular expression into an automaton:

```
use fst::{IntoStreamer, Set};
use regex_automata::dense; // regex-automata crate with 'transducer' feature

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let set = Set::from_iter(&["FoO", "Foo", "fOO", "foo"])?;
    let pattern = r"(?i)foo";
    // Setting 'anchored' is important, otherwise the regex can match anywhere
    // in the key. This would cause the regex to iterate over every key in the
    // FST set.
    let dfa = dense::Builder::new().anchored(true).build(pattern).unwrap();

    let keys = set.search(&dfa).into_stream().into_strs()?;
    assert_eq!(keys, vec!["FoO", "Foo", "fOO", "foo"]);
    println!("{:?}", keys);
    Ok(())
}
```

请注意，要使其正常工作，regex-automata crate 必须在transducer下启用编译：

Note that for this to work, the regex-automata crate must be compiled with the transducer feature enabled:

```
[dependencies]
fst = "0.4"
regex-automata = { version = "0.1.9", features = ["transducer"] }
```

### 示例：有效地搜索多个集合 Example: searching multiple sets efficiently

由于查询可以在不将整个数据结构读入内存的情况下搜索一个转换器，因此可以非常快速地搜索多个转换器。

Since queries can search a transducer without reading the entire data structure into memory, it is possible to search many transducers very quickly.

这个 crate 提供了有效的 set/map 操作，允许组合多个搜索结果流。每个操作只使用与流数量成比例的内存。

This crate provides efficient set/map operations that allow one to combine multiple streams of search results. Each operation only uses memory proportional to the number of streams.

下面的示例显示了如何查找所有以B or G开头的key。下面的示例使用集合，但map上也可以使用相同的操作。

The example below shows how to find all keys that start with B or G. The example below uses sets, but the same operations are available on maps too.

```
use fst::automaton::{Automaton, Str};
use fst::set;
use fst::{IntoStreamer, Set, Streamer};

fn example() -> Result<(), Box<dyn std::error::Error>> {
    let set1 = Set::from_iter(&["AC/DC", "Aerosmith"])?;
    let set2 = Set::from_iter(&["Bob Seger", "Bruce Springsteen"])?;
    let set3 = Set::from_iter(&["George Thorogood", "Golden Earring"])?;
    let set4 = Set::from_iter(&["Kansas"])?;
    let set5 = Set::from_iter(&["Metallica"])?;

    // Create the matcher. We can reuse it to search all of the sets.
    let matcher = Str::new("B")
        .starts_with()
        .union(Str::new("G").starts_with());

    // Build a set operation. All we need to do is add a search result stream
    // for each set and ask for the union. (Other operations, like intersection
    // and difference are also available.)
    let mut stream =
        set::OpBuilder::new()
        .add(set1.search(&matcher))
        .add(set2.search(&matcher))
        .add(set3.search(&matcher))
        .add(set4.search(&matcher))
        .add(set5.search(&matcher))
        .union();

    // Now collect all of the keys. Alternatively, you could build another set
    // here using `SetBuilder::extend_stream`.
    let mut keys = vec![];
    while let Some(key) = stream.next() {
        keys.push(String::from_utf8(key.to_vec())?);
    }
    assert_eq!(keys, vec![
        "Bob Seger",
        "Bruce Springsteen",
        "George Thorogood",
        "Golden Earring",
    ]);
    Ok(())
}
```

### 内存使用情况

使用有限状态转换器来表示集合和映射的一个重要优点是它们可以根据key的分布很好地压缩。您的集合/map越小，它就越有可能适合内存。如果它在内存中，那么搜索它会更快。因此，重要的是要尽我们所能来限制内存中实际需要的内容。

An important advantage of using finite state transducers to represent sets and maps is that they can compress very well depending on the distribution of keys. The smaller your set/map is, the more likely it is that it will fit into memory. If it’s in memory, then searching it is faster. Therefore, it is important to do what we can to limit what actually needs to be in memory.

这就是自动机大放异彩的地方，因为它们可以在压缩状态下进行查询，而无需将整个数据结构加载到内存中。这意味着可以将由这个crate创建的集合/map存储在磁盘上并搜索它，而无需实际将整个集合/map读入内存。内存映射很好地服务于这个用例，它允许将文件的全部内容分配到虚拟内存的连续区域。

This is where automata shine, because they can be queried in their compressed state without loading the entire data structure into memory. This means that one can store a set/map created by this crate on disk and search it without actually reading the entire set/map into memory. This use case is served well by memory maps, which lets one assign the entire contents of a file to a contiguous region of virtual memory.

事实上，这个crate鼓励这种操作模式。集合和map都可以由任何提供AsRef<[u8]>实现的东西构成，任何内存映射都应该这样做。

Indeed, this crate encourages this mode of operation. Both sets and maps can be constructed from anything that provides an AsRef<[u8]> implementation, which any memory map should.

这对于使用此crate的长时间运行的进程尤其重要，因为它使操作系统能够确定有限状态转换器的哪些区域实际上在内存中。

This is particularly important for long running processes that use this crate, since it enables the operating system to determine which regions of your finite state transducers are actually in memory.

当然，这种方法也有缺点。即，在key查找或搜索期间转换器可能会遵循近似随机访问的模式。磁盘读取时随机访问可能会非常慢，因为seek必须经常调用（或者，在内存映射的情况下，页面错误）。消除寻道时间的固态驱动器的普及在一定程度上缓解了这种情况。尽管如此，固态驱动器并不是无处不在，而且操作系统可能不够智能，无法将内存映射的转换器保存在页面缓存中。在这种情况下，建议将整个转换器加载到进程的内存中（例如，使用Set::new和Vec<u8>）。

Of course, there are downsides to this approach. Namely, navigating a transducer during a key lookup or a search will likely follow a pattern approximating random access. Supporting random access when reading from disk can be very slow because of how often seek must be called (or, in the case of memory maps, page faults). This is somewhat mitigated by the prevalence of solid state drives where seek time is eliminated. Nevertheless, solid state drives are not ubiquitous and it is possible that the OS will not be smart enough to keep your memory mapped transducers in the page cache. In that case, it is advisable to load the entire transducer into your process’s memory (e.g., calling Set::new with a Vec<u8>).

### 流 Streams

搜索集合或map需要提供某种方式来迭代搜索结果。惯用的Rust 要求在Iterator这里使用满足特征的东西。不幸的是，这不可能有效地完成。因为该Iterator特征不允许迭代器发出的值从迭代器借用。在我们的例子中需要从迭代器中借用，因为keys和values是在迭代期间构造的。

Searching a set or a map needs to provide some way to iterate over the search results. Idiomatic Rust calls for something satisfying the Iterator trait to be used here. Unfortunately, this is not possible to do efficiently because the Iterator trait does not permit values emitted by the iterator to borrow from the iterator. Borrowing from the iterator is required in our case because keys and values are constructed during iteration.

也就是说，如果我们要使iterators，那么每个key都需要自己分配，这可能会非常昂贵。

Namely, if we were to use iterators, then every key would need its own allocation, which could be quite costly.

相反，这个 crate 提供了一个Streamer，它可以被认为是一个流iterator。也就是说，这个 crate 中的一个流维护一个单独的key缓冲区，并在每次迭代时将其借出。

Instead, this crate provides a Streamer, which can be thought of as a streaming iterator. Namely, a stream in this crate maintains a single key buffer and lends it out on each iteration.

有关更多详细信息，包括重要限制，请参阅Streamer trait。

For more details, including important limitations, see the Streamer trait.

### 特别之处 Quirks

毫无疑问，有限状态转换器是一种特殊的数据结构。它们有许多不适用于标准库中其他类似数据结构的限制，例如BTreeSet和 BTreeMap. 这里是其中的一些：

There’s no doubt about it, finite state transducers are a specialty data structure. They have a host of restrictions that don’t apply to other similar data structures found in the standard library, such as BTreeSet and BTreeMap. Here are some of them:

1. 集合只能包含字节字符串的keys。 
    Sets can only contain keys that are byte strings.
2. map也只能包含字节字符串的keys，其值仅限于u64位整数。（也许有一天会放宽对值的限制。） 
    Maps can also only contain keys that are byte strings, and its values are limited to unsigned 64 bit integers. (The restriction on values may be relaxed some day.)
3. 创建集合或map需要按字典顺序插入key。通常，key尚未排序，这会使构建大型集合或map变得棘手。一种方法是对数据片段进行排序并为每个片段构建一个集合/map。这可以简单地并行化。完成后，如果需要，它们可以合并到一个大集合/map中。fst-bin/src/merge.rs是从这个crate的存储库的根目录可以看到这个过程的一个有点简单的例子 。

    Creating a set or a map requires inserting keys in lexicographic order. Often, keys are not already sorted, which can make constructing large sets or maps tricky. One way to do it is to sort pieces of the data and build a set/map for each piece. This can be parallelized trivially. Once done, they can be merged together into one big set/map if desired. A somewhat simplistic example of this procedure can be seen in fst-bin/src/merge.rs from the root of this crate’s repository.