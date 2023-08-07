---
title: "使用rust入门discord机器人遇到的一些坑"
date: 2023-08-07T17:29:37+08:00
draft: false
categories: rust
keywords: rust serenity discord bot 
---

最近几天在研究discord，其实之前也玩过，只不过当时没怎么玩下去，感觉怎么用都不知道。这两天感觉突然开悟了，用起来非常顺手，突发奇想弄个discord的bot玩玩。

快进到官网的教程，在进入开发之前的步骤非常流程，新加的机器人顺利的加入了群组，但是接下来的就看不懂了。

官网的示例是用的http做的，我研究了一下rust下的相关库，找到了serenity。看了看它的hello world，倒是也正常跑起来了，但是这鬼东西，连个http的端口都没有，怎么让discord访问呢？

接着又去研究官网，研究了了一两天，我发现discord还有一种使用websocket的连接方案。猜一下，大概率serenity大概率用的这种方案，这也就解释了它为什么不需要http的端口了。直接都连上了discord服务器的websocket，还需要什么自行车。暴露什么端口徒增风险呢。

仔细一想，serenity使用websocket其实是满明智的，Im这种应用消息的频率是相当高的，如果使用http的话，目标服务器的负载基本上都是很大的，毕竟很大一部分bot是需要读取频道里面的每条消息的，一个成熟的bot是会有相当多的频段和用户消息的，这个时候可以复用tcp通道，主动接受推送的websocket简直不要太合适。

知道了serenity是使用websocket的，那么它的示例应该是可以正常让机器人在线的啊，我应用都跑起来了，不科学啊，这么广泛使用的类库不应该会犯这么低级的错误。

这个时候尝试了一下telnet discord的wss服务器，我靠，果然是不通的，这个示例怎么还一直正常运行，这不是坑爹吗。调整日志级别，打印出serenity的DEBUG日志，看日志果然是连接discord的wss服务器异常，而且DEBUG级别日志下，应用自己报错关闭了，真坑。

知道网络不通剩下就好办了，打开clash for windows的tun模式，再次启动程序。我的discord bot终于正常上线了。

在频道中输入!ping, bot自动回复了Pong。终于discord的入门级bot完成了！！！！