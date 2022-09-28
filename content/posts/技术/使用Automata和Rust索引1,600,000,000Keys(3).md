---
title: "使用Automata和Rust索引1,600,000,000Keys(3)"
date: 2022-09-28T09:44:02+08:00
draft: false
draft: false
categories: 技术
keywords: Rust Automata fst RocksDB 状态机 有限状态机 倒排索引 Lucene 有序集合 fsa
---

### 有序map(Ordered maps)

与有序集合一样，有序map类似于map，但在map的键上定义了顺序。就像集合一样，有序map通常使用二叉搜索树或btree实现，无序map通常使用哈希表实现。在我们的例子中，我们将看一个使用确定性非循环有限状态转换器的实现（缩写为 FST）。

As with ordered sets, an ordered map is like a map, but with an ordering defined on the keys of the map. Just like sets, ordered maps are typically implemented with a binary search tree or a btree, and unordered maps are typically implemented with a hash table. In our case, we will look at an implementation that uses a deterministic acyclic finite state transducer (abbreviated FST).

确定性非循环有限状态转换器是一个有限状态机（前两个标准与上一节相同）：

A deterministic acyclic finite state transducer is a finite state machine that is (the first two criteria are the same as the previous section):

1. 确定性。这意味着在任何给定状态下，对于任何输入，最多可以遍历一个转换。

    Deterministic. This means that at any given state, there is at most one transition that can be traversed for any input.

2. 无环。这意味着不可能访问已经访问过的状态。

    Acyclic. This means that it is impossible to visit a state that has already been visited.

3. 一个转换器。这意味着有限状态机发出一个与给定机器的特定输入序列相关的值。当且仅当输入序列导致机器以最终状态结束时才会发出一个值

    A transducer. This means that the finite state machine emits a value associated with the specific sequence of inputs given to the machine. A value is emitted if and only if the sequence of inputs causes the machine to end in a final state.

换句话说，FST 就像 FSA，但给定一个键，它不会回答“是”/“否”，而是会回答“否”或“是”，这是与该键关联的值

In other words, an FST is just like an FSA, but instead of answering “yes”/“no” given a key, it will answer either “no” or “yes, and here’s the value associated with that key.”

在上一节中，表示一个集合只需要一个将key存储在机器的转换中。当且仅当它代表集合中的一个key时，机器“接受”一个输入序列。在这种情况下，map需要做的不仅仅是“接受”输入序列；它还需要返回与该key关联的值

In the previous section, representing a set only required one to store the keys in the transitions of the machine. The machine “accepts” an input sequence if and only if it represents a key in the set. In this case, a map needs to do more than just “accept” an input sequence; it also needs to return a value associated with that key.

将value与key关联的一种方法是将一些数据附加到每个转换。正如输入序列被消耗以将机器从一个状态移动到另一个状态一样，当机器从一个状态移动到另一个状态时，可以产生一个输出序列。这种额外的“power”使机器成为传感器。

One way to associate a value with a key is to attach some data to each transition. Just as an input sequence is consumed to move the machine from state to state, an output sequence can be produced as the machine moves from state to state. This additional “power” makes the machine a transducer.

让我们看一个带有一个元素的map示例，key jul与value 7相关联：

Let’s take a look at an example of a map with one element, jul, which is associated with the value 7:

![](/img/map1.png)

这台机器与对应的集合相同，只是j从状态0到状态1的的第一个转换具有与之关联的7输出。其他转换，u和l也有一个0与它们相关的输出，该输出未显示在图像中。

This machine is the same as the corresponding set, except that the first transition j from state 0 to 1 has the output 7 associated with it. The other transitions, u and l, also have an output 0 associated with them that isn’t shown in the image.

与集合一样，我们可以询问map是否包含key“jul”。但是我们还需要返回输出。以下是机器如何处理“jul”的键查找：

As with sets, we can ask the map if it contains the key “jul.” But we also need to return the output. Here’s how the machine processes a key lookup for “jul”:

