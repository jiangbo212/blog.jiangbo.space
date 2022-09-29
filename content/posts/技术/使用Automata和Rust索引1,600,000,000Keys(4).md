---
title: "使用Automata和Rust索引1,600,000,000Keys(4)"
date: 2022-09-29T16:43:25+08:00
draft: false
categories: 技术
keywords: Rust Automata fst RocksDB 状态机 有限状态机 倒排索引 Lucene 有序集合 fsa
---

在前两节中，我一直小心避免谈论用于表示有序集或map的有限状态机的构造。也就是说，构造比简单的遍历要复杂一些。

In the previous two sections, I have been careful to avoid talking about the construction of finite state machines that are used to represent ordered sets or maps. Namely, construction is a bit more complex than simple traversal.

为了简单起见，我们对 set 或 map 中的元素进行了限制：它们必须按字典顺序添加。这是一个繁重的限制，但我们稍后会看到如何减轻它。

To keep things simple, we place a restriction on the elements in our set or map: they must be added in lexicographic order. This is an onerous restriction, but we will see later how to mitigate it.

为了激发有限状态机构造的灵感，让我们尝试谈谈。

To motivate construction of finite state machines, let’s talk about tries.

### trie构造 Trie construction

可以将 trie 视为确定性非循环有限状态接受器。因此，您在上一节中学到的关于有序集合的所有内容同样适用于它们。trie 和本文中显示的 FSA 之间的唯一区别是 trie 允许在密钥之间共享前缀，而 FSA 允许共享前缀和后缀。

A trie can be thought of as a deterministic acyclic finite state acceptor. Therefore, everything you learned in the previous section on ordered sets applies equally well to them. The only difference between a trie and the FSAs shown in this article is that a trie permits the sharing of prefixes between keys while an FSA permits the sharing of both prefixes and suffixes.

考虑一个带有key mon, tues和thurs的集合。以下是受益于共享前缀和后缀的FSA：

Consider a set with the keys mon, tues and thurs. Here is the corresponding FSA that benefits from sharing both prefixes and suffixes:

![](/img/days3.png)

这是对应的 trie，它只共享前缀：

And here is the corresponding trie, which only shares prefixes:

![](/img/days3-trie.png)

请注意，现在有三个不同的最终状态，并且keys tues和thurs需要复制s的最终转换到最终状态.

Notice that there are now three distinct final states, and the keys tues and thurs require duplicating the final transition for s to the final state.

构造一个 trie 相当简单。给定要插入的新key，只需执行正常查找即可。如果输入已用尽，则应将当前状态标记为最终状态。如果机器在输入耗尽之前停止，因为没有有效的转换可以遵循，那么只需为每个剩余的输入创建一个新的转换和节点。最后创建的节点应标记为最终节点。

Constructing a trie is reasonably straight-forward. Given a new key to insert, all one needs to do is perform a normal lookup. If the input is exhausted, then the current state should be marked as final. If the machine stops before the input is exhausted because there are no valid transitions to follow, then simply create a new transition and node for each remaining input. The last node created should be marked final.

### FSA构造 FSA construction

回想一下，trie 和 FSA 之间的唯一区别是 FSA 允许在key之间共享后缀。由于 trie 本身就是一个 FSA，我们可以构造一个 trie，然后应用一个通用的最小化算法，这将实现我们共享后缀的目标。

Recall that the only difference between a trie and an FSA is that an FSA permits the sharing of suffixes between keys. Since a trie is itself an FSA, we could construct a trie and then apply a general minimization algorithm, which would achieve our goal of sharing suffixes.

然而，一般的最小化算法在时间和空间上都可能很昂贵。例如，trie 通常比在key后缀之间共享结构的 FSA大得多。相反，如果我们可以假设键是按字典顺序添加的，我们可以做得更好。基本技巧是意识到在插入新key时，FSA 的任何不与新key共享前缀的部分都可以被冻结。也就是说，添加到 FSA 的任何新key都不可能使 FSA 的该部分更小。

However, general minimization algorithms can be expensive both in time and space. For example, a trie can often be much larger than an FSA that shares structure between suffixes of keys. Instead, if we can assume that keys are added in lexicographic order, we can do better. The essential trick is realizing that when inserting a new key, any parts of the FSA that don’t share a prefix with the new key can be frozen. Namely, no new key added to the FSA can possibly make that part of the FSA smaller.

一些图片可能有助于更好地解释这一点。再次考虑key mon, tues 和 thurs。由于我们必须按字典顺序添加它们，因此我们将mon先添加，然后再添加thurs和thes。这是添加第一个key后FSA的样子：

Some pictures might help explain this better. Consider again the keys mon, tues and thurs. Since we must add them in lexicographic order, we’ll add mon first, then thurs and then tues. Here’s what the FSA looks like after the first key has been added:

![](/img/days3-fsa-1.png)


这是不是很有趣。下面是我们插入thurs时发生的情况：

This isn’t so interesting. Here’s what happens when we insert thurs:

![](/img/days3-fsa-2.png)


插入thurs导致第一个key mon被冻结（由图像中的蓝色表示）。当 FSA 的特定部分被冻结时，我们就知道它将来永远不需要修改。即，由于所有将来添加的键都将是>= thurs，我们知道未来的键不会以mon开头。这很重要，因为它让我们可以重用自动机的那一部分，而不必担心它将来是否会改变。换句话说，蓝色的状态是其他key重用的候选状态。

