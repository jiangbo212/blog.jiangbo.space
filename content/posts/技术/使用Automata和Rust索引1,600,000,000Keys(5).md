---
title: "使用Automata和Rust索引1,600,000,000Keys(5)"
date: 2022-09-30T11:04:40+08:00
draft: false
categories: 技术
keywords: Rust Automata fst RocksDB 状态机 有限状态机 倒排索引 Lucene 有序集合 fsa
---

### FST构造

构造确定性非循环有限状态转换器的工作方式与构造确定性非循环有限状态接受器的工作方式大致相同。关键区别在于转换时输出的放置和共享。

Constructing deterministic acyclic finite state transducers works in much the same way as constructing deterministic acyclic finite state acceptors. The key difference is the placement and sharing of outputs on transitions.

为了降低心理负担，我们将重用上一节中的示例，其中包含key mon，tues和thurs。由于使用FST表示map，我们将一周中的数字日期2，3，5分别与每个key相关联。

To keep the mental burden low, we will reuse the example in the previous section with keys mon, tues and thurs. Since FSTs represent maps, we will associate the numeric day of the week with each key: 2, 3 and 5, respectively.

和以前一样，我们将从插入第一个key mon开始：

As before, we’ll start with inserting the first key, mon:

![](/img/days3-fst-1.png)

（回想一下，FST中虚线对应的片段，这些片段可能会在随后的key插入时发生变化。）

（Recall that the dotted lines correspond to pieces of the FST that may change on subsequent key insertion.）

这是不是很有趣，值得注意的是输出2并不一定需要放置在第一个转换上。从技术上讲，以下转换器同样正确：

This isn’t so interesting, but it is at least worth noting that the output 2 is placed on the first transition. Technically, the following transducer would be equally correct:

![](/img/days3-fst-1-alt.png)

但是，将输出放置在尽可能接近初始状态的位置可以更容易地编写一个在key之间共享输出转换的算法。

However, placing the outputs as close to the initial state as possible makes it much easier to write an algorithm that shares output transitions between keys.

让我们继续向map插入thurs=5：

Let’s move on to inserting the key thurs mapped to the value 5:

![](/img/days3-fst-2.png)

与 FSA 构造一样，插入key=thurs使我们可以得出mon的部分永远不会改变的结论。（如图中蓝色所示。）

As with FSA construction, insertion of the key thurs allows us to conclude that the mon portion of the FST will never change. (As represented in the image by the color blue.)

由于mon和thurs key不共享公共前缀，并且它们是map中仅有的两个键，因此它们的整个输出值都可以放置在从起始状态开始的第一个转换中。

Since the mon and thurs keys don’t share a common prefix and they are the only two keys in the map, their entire output values can each be placed in the first transition out of the start state.

然而，当我们添加下一个key tues时，事情变得更有趣了：

However, when we add the next key, tues, things get a little more interesting:

![](/img/days3-fst-3.png)

与 FSA 构造一样，这证明了FST的另一部分永远不会改变因此冻结它。这里的不同之处在于，从0到4的状态转换已经变成了从5变为3。这是因为tues key的value是3，所以如果将初始t转换加到value 5上，那么 value 就太大了。我们希望尽可能多地共享结构，所以当我们识别一个公共前缀时，我们也在输出值中寻找公共前缀。在这种情况下， 5和3的前缀时3。由于3是与key=tues关联的值，因此其剩余的转换都是0输出。

As with FSA construction, this identifies another portion of the FST that can never change and freezes it. The difference here is that the output on the transition from state 0 to 4 has changed from 5 to 3. This is because the tues key’s value is 3, so if the initial t transition added 5 to the value, then the value would be too big. We want to share as much structure as is possible, so when we identify a common prefix, we look for the common prefix in the output values as well. In this case, the prefix of 5 and 3 is 3. Since 3 is the value associated with the key tues, its remaining transitions can all have an output of 0.

但是，如果我们将转换的输出0->4从更改5->3，则与key thurs关联的值现在将是错误的。然后，我们必须“推送”5和3前缀中剩余的值。在这种情况下5 - 3 = 2，所以我们添加2到下一个转换4上（除了我们添加的新转换u）。

However, if we changed the output of the 0->4 transition from 5 to 3, the value associated with the key thurs would now be wrong. We then have to “push” the left over value from taking the prefix of 5 and 3 down. In this case 5 - 3 = 2, so we add 2 to each transition on 4 (except for the new u transition we added).

