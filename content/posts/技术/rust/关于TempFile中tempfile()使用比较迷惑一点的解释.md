---
title: "关于TempFile中tempfile()使用比较迷惑一点的解释"
date: 2023-01-14T07:22:30+08:00
draft: false
categories: rust
keywords: rust tempfile() TempFile seek meilisearch
---

在我最初向meilisearch提交PR[3164](https://github.com/meilisearch/meilisearch/pull/3164)时，我是使用NamedTempFile来创建临时文件的。当时审核人员在审核之后给出了建议：

“从需求上看，这里使用tempfile()可能更合适，因为我们不需要去知道文件名字或者持久化该文件”

在查看了Tempfile这个crate之后，我认可了这个建议，同时也使用tempfile()调整了相关代码。

在调整的过程中我遇到了一个非常奇怪的问题。在使用NamedTempFile时，我写入文件之后，再进行读取，中间不需要任何其他的步骤。但是在我调整直接使用tempfile()方法之后，直接进行读取，那么读到的是一个空的文件，这个问题花费了我好多时间，终于找到了一个可以使用的方案，那就是在写入和读取之间加入如下一行代码：

``` rust
    // Seek to start
    buffer.seek(std::io::SeekFrom::Start(0)).await
```

如果你是直接使用tempfile进行读写，那么下面这个代码正是你需要的：

``` rust
    // Seek to start
    tmpfile.seek(SeekFrom::Start(0)).unwrap();
```

seek()方法做的事情很简单，就是把文件里面的cursor(游标)指向它开头的位置。这个就让人很疑惑了，感觉这里也没有什么实际的动作呢。怎么就让原本写入读取不到数据的文件，从而读取到数据呢？这个问题困扰了我很长时间，即使在相关的PR已经合并完之后。

后续我在研究milli的源码的时候，发现里面使用tempfile()的地方同样使用了相同的代码。我意识到这应该不是一个偶然的用法，而是必须添加这一步才能够正常的读取到tempfile()创建的文件。我又去翻阅了一遍Tempfile crate，在它的github的README中看到了相同的写法，但是也没有进一步说明为什么要这么做。

最终我在tempfile的github issues中找到了明确的答案。具体issue号是[142](https://github.com/Stebalien/tempfile/issues/142)。

2021年的时候有人跟我提出了相同的疑问，作者做出了以下的回复：

"After you write to the temporary file, the file "cursor" well be at the end. You need to seek back to the beginning before you can copy."

意思吗就是在tempfile写入之后，文件的cursor(游标)指向了文件结束的位置，这个时候如果需要进行copy，需要重新将cursor指向到文件开头的位置。这就很明确了，读取的时候其实也是类似的问题，读取的时候也是要通过cursor进行读取，从文件的末尾进行读取，当然读取的是0字节。当使用seek()将cursor重新指向文件开头，就可以正确的读取到所有数据了。