The insertion of thurs caused the first key, mon, to be frozen (indicated by blue coloring in the image). When a particular part of the FSA has been frozen, then we know that it will never need to be modified in the future. Namely, since all future keys added will be >= thurs, we know that no future keys will start with mon. This is important because it lets us reuse that part of the automaton without worrying about whether it might change in the future. Stated differently, states that are colored blue are candidates for reuse by other keys.


虚线表示thurs尚未实际添加到 FSA。实际上，添加它需要检查是否存在任何可重用的状态。不幸的是，我们还不能这样做。例如，状态3和8是等价的：两者都是最终的，都没有任何转换。但是， 状态8永远等于状态3是不正确的。即，我们添加的下一个键可以是，例如 thursday。这会将更改状态8为具有d转换，这将使其不等于状态3。因此，我们还不能完全断定自动机中的key thurs是什么样子的。

The dotted lines represent that thurs hasn’t actually been added to the FSA yet. Indeed, adding it requires checking whether there exists any reusable states. Unfortunately, we can’t do that yet. For example, it is true that states 3 and 8 are equivalent: both are final and neither has any transitions. However, it is not true that state 8 will always be equal to state 3. Namely, the next key we add could, for example, be thursday. That would change state 8 to having a d transition, which would make it not equal to state 3. Therefore, we can’t quite conclude what the key thurs looks like in the automaton yet.

让我们继续插入下一个key tues：

Let’s move on to inserting the next key, tues:

![](/img/days3-fsa-3.png)

在添加tues的过程中，我们推断出key thurs一部分hurs 可以被冻结。为什么？因为keys是按字典顺序插入的，因此没有将来插入的key可能会最小化所采用路径hurs。例如，我们现在知道 key thursday不可能是集合的一部分，所以我们可以得出结论,thurs的最终状态thurs等价于mon的最终状态.它们都是最终的并且都没有转换，这将永远是正确的.

In the process of adding tues, we deduced that the hurs part of the thurs key could be frozen. Why? Because no future key inserted could possibly minimize the path taken by hurs since keys are inserted in lexicographic order. For example, we now know that the key thursday cannot ever be part of the set, so we can conclude that the final state of thurs is equivalent to the final state of mon: they are both final and both have no transitions, and this will forever be true.

请注意，状态4仍然是点状的：状态4可能会在随后的key插入时发生变化，因此我们还不能认为它等于任何其他状态。

Notice that state 4 remained dotted: it is possible that state 4 could change upon subsequent key insertions, so we cannot consider it equal to any other state just yet.

让我们再添加一个key来分析。考虑插入zon：

Let’s add one more key to drive the point home. Consider the insertion of zon:

![](/img/days3-fsa-4.png)

我们在这里看到状态4终于被冻结了，因为后面插入的zon不可能改变状态4。此外，我们还可以得出结论thurs和tues共享一个共同的后缀，并且确实，状态7和9（来自上图中）是等价的，因为它们都不是最终的，并且都具有指向相同状态的输入为s的单个转换。关键是它们的两个s转换都指向相同的状态，否则我们不能重用相同的结构。

We see here that state 4 has finally been frozen because no future insertion after zon can possibly change the state 4. Additionally, we could also conclude that thurs and tues share a common suffix, and that, indeed, states 7 and 9 (from the previous image) are equivalent because neither of them are final and both have a single transition with input s that points to the same state. It is critical that both of their s transitions point to the same state, otherwise we cannot reuse the same structure.

最后，我们必须表示我们已完成插入键。我们现在可以冻结FSA的最后一部分, zon，并寻找冗余结构：

Finally, we must signal that we are done inserting keys. We can now freeze the last portion of the FSA, zon, and look for redundant structure:

![](/img/days3-fsa-5.png)

当然，由于mon和zon共享一个共同的后缀，确实存在冗余结构。也就是说，前一个图像中的状态9在各个方面都等同于状态1。这是正确的，因为状态10和11也等价于状态2和3。如果这不是真的，那么我们就不能考虑状态9和1是一致的。例如，如果我们将key mom插入到我们的集合中，并且仍然假设状态9和1相等，那么生成的 FSA 将如下所示：

And of course, since mon and zon share a common suffix, there is indeed redundant structure. Namely, the state 9 in the previous image is equivalent in every way to state 1. This is only true because states 10 and 11 are also equivalent to states 2 and 3. If that weren’t true, then we couldn’t consider states 9 and 1 equal. For example, if we had inserted the key mom into our set and still assumed that states 9 and 1 were equal, then the resulting FSA would look something like this:

![](/img/days3-fsa-6.png)

这是错误的！为什么？因为这个 FSA 会声称key zom在集合中——但我们从未真正添加它。

And this would be wrong! Why? Because this FSA will claim that the key zom is in the set—but we never actually added it.

最后，值得注意的是，这里概述的构造算法可以以O(n)运行，其中n指的是key的数量。很容易看出，假设在每个状态中查找转换需要恒定的时间，那么在不检查冗余结构的情况下最初将键插入FST不会比遍历键中的每个字符花费更长的时间。更棘手的一点是：我们如何在恒定时间内找到冗余结构？简短的回答是一个哈希表，但我将在实践中的构造部分解释一些挑战。

Finally, it is worth noting that the construction algorithm outlined here can run in O(n) time where n is the number of keys. It is easy to see that inserting a key initially into the FST without checking for redundant structure does not take any longer than looping over each character in the key, assuming that looking up a transition in each state takes constant time. The trickier bit is: how do we find redundant structure in constant time? The short answer is a hash table, but I will explain some of the challenges with that in the section on construction in practice.