+ 初始化value为0。 Initialize value to 0.
+ 给定j，FST 从起始状态0移动到1. 添加7到value. Given j, the FST moves from the start state 0 to 1. Add 7 to value.
+ 给定u，FST 从1移动到2。添加0到value. Given u, the FST moves from 1 to 2. Add 0 to value.
+ 给定l，FST 从2移动到3。添加0到value. Given l, the FST moves from 2 to 3. Add 0 to value.

由于所有输入都已输入 FST，我们现在可以问：FST 是否处于最终状态？它是，所以我们知道jul它在地图中。此外，我们可以报告与key jul关联的值为7。

Since all inputs have been fed to the FST, we can now ask: is the FST in a final state? It is, so we know jul is in the map. Additionally, we can report value as the value associated with the key jul, which is 7.

没那么神奇，对吧？这个例子有点太简单了。具有单个键的map不是很有指导意义。让我们看看当我们添加mar=3到map时会发生什么：

Not so amazing, right? The example is a bit too simplistic. A map with a single key isn’t very instructive. Let’s see what happens when we add mar to the map, associated with the value 3:

![](/img/map2.png)

开始状态已经增长了一个新的转换，m，输出为3。如果我们查找 key jul，那么这个过程和之前的 map 一样：我们会得到一个值7。如果我们查找 key mar，那么过程如下所示：

The start state has grown a new transition, m, with an output of 3. If we lookup the key jul, then the process is the same as in the previous map: we’ll get back a value of 7. If we lookup the key mar, then the process looks like this:

+ 初始化value为0。 Initialize value to 0.
+ 给定m，FST 从起始状态0移动到1. 添加3到value. Given m, the FST moves from the start state 0 to 1. Add 3 to value.
+ 给定a，FST 从1移动到2。添加0到value. Given a, the FST moves from 1 to 2. Add 0 to value.
+ 给定r，FST 从2移动到3。添加0到value. Given r, the FST moves from 2 to 3. Add 0 to value.

这里唯一的变化——除了跟随不同的输入转换——是在第一步移动中添加3。v由于所有后续移动相加0到value，value将报告与mar关联的值是3。

The only change here—other than following different input transitions—is that 3 was added to value in the first move. Since all subsequent moves add 0 to value, the machine reports 3 as the value associated with mar.

我们继续吧。当我们拥有共享公共前缀的键时会发生什么？考虑与上面相同的map，但添加了jun=6：

Let’s keep going. What happens when we have keys that share a common prefix? Consider the same map as above, but with the jun key added associated with the value 6:

![](/img/map3.png)

与集合一样，添加了一个额外的转换n连接状态5和3. 但是还有两个额外的变化！

As with sets, an additional n transition was added connecting states 5 and 3. But there were two additional changes!

1. 输入的0->4转换j将其输出从7更改为6。 The 0->4 transition for input j had its output changed from 7 to 6.
2. 输入的5->3转换l将其输出从0更改为1。 The 5->3 transition for input l had its output changed from 0 to 1.

输出中的这些更改非常重要，因为它现在更改了一些用于查找与key=jul关联的值的细节：

Those changes in outputs are really important, because it now changes some of the details for looking up the value associated with the key jul:

+ 初始化value为0。Initialize value to 0. 
+ 给定j，FST 从起始状态0移动到4. 添加6到value. Given j, the FST moves from the start state 0 to 4. Add 6 to value.
+ 给定u，FST 从4移动到5。添加0到value. Given u, the FST moves from 4 to 5. Add 0 to value.
+ 给定l，FST 从5移动到3。添加1到value. Given l, the FST moves from 5 to 3. Add 1 to value.

最终值仍然是7，但我们得出的值不同。我们只添加了6，而不是添加7到初始j转换，但我们通过在最终转换l中添加额外的1来相加。

The final value is still 7, but we arrived at the value differently. Instead of adding 7 in the initial j transition, we only added 6, but we made up the extra 1 by adding it in the final l transition.

我们还应该确信自己查找key=jun也是正确的：

We should also convince ourselves that looking up the jun key is correct too:

