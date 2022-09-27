---
title: "使用Automata和Rust索引1,600,000,000Keys(2)"
date: 2022-09-27T11:15:35+08:00
draft: false
categories: 技术
keywords: Rust Automata fst RocksDB 状态机 有限状态机 倒排索引 Lucene 有序集合
---

[原文链接](https://blog.burntsushi.net/transducers/)

### 目录 (Table of Contents)

这篇文章很长，所以我整理了一个目录, 如果你想跳过某些内容。

This article is pretty long, so I’ve put together a table of contents in case you want to skip around.

第一部分讨论了有限状态机及其作为数据结构的抽象用途。本节旨在为您提供一个推理数据结构的思维模型。本节没有代码。

The first section discusses finite state machines and their use as data structures in the abstract. This section is meant to give you a mental model with which to reason about the data structure. There is no code in this section.

第二部分采用第一部分中开发的抽象，并用一个实现来演示它。本节主要是概述如何使用我的fst 库。本节包含代码。我们将讨论一些实现细节，但会避免过于混乱。如果您不关心代码而只想查看真实数据的实验，则可以跳过此部分。

The second section takes the abstraction developed in the first section and demonstrates it with an implementation. This section is mostly intended to be an overview of how to use my fst library. This section contains code. We will discuss some implementation details, but will avoid the weeds. It is okay to skip this section if you don’t care about the code and instead only want to see experiments on real data.

第三部分也是最后一部分演示了使用简单的命令行工具来构建索引。我们将查看一些真实的数据集，并尝试将有限状态机的性能作为数据结构进行推理。

The third and final section demonstrates use of a simple command line tool to build indexes. We will look at some real data sets and attempt to reason about the performance of finite state machines as a data structure.

### 作为数据结构的有限状态机

### Finite state machines as data structures

有限状态机 (FSM) 是状态的集合和从一个状态移动到下一个状态的转换的集合。一个状态被标记为开始状态，零个或多个状态被标记为最终状态。FSM 一次总是处于一种状态。

A finite state machine (FSM) is a collection of states and a collection of transitions that move from one state to the next. One state is marked as the start state and zero or more states are marked as final states. An FSM is always in exactly one state at a time.

FSM 相当通用，可用于对许多过程进行建模。例如，考虑一下我的猫 Cauchy 的日常生活：

FSM’s are rather general and can be used to model a number of processes. For example, consider an approximation of the daily life of my cat Cauchy:

![cauchy](/img/cauchy.png)

一些状态是“睡着了”或“正在吃东西”，而一些转变是“食物已送达”或“有东西移动了”。这里没有任何最终状态，因为那将是不必要的病态！

Some states are “asleep” or “eating” and some transitions are “food is served” or “something moved.” There aren’t any final states here because that would be unnecessarily morbid!

请注意，FSM 近似于我们对现实的概念。Cauchy不能同时在玩耍和睡觉，所以它满足了我们的条件，即机器一次只处于一种状态。此外，请注意，从一种状态转换到下一种状态只需要来自环境的单个输入。也就是说，“睡着”不记得是因为玩累了还是饭后满足了。不管Cauchy是怎么睡着的，如果他听到有什么动静，或者晚餐的铃声响起，他总会醒来。

Notice that the FSM approximates our notion of reality. Cauchy cannot be both playing and asleep at the same time, so it satisfies our condition that the machine is only ever in one state at a time. Also, notice that transitioning from one state to the next only requires a single input from the environment. Namely, being “asleep” carries no memory of whether it was caused by getting tired from playing or from being satisfied after a meal. Regardless of how Cauchy fell asleep, he will always wake up if he hears something moving or if the dinner bell rings.

Cauchy的有限状态机可以在给定输入序列的情况下执行计算。例如，考虑以下输入：

Cauchy’s finite state machine can perform computation given a sequence of inputs. For example, consider the following inputs:

+ food is served 提供食物
+ loud noise 巨响
+ quiet calm 安静平静
+ food digests 食物消化

如果我们将这些输入应用到上面的机器上，那么 Cauchy 将依次经历以下状态：“睡着”、“吃”、“隐藏”、“吃”、“猫砂箱”。因此，如果我们观察到食物送达，然后是一声巨响，然后是安静的平静，最后是Cauchy的消化，那么我们可以得出结论，Cauchy目前在猫砂箱里。

If we apply these inputs to the machine above, then Cauchy will move through the following states in order: “asleep,” “eating,” “hiding,” “eating,” “litter box.” Therefore, if we observed that food was served, followed by a loud noise, followed by quiet calm and finally by Cauchy’s digestion, then we could conclude that Cauchy was currently in the litter box.

这个特别愚蠢的例子展示了有限状态机的真实性。出于我们的目的，我们需要对用于实现有序集和映射数据结构的有限状态机的类型进行一些限制。

This particularly silly example demonstrates how general finite state machines truly are. For our purposes, we will need to place a few restrictions on the type of finite state machine we use to implement our ordered set and map data structures.

### 有序集合 Ordered sets

有序集合与普通集合类似，只是集合中的键是有序的。也就是说，有序集提供了对其键的有序迭代。通常，有序集用二叉搜索树或btree实现，无序集用哈希表实现。在我们的例子中，我们将看一个使用确定性非循环有限状态接受器（缩写为 FSA）的实现。

An ordered set is like a normal set, except the keys in the set are ordered. That is, an ordered set provides ordered iteration over its keys. Typically, an ordered set is implemented with a binary search tree or a btree, and an unordered set is implemented with a hash table. In our case, we will look at an implementation that uses a deterministic acyclic finite state acceptor (abbreviated FSA).

确定性非循环有限状态接受器是一个有限状态机，它是：

A deterministic acyclic finite state acceptor is a finite state machine that is:

1. 确定性。这意味着在任何给定状态下，对于任何输入，最多可以遍历一个转换。

    Deterministic. This means that at any given state, there is at most one transition that can be traversed for any input.

2. 无环。这意味着不可能访问已经访问过的状态。

    Acyclic. This means that it is impossible to visit a state that has already been visited.

3. 一个接受者。这意味着有限状态机“接受”特定的输入序列当且仅当它在输入序列的末尾处于“最终”状态。（与前两个不同，这个标准将在我们在下一节中查看有序映射时发生变化。）

    An acceptor. This means that the finite state machine “accepts” a particular sequence of inputs if and only if it is in a “final” state at the end of the sequence of inputs. (This criterion, unlike the former two, will change when we look at ordered maps in the next section.)

我们如何使用这些属性来表示一个集合？诀窍是将集合的键存储在机器的转换中。这样，给定一系列输入（即字符），我们可以根据评估 FSA 是否以最终状态结束来判断key是否在集合中。

How can we use these properties to represent a set? The trick is to store the keys of the set in the transitions of the machine. This way, given a sequence of inputs (i.e., characters), we can tell whether the key is in the set based on whether evaluating the FSA ends in a final state.

考虑一个带有一个键“jul”的集合。FSA 看起来像这样：

Consider a set with one key “jul.” The FSA looks like this:

![](/img/set1.png)

考虑一下如果我们询问 FSA 是否包含key“jul”会发生什么。我们需要按顺序处理字符：

Consider what happens if we ask the FSA if it contains the key “jul.” We need to process the characters in order:


+ 给定j，FSA 从开始状态移动0到1. Given j, the FSA moves from the start state 0 to 1.
+ 给定u，FSA 从 移动1到2.  Given u, the FSA moves from 1 to 2.
+ 给定l，FSA 从 移动2到3。Given l, the FSA moves from 2 to 3.

由于密钥的所有成员都已被提供给 FSA，我们现在可以问：FSA 是否处于最终状态？它是（注意 state 周围的双圈3），所以我们可以说它jul在集合中。

Since all members of the key have been fed to the FSA, we can now ask: is the FSA in a final state? It is (notice the double circle around state 3), so we can say that jul is in the set.

考虑当我们测试一个不在集合中的键时会发生什么。例如 jun：

Consider what happens when we test a key that is not in the set. For example, jun:

+ 给定j，FSA 从开始状态移动0到1 Given j, the FSA moves from the start state 0 to 1.
+ 给定u，FSA 从 移动1到2.  Given u, the FSA moves from 1 to 2.
+ 给定n，FSA 不能移动。处理停止。 Given n, the FSA cannot move. Processing stops.

FSA 不能移动，因为唯一的状态转换2是l，但当前输入是n。由于l != n，FSA 无法遵循该转换。一旦给定输入，FSA 不能移动，它就可以断定key不在集合中。无需进一步处理输入。

The FSA cannot move because the only transition out of state 2 is l, but the current input is n. Since l != n, the FSA cannot follow that transition. As soon as the FSA cannot move given an input, it can conclude that the key is not in the set. There’s no need to process the input further.

考虑另一个键ju：

Consider another key, ju:

+ 给定j，FSA 从开始状态移动0到1 Given j, the FSA moves from the start state 0 to 1.
+ 给定u，FSA 从 移动1到2。 Given u, the FSA moves from 1 to 2.

在这种情况下，整个输入被耗尽，FSA 处于状态2。要确定是否ju在集合中，它必须询问是否2是最终状态。既然不是，它可以报告ju key不在集合中。

In this case, the entire input is exhausted and the FSA is in state 2. To determine whether ju is in the set, it must ask whether 2 is a final state or not. Since it is not, it can report that the ju key is not in the set.

这里值得指出的是，确认某个键是否在集合中所需的步骤数受该键中字符数的限制！也就是说，查找密钥所需的时间与集合的大小完全无关。

It is worth pointing out here that the number of steps required to confirm whether a key is in the set or not is bounded by the number of characters in the key! That is, the time it takes to lookup a key is not related at all to the size of the set.

让我们在集合中添加另一个键，看看它是什么样子的。以下 FSA 表示具有key “jul”和“mar”的有序集：

Let’s add another key to the set to see what it looks like. The following FSA represents an ordered set with keys “jul” and “mar”:

![](/img/set2.png)

FSA 变得更加复杂。开始状态0现在有两个转换：j和m。因此，给定 key mar，它将首先跟随m转换。

The FSA has grown a little more complex. The start state 0 now has two transitions: j and m. Therefore, given the key mar, it will first follow the m transition.

这里还有另一件重要的事情需要注意：状态3在jul和mar之间共享。即状态3有两个转换进入它. l和r。key之间的这种状态共享非常重要，因为它使我们能够在更小的空间中存储更多信息。

There’s one other important thing to notice here: the state 3 is shared between the jul and mar keys. Namely, the state 3 has two transitions entering it: l and r. This sharing of states between keys is really important, because it enables us to store more information in a smaller space.

让我们看看当我们添加jun到我们的集合时会发生什么，它与jul共享一个公共前缀：

Let’s see what happens when we add jun to our set, which shares a common prefix with jul:

![](/img/set3.png)

你看得到差别吗？这是一个小小的改变。这个 FSA 看起来很像前一个。只有一个区别：添加了n从状态5到状态3的转换。值得注意的是，FSA 没有新的状态！由于jun和jul共享前缀ju，这些状态可以被两个键重用。

Do you see the difference? It’s a small change. This FSA looks very much like the previous one. There’s only one difference: a new transition, n, from states 5 to 3 has been added. Notably, the FSA has no new states! Since both jun and jul share the prefix ju, those states can be reused for both keys.

让我们稍微改变一下，看看带有以下键的集合： october，november和december：

Let’s switch things up a little bit and look at a set with the following keys: october, november and december:

![](/img/set3-suffixes.png)

由于所有三个密钥都共享共同的后缀ber，因此仅将其编码到 FSA 中一次。其中两个密钥共享一个更大的后缀：ember，它也只被编码到 FSA 中一次。

Since all three keys share the suffix ber in common, it is only encoded into the FSA exactly once. Two of the keys share an even bigger suffix: ember, which is also encoded into the FSA exactly once.

在继续讨论有序map之前，我们应该花点时间说服自己这确实是一个有序集合。也就是说，给定一个 FSA，我们如何迭代集合中的键？

Before moving on to ordered maps, we should take a moment and convince ourselves that this is indeed an ordered set. Namely, given an FSA, how can we iterate over the keys in the set?

为了证明这一点，让我们使用我们之前构建的一组键jul， jun和mar：

To demonstrate this, let’s use a set we built earlier with the keys jul, jun and mar:

![](/img/set3.png)

我们可以通过按照字典顺序遍历整个 FSA 来枚举集合中的所有键。例如：

We can enumerate all keys in the set by walking the entire FSA by following transitions in lexicographic order. For example:

+ 初始化状态0。key是空的。 Initialize at state 0. key is empty.
+ 移动到状态4。添加j到key. Move to state 4. Add j to key.
+ 移动到状态5。添加u到key. Move to state 5. Add u to key.
+ 移动到状态3。添加l到key. 组成 jul。Move to state 3. Add l to key. Emit jul.
+ 移回状态5。从key中删除l. Move back to state 5. Drop l from key.
+ 移动到状态3。添加n到key. 组成 jun。 Move to state 3. Add n to key. Emit jun.
+ 移回状态5。从key中删除n. Move back to state 5. Drop n from key.
+ 移回状态4。从key中删除u. Move back to state 4. Drop u from key.
+ 移回状态0。从key中删除j. Move back to state 0. Drop j from key.
+ 移动到状态1。添加m到key. Move to state 1. Add m to key.
+ 移动到状态2。添加a到key. Move to state 2. Add a to key.
+ 移动到状态3。添加r到key. 组成 mar。Move to state 3. Add r to key. Emit mar.

这个算法很容易用一堆要访问的状态和一堆已经遵循的转换来实现。它的时间复杂度是O(n)，n为集合中元素的数量。空间复杂度是O(k), k是集合中最大key的大小。

This algorithm is straight-forward to implement with a stack of the states to visit and a stack of transitions that have been followed. It has time complexity O(n) in the number of keys in the set with space complexity O(k) where k is the size of the largest key in the set.
