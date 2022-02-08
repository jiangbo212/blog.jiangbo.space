---
title: "自定义推送消息至自己的apple设备"
date: 2022-01-26T16:11:08+08:00
draft: false
categories: 技术
keywords: bark 推送 ios 苹果通知 apple通知 系统通知
---
今天逛v2ex时，看到了一个之前自己有想法但是一直没动手的应用。bark，可以让用户在没有自己的app的前提下，推送消息到个人的apple设备。

使用起来也很简单，下载bark应用，打开就可以看到各种推送接口。而且还可以测试。点击测试之后，一个简单的http get请求即打开在safari中，立刻手机也收到了通知。使用起来非常便捷。

看了看作者的github，发现作者还兼顾了隐私要求，可以让用户自己定义服务端，无需通过作者自己的服务器。按照作者的说明在自己的nas立刻搭建起来，全程花了不到10分钟。然后在app上设置了下自己的服务器地址，curl测试下，非常完美。推送即刻到达。后续研究下怎么把notion上的通知推送到自己的手机上。个人的bark服务器：[bark](http://bark.jiangbo.space). 欢迎大家使用，仅支持ipv6🤔.(现已支持ipv4，2022.02.05)


![自定义bark服务器](/img/WechatIMG45.jpeg)

![自定义bark服务器](/img/WechatIMG46.jpeg)