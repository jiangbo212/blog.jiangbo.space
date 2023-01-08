---
title: "dyn Trait的性能影响及其相应的性能提升方案"
date: 2023-01-08T22:11:30+08:00
draft: true
categories: rust
keywords: rust "dyn trait" enum_dispatch "dynamic dispatch"
---
本篇文章受meilisearch的issue [#2456](https://github.com/meilisearch/meilisearch/issues/2456) 启发。并主要参考以下两篇英文文章 
 + [a basic explanation of how dyn Traits work internally ](https://doc.rust-lang.org/std/keyword.dyn.html)
 + [a basice explanation of enum_dispath](https://docs.rs/enum_dispatch/latest/enum_dispatch) 。