+ 初始化value为0。Initialize value to 0.
+ 给定j，FST 从起始状态0移动到4. 添加6到value. Given j, the FST moves from the start state 0 to 4. Add 6 to value.
+ 给定u，FST 从4 移动到5。添加0到value. Given u, the FST moves from 4 to 5. Add 0 to value.
+ 给定n，FST 从5 移动到3。添加0到value. Given n, the FST moves from 5 to 3. Add 0 to value.

第一个转换添加6到value，但我们从不添加任何其他超过0的value到任何后续转换。这是因为jun key没有经过jul相同的最终l的转换。这样，两个key都有不同的值，但我们这样做的方式是在具有公共前缀的key之间共享大部分数据结构。

The first transition adds 6 to value, but we never add anything more than 0 to value on any subsequent transitions. This is because the jun key does not go through the same final l transition that jul does. In this way, both keys have distinct values, but we’ve done it in a way that shares much of the data structure between keys with common prefixes.

实际上，实现这种共享的key属性是map中的每个key通过机器的唯一路径。因此，map中的每个key总会有一些转换组合，对于特定的key是唯一的。我们所要做的就是弄清楚如何沿着转换输出。（我们将在下一节中看到如何做到这一点。）

Indeed, the key property that enables this sharing is that each key in the map corresponds to a unique path through the machine. Therefore, there will always be some combination of transitions followed for each key that is unique to that particular key. All we have to do is figure out how to place the outputs along the transitions. (We will see how to do this in the next section.)

这种输出共享也适用于具有公共前缀和后缀的key。考虑key tuesday和 thursday, 相关值分别是3和5（为一周中的一天）

This sharing of outputs works for keys with both common prefixes and suffixes too. Consider the keys tuesday and thursday, associated with the values 3 and 5, respectively (for day of the week).

![](/img/map2-suffixes.png)

两个key都有一个共同的前缀t，和一个共同的后缀sday。请注意，与key相关的值也有一个共同的前缀来添加值。即，3可以写成3 + 0，5也可以写成3 + 2。这个想法被机器捕捉到了；公共前缀t的输出为3，而h转换（不存在于tuesday）具有与之相关2的输出。即，在查找 key tuesday 时，第一个输出t ，但不会跟随转换，因此不会发出与其相关2的输出。其余的转换有一个输出0，它不会改变最终值的输出。

Both keys have a common prefix, t, and a common suffix, sday. Notice that the values associated with the keys also have a common prefix with respect to addition on the values. Namely, 3 can be written as 3 + 0 and 5 can be written as 3 + 2. This idea is captured in the machine; the common prefix t has an output of 3, while the h transition (which is not present in tuesday) has the output 2 associated with it. Namely, when looking up the key tuesday, the first output on t will be emitted, but the h transition won’t be followed, so the 2 output associated with it won’t be emitted. The rest of the transitions have an output of 0, which does not change the final value emitted.

我描述输出的方式似乎有点限制。如果它们不是整数怎么办？实际上，可以在 FST 中使用的输出类型仅限于定义了以下操作的事物：

The way I’ve described outputs might seem a bit restrictive; what if they aren’t integers? Indeed, the types of outputs that can be used in an FST are limited to things with the following operations defined:

+ 加。 Addition
+ 减。 Subtraction
+ 前缀（即，查找两个输出的前缀）。 Prefix (i.e., find the prefix of two outputs).

输出还必须有一个加法恒等式 ，以下定律成立（I）：

Outputs must also have an additive identity, such that the following laws hold:

1. x + I = x
2. x - I = x
3. prefix(x, y) = I x和y不共享一个公共前缀。 when x and y do not share a common prefix

整数可以简单地满足这个代数（其中prefix定义为min），另外还有一个好处是它们非常小。可以制作其他类型来满足这个代数，但现在，我们只使用整数。

Integers satisfy this algebra trivially (where prefix is defined as min) with the added benefit that they are very small. Other types can be made to satisfy this algebra, but for now, we will only work with integers.

在上面的例子中我们只需要使用加法，但是我们需要另外两个操作来构建FST。这就是我们接下来要介绍的内容。

We only needed to use addition in the above examples, but we will need the other two operations for building a FST. That’s what we’ll cover next.