通过这种方式，我们保留先前key的输出，为新key添加新输出，并在FST中共享尽可能多的结构。

In this way, we preserve the outputs of previous keys, add a new output for a new key and share as much structure as possible in the FST.

和以前一样，让我们​​尝试再添加一个key。这一次，让我们选择一个对输出产生更有趣影响的key。让我们添加tye=99到map，看看会发生什么。

As with before, let’s try adding one more key. This time, let’s pick a key that has a more interesting impact on outputs. Let’s add tye to the map and associate it with the value 99 to see what happens.

![](/img/days3-fst-4.png)

插入key=tye允许我们冻结key tues的一部分 。特别的是，与 FSA 构造一样，我们确定等效状态，以便thurs和tues可以在 FST 中共享状态。

Insertion of the tye key allowed us to freeze the es part of the tues key. In particular, as with FSA construction, we identified equivalent states so that thurs and tues could share states in the FST.

FST 构造的不同之处在于，与4->9转换相关的输出（刚刚为key tye添加）为 96. 它之所以选择96，是因为在它之前的转换，0->4，有一个输出3。由于 99 and 33的公共前缀是3，所以0->4输出保持不变，而4->9的输出设置为99 - 3 = 96。

What is different here for FST construction is that the output associated with the 4->9 transition (which was just added for the tye key) has an output of 96. It chose 96 because the transition prior to it, 0->4, has an output of 3. Since the common prefix of 99 and 3 is 3, the output of 0->4 is left unchanged, and the output for 4->9 is set to 99 - 3 = 96.

为了完整起见，这里是表示不再添加key后的最终 FST：

For completeness, here is the final FST after indicating that no more keys will be added:

![](/img/days3-fst-5.png)

与上一步相比，这里唯一真正的变化是key tye的最终转换连接到所有其他key共享的最终状态。

The only real change here from the previous step is that the final transition of the tye key is connected to the final state shared by all other keys.

### 实操 Construction in practice

实际上，编写代码来实现上面图示描述的算法有点超出了本文的范围。（它的快速实现当然可以在我的fst 库中免费获得。）但是，有一些重要的挑战值得讨论。

Actually writing the code to implement the pictorially described algorithms above is a bit beyond the scope of this article. (A fast implementation of it is of course freely available in my fst library.) However, there are some important challenges worth discussing.

FST 数据结构的关键用例之一是它能够存储和搜索大量key。这个目标与上述算法有些不一致，因为它需要将所有冻结状态保存在内存中。即，为了检测 FST 的某些部分是否可以针对给定的key重用，您必须能够实际搜索等效状态。

One of the critical use cases of an FST data structure is its ability to store and search a very large number of keys. This goal is somewhat at odds with the algorithm described above, since it requires one to keep all frozen states in memory. Namely, in order to detect whether there are parts of the FST that can be reused for a given key, you must be able to actually search for equivalent states.

描述该算法的文献（在下一节中链接）指出，可以为此使用哈希表，它提供对任何特定状态的恒定时间访问（假设一个良好的哈希函数）。这种方法的问题在于，为了将所有状态实际存储在内存中之外，哈希表通常会产生某种开销。

The literature which describes this algorithm (linked in the next section) states that one can use a hash table for this, which provides constant time access to any particular state (assuming a good hash function). The problem with this approach is that a hash table usually incurs some kind of overhead, in addition to actually storing all of the states in memory.

可以通过牺牲所得到的 FST 的保证最小性来减轻所需的繁重内存。也就是说，可以维护一个大小有界的哈希表。这意味着经常重用的状态被保留在哈希表中，而不太常见的重用状态被驱逐。在实践中，在我自己的非科学实验中，具有约10000个槽的哈希表取得了不错的折衷效果，并且非常接近最小化。（实际的实现稍微好一点，在每个 slot 中存储一个小的 LRU 缓存，这样如果两个公共但不同的节点映射到同一个桶，它们仍然可以被重用。）

It is possible to mitigate the onerous memory required by sacrificing guaranteed minimality of the resulting FST. Namely, one can maintain a hash table that is bounded in size. This means that commonly reused states are kept in the hash table while less commonly reused states are evicted. In practice, a hash table with about 10,000 slots achieves a decent compromise and closely approximates minimality in my own unscientific experiments. (The actual implementation does a little better and stores a small LRU cache in each slot, so that if two common but distinct nodes map to the same bucket, they can still be reused.)

