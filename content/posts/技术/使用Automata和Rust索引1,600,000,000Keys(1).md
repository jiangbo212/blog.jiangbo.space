---
title: "使用Automata和Rust索引1,600,000,000Keys(1)"
date: 2022-09-26T20:55:30+08:00
draft: false
categories: 技术
keywords: Rust Automata fst RocksDB 状态机 有限状态机 倒排索引 Lucene
---
[原文链接](https://blog.burntsushi.net/transducers/)

事实证明，有限状态机对表达计算以外的其他事情很有用。有限状态机也可以用来紧凑地表示可以快速搜索的有序集合(Maps)或字符串映射(Sets)。

It turns out that finite state machines are useful for things other than expressing computation. Finite state machines can also be used to compactly represent ordered sets or maps of strings that can be searched very quickly.

在本文中，我将向您介绍有限状态机作为表示有序集合和映射的数据结构。这包括引入一个用 Rust 编写的实现，称为 fst crate。它带有完整的 API 文档。我还将向您展示如何使用简单的命令行工具构建它们。最后，我将讨论一些实验，最终从 2015 年 7 月的 Common Crawl Archive 中索引超过 1,600,000,000 个 URL（134 GB）

In this article, I will teach you about finite state machines as a data structure for representing ordered sets and maps.  This includes introducing an implementation written in Rust called the fst crate. It comes with complete API documentation.I will also show you how to build them using a simple command line tool. Finally, I will discuss a few experiments culminating in indexing over 1,600,000,000 URLs (134 GB) from the July 2015 Common Crawl Archive.

本文介绍的技术也是 Lucene 如何表示其倒排索引的一部分。

The technique presented in this article is also how Lucene represents a part of its inverted index.

在此过程中，我们将讨论内存映射、与正则表达式的自动机交集、与 Levenshtein 距离的模糊搜索和流式集合操作。

Along the way, we will talk about memory maps, automaton intersection with regular expressions, fuzzy searching with Levenshtein distance and streaming set operations.

目标受众：对编程和基本数据结构有一定了解。不需要自动机理论或 Rust 的经验。

Target audience: Some familiarity with programming and fundamental data structures. No experience with automata theory or Rust is required.

前序(Teaser)

作为展示我们前进方向的前序，让我们快速浏览一个示例。我们还不会看到 1,600,000,000 个字符串。相反，请考虑大约 16,000,000 条维基百科文章标题 (384 MB)。以下是索引它们的方法:

As a teaser to show where we’re headed, let’s take a quick look at an example. We won’t look at 1,600,000,000 strings quite yet. Instead, consider ~16,000,000 Wikipedia article titles (384 MB). Here’s how to index them:

```
$ time fst set --sorted wiki-titles wiki-titles.fst

real    0m18.310
```

生成的索引 wiki-titles.fst 为 157 MB。相比之下，gzip 需要 12 秒并压缩到 91 MB。 （对于某些数据集，我们的索引方案在速度和压缩率上都可以击败 gzip。）

The resulting index, wiki-titles.fst, is 157 MB. By comparison, gzip takes 12 seconds and compresses to 91 MB. (For some data sets, our indexing scheme can beat gzip in both speed and compression ratio.)

但是，这里有 gzip 不能做的事情：快速找到所有以 Homer 开头的文章标题：

However, here’s something gzip cannot do: quickly find all article titles starting with Homer the:

```
$ time fst grep wiki-titles.fst 'Homer the.*'
Homer the Clown
Homer the Father
Homer the Great
Homer the Happy Ghost
Homer the Heretic
Homer the Moe
Homer the Smithers
...

real    0m0.023s
```

相比之下，grep 处理原始未压缩数据需要 0.3 秒。

By comparison, grep takes 0.3 seconds on the original uncompressed data.

最后，对于连 grep 都做不到的事情：快速找到 Homer Simpson 一定编辑距离内的所有文章标题：

And finally, for something that even grep cannot do: quickly find all article titles within a certain edit distance of Homer Simpson:

```
$ time fst fuzzy wiki-titles.fst --distance 2 'Homer Simpson'
Home Simpson
Homer J Simpson
Homer Simpson
Homer Simpsons
Homer simpson
Homer simpsons
Hope Simpson
Roger Simpson

real    0m0.094s
```
这篇文章很长，所以如果你只是为了粉丝来的，那么你可以直接跳到我们索引 1,600,000,000 个键的部分。

This article is quite long, so if you only came for the fan fare, then you may skip straight to the section where we index 1,600,000,000 keys.