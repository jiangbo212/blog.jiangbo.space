---
title: "家庭宽带折腾ipv6"
date: 2022-01-05T21:58:20+08:00
draft: false
categories: 技术
keywords: ipv6 netflix 中国移动 光猫 管理员 traceroute 防火墙策略
---
也不知最近怎么突然想起来要看netflix，然后就发现就只能看看自制剧。也切换过不通的节点，但是依旧不行。网上查了下，说是netflix在8月的时候切换了策略。大部分梯子就熄火了，最多也就看看netflix的自制剧了。研究了一下，还是ip的问题，同一ip使用的netflix超过3个，netflix就限制只能看看它自己的自制剧，不得不说这手是真的高。这样搞法大部分的梯子就没用了，毕竟对机场来说费用最昂贵的就是ip了。

但是一般来说贵的是ipv4的地址，ipv6的地址理论上不值钱，ipv6的地址都是按段来的，可以说几乎是无限供应，netflix也不应该会封掉那么多的ipv6地址。不过这也是理论上的，具体还是要看实践，现阶段还是先解决家庭宽带可以访问ipv6的问题了。

托国家推广ipv6的福，现在各大运营商基本上都是在卖力的支持ipv6，我家用的几乎免费的移动管带也是支持的。不过我家的那个我自己后面配的腾达路由器是个垃圾，还是我选的2019最冒尖的货，可惜了。只能将它放到出生的盒子，等我找个机会在闲鱼上出掉，期望可以挽回点损失。

连上移动光猫自带的wifi，首先的想法就是登上路由器的管理地址，192.168.1.1。到这里不得不说程序员的思维害死人，虽然打电话给了移动宽带的片区维修，人家也肯定的答复我说给我支持了ipv6，但我依旧是不相信，总觉得那些小哥啥都不知道，总是自己最聪明。用路由器背面的用户名密码登陆进光猫，好家伙，几乎是啥都没有，只能看看是谁连接到路由器，对于我这种专业用户来讲，基本就相当于没有。没法子，只能百度大法好。但是试了几个网上的密码都不行，最后只能继续找维修小哥，维修小哥也是费了大劲，向上级申请到了我的光猫超级管理员用户密码。仅拿到密码就花了我几个小时，差点累出了老血。在此奉劝各位如果没有什么大的欲望，也没啥动手能力，就不要想我这样折腾了。一般小哥都是讲ipv4/ipv6都配置好的。我这个光猫后来才发现其实也是配置好的，电脑，手机什么的也是拿到了ipv6地址，之所以不能用的原因竟然是我的代理问题，也是哔了狗了。所以大家要切记目前为止，一遍的代理都是不支持ipv6的，如果需要明确使用ipv6，一定要记得做特殊配置。对于clash而言，就是在配置文件中加上ipv6: true即可。

拿到光猫超级管理员密码的时候，其实我已经知道我自己已经是正常的ipv6用户了。不过最终我也没浪费辛苦得来的密码，有些我自己的特殊需求还是用到它了。既然使用到了ipv6，而且发现都是公网时候，我就将我的博客和群晖的地址都切换到ipv6地址上了，反正这些也就是我自己自娱自乐的东西。当我想访问他们的时候，随时掏出手机连上热点即可，移动大内网的ipv6地址，简直不要太稳。

就当我要在我手机上访问我的博客的时候，竟然发现访问不了。这是真的奇怪了，按说都是移动的亲生儿子，不应该了。没办法，只能自己动手操作。电脑连上手机热点，然后访问我群晖上的blog，依旧是访问不了。这也在意料之中，毕竟手机热点，手机访问不了，那连接热点的电脑当然也不行。

下面就有点计算机专业知识的味道了。traceroute我的群晖ipv6地址，一条一条又一条，一行一行又一行，路由一步步的逼近我的群晖，但是终于它停顿了，停顿了。看着他停顿的地方，有点熟悉了。前缀竟然都一样，登上我管理员权限的光猫，真想它来了。路由竟然止步在了我的光猫上，进不去了。到这里问题就很好解决了，光猫干的路由器的货，不可能路由过不去的，那么真想只有一个了。光猫的防火墙阻止了它。轻轻的点击几下，关闭了光猫的防火墙。再刷新下页面，我的博客流程的出现了在眼前。

由此，家庭宽带的ipv6支持告一段落。可以访问ipv6的网站了，也拥有了仅ipv6支持的网站。ipv6支持的家庭网络pt下载简直飞起，都超出了理论上移动宽带的最大上限。下一步就是看怎么用ipv6支持看netflix了。也不知下次更文是啥时候了。