使用仅存储一些状态的有界哈希表的一个有趣结果是 FST 的构造可以流式传输到磁盘上的文件。也就是说，当状态如前两节所述被冻结时，没有理由将所有状态都保存在内存中。相反，我们可以立即将它们写入磁盘（或套接字，或其他）。

An interesting consequence of using a bounded hash table which only stores some of the states is that construction of an FST can be streamed to a file on disk. Namely, when states are frozen as described in the previous two sections, there’s no reason to keep all of them in memory. Instead, we can immediately write them to disk (or a socket, or whatever).

最终结果是我们可以在线性时间和恒定内存中从预排序的key构造一个近似最小的 FST。

The end result is that we can construct an approximately minimal FST from pre-sorted keys in linear time and in constant memory.

### 参考

上面介绍的算法不是我自己的。（据我所知，我确实提出了 LRU 缓存的想法。但也就仅此而已了！）

The algorithms presented above are not my own. (I did, to the best of my knowledge, come up with the LRU cache idea. But that’s it!)

我从Incremental Construction of Minimal Acyclic Finite State Automata得到了 FSA 构造算法 。特别是，第 3 节在解释细节方面做得相当不错，总体而言，这篇论文还是一本不错的读物。

I got the algorithm for FSA construction from Incremental Construction of Minimal Acyclic Finite State Automata. In particular, section 3 does a reasonably good job of explaining the particulars, but the paper overall is a good read.

我从Direct Construction of Minimal Acyclic Subsequential Transducers得到了 FST 构造算法 。整篇论文读起来非常好，但我不得不在一周的时间里阅读它大约 3-5 次才能真正深入其中。论文末尾附近有算法的伪代码，一旦您的大脑理解了所有变量的含义，就非常易读。

I got the algorithm for FST construction from Direct Construction of Minimal Acyclic Subsequential Transducers. The whole paper is a really good read, but I had to read it about 3-5 times over the course of a week to really let it sink in. There is pseudo-code for the algorithm near the end of the paper, which is very readable once your brain gets acclimated to what all of the variables mean.

到目前为止，这两篇论文几乎涵盖了本文中的所有内容。但是，实际编写一个有效的实现还有更多值得一读的内容。特别是，本文不会详细介绍 FST 中如何表示节点和转换。简短的回答是，FST 的表示是内存中的字节序列，并且绝大多数状态只占用一个字节的空间。事实上，表示有限状态机是一个活跃的研究领域。以下是对我帮助最大的两篇论文：

Those two papers pretty much cover everything in the article so far. However, there is more worth reading to actually write an efficient implementation. In particular, this article will not cover in detail how nodes and transitions are represented in an FST. The short answer is that the representation of an FST is a sequence of bytes in memory and the vast majority of states take up exactly one byte of space. Indeed, representing finite state machines is an active area of research. Here are two papers that helped me the most:

+ Experiments with Automata Compression （不幸的是，如果您单击此链接，researchgate.net 似乎会将您重定向到一个非常不友好的 UI。如果您只想要 PDF，请复制链接并将其直接粘贴到您的地址栏中。实际文章在 PDF 的第 116 页或会议集的第 105 页。）

    Experiments with Automata Compression (Unfortunately, if you click on this link, researchgate.net will seem to redirect you to a very unfriendly UI. If you just want the PDF already, copy the link and paste it directly in your address bar. The actual article is on page 116 of the PDF or page 105 of the conference collection.)
+ Smaller Representation of Finite State Automata

Jan Daciuk’s的论文对该领域有非常出色且深入的概述（gzipped PostScript warning）

For an excellent but very long and in depth overview of the field, Jan Daciuk’s dissertation (gzipped PostScript warning) is excellent.

Comparison of Construction Algorithms for Minimal, Acyclic, Deterministic, Finite-State Automata from Sets of Strings 中关于构建算法的简短而温馨的实验性概述是非常好的

For a short and sweet experimentally motivated overview of construction algorithms, Comparison of Construction Algorithms for Minimal, Acyclic, Deterministic, Finite-State Automata from Sets of Strings is very good.