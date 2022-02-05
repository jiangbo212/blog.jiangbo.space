---
title: "群晖+ipv6+cloudflare实现外部访问"
date: 2022-01-27T17:52:12+08:00
draft: false
categories: 技术
---

前提🎈🎈🎈🎈🎈🎈：拥有自己的域名。



域名如果没有的话，可以找供应商买一个，一般的小众域名10年也就100来块。国内腾讯云和阿里云都提供的。



家里的nas之前折腾了好久的ipv6，终于实现了ipv6的对外访问。美中不足的是现在好多地方的wifi仍旧是不支持ipv6的。虽然可以通过中国移动的热点解决。但终究还是不够完美。



今天突然想起来，cloudflare是支持免费cdn的。登上自己好久不用的cloudflare账户，看了下果然是可以的。剩下的就是根据提示一步步添加自己的网站。要注意的一点是cloudflare同步原来的域名供应商的解析时可能会漏掉一部分解析。这个需要自己小心检查下，可以手工再补上。其他的就按照cloudflare的提示操作即可。



在cloudflare上添加好自己的网站，并确定把自己的域名解析服务器切换到cloudflare的域名服务器上(这个解析是要在原来域名解析的供应商出修改，比如我的域名是在腾讯云上购买的，那么就需要在腾讯云上进行调整)。如果你调整好了，那么剩下就要等待了。cloudflare发现你的解析已经生效之后，在页面上时可以看到你的域名已经被cloudflare托管了。



后续你的域名解析就都可以在cloudflare上配置了。注意配置的时候要选择被cloudflare代理，这样网络流量就先走cloudflare，然后才是你的服务器。这样配置完之后，你的dns解析地址就可以只配置AAAA记录(也就是ipv6解析)。因为访问你网站的不再是真实客户，而是cloudflare。cloudflare就是v4/v6通吃了。



以下为特殊注意的地方



一个是如果你的网站是http，而你在cloudflare上勾选了强制https的话。那么你的网站可能会有一定几率报错。因为网站上的某些连接并不支持强制https。我这里的wordpress就遇到了这种情况。解决方案就是上插件。worepress插件<a href="https://cn.wordpress.org/plugins/really-simple-ssl/" target="_blank" rel="noopener">really simple ssl</a>非常好用，安装完之后，网站就再也没有遇到这个错误了。



第二个就是cloudflare上的DNS解析的DDNS问题了。一般家庭宽带是没有固定的ip的。即使ipv6运营商也没有给固定的。这个时候就需要DDNS解决这个问题了。这个大家可以参考<a href="https://github.com/timothymiller/cloudflare-ddns" target="_blank" rel="noopener">cloudflare-ddns</a>，非常好用，配置也简单。docker和linux-cron用起来都是很简单的。

