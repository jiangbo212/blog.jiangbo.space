---
title: "使用Cloudflare Pages及Meilsearch搭建一个Linux Git提交日志查询网站"
date: 2023-01-30T13:46:47+08:00
draft: false
categories: 技术
keywords:  Meilsearch Cloudflare workers "cloudflare functions" "Pages Binding Workers" "Cloudflare Page"
---

本文章主要受网站[https://linux-commits-search.typesense.org/](https://linux-commits-search.typesense.org/)启发，并根据其源码进行修改。

文章重点讲述项目部署在Cloudflare Pages过程，其他细节只做简略的叙述。目前可通过域名[https://linux-commit-search.jiangbo.space](https://linux-commit-search.jiangbo.space)进行访问。

网站本身是一个前后端分离的网站, 后台有meilsearch提供http服务。meilisearch是一个开源的搜索服务，可以理解为一个简单的ES服务，便于搭建和使用。

前端使用Parcel进行打包，js方面使用的还是比较老的jquery。使用instant-meilisearch和instantsearch进行搜索控件的构建，整体来说结构比较简单。如果进行常规的部署，还是相当简单的。之前我是在自己的轻量级服务器上进行的部署，只是用了nginx就轻松的部署完成。为什么要使用cloudflare呢？主要是考虑到服务的稳定性以及访问的广度，毕竟cloudflare的机器遍布全球，而我的服务器只有一台。同时，使用cloudflare pages可以轻松的做到部署自动化。

Cloudflare pages部署前端项目还是非常简单的，选择好git项目以及构建步骤之后，前端代码就轻松的部署到了Cloudflare的全球网络上了，配置好相应的域名之后，测试访问，前端页面展示的非常完美。

接下来问题就来了, 在自有服务器上可以简单实现的代理功能，在Cloudflare Pages上就比较麻烦了。我先后尝试了以下三种方式：

1. 使用Cloudflare Pages提供的Functions功能进行代理功能的实现。

    这里遇到的问题是，Functions提供的fetch功能，只能使用env.ASSET进行调用。开始的时候，我没有自己，认为fetch并不是什么大问题，直接将前端页面域名中的域名替换成meilisearch服务的后端页面进行调用。functions部署之后，进行简单调用，发现返回的竟然还是前端页面。这是什么鬼啊，我又详细的检查了一遍functions文档, 返现evn.ASSET只能访问CloudFlare Pages域名下的静态文件。这就太坑了，只能放弃这种处理方式了。

2. 使用Cloudflare Pages提供的Rediects功能进行代理功能的实现。

    这里面耽误的时间到不是很多，看文档的时候，我就看到了proxy功能是不支持的，说是未来将会支持。但是总归不信邪，试一下吧。打包阶段就通知我这种配置方式是不支持的。哎！

3. 使用Cloudflare Pages提供的Functions与Workers绑定的功能实现。

    不得不说，Cloudflare Pages中的Functions的文档写的有点稀碎，同时很多措辞感觉有点模棱两可的感觉，也不知道是不是我英语不过关的原因。Functions与Workers绑定的文档我看了好多次，也尝试了多次，才最终完成绑定的工作。

    在配置界面进行Functions和Workers(Service)进行绑定的时候，页面显示的Variable Name。在文档界面，又显示的是select the environment， 同时文档界面Service bindings下面就是Environment variables, 搞的我一直以为设置界面设置的参数是环境变量而已，是用来区分环境的。真正调用workers直接使用文档里面的例子就可以了。

    Functions与Service绑定配置页面的配置：

    ![Functions与Service绑定配置页面](/img/20230130142855.png)

    {{<imgda center Functions与Service绑定配置页面>}}

    Functions与Service绑定的文档页面：

    ![Functions与Service绑定的文档页面](/img/20230130143208.png)

    {{<imgda center Functions与Service绑定的文档页面>}}


    文档中的调用服务的例子如下：

    ``` javascript
    export async function onRequestGet(context) {    
        return context.env.SERVICE.fetch(context.request);
    }
    ```

    我一直认为其中的SERVICE是固定写法，文档中是这样写的："Here is an example of how to use service bindings in your Function, our service binding is named “SERVICE”"。同样的写法我怎么调用，都直接告诉我系统异常。困扰了我好久，后来我突然想到，是不是我在设置页面设置的变量值才是env.后面应该跟随的参数，而不是固定的SERVICE。我进行了尝试，果然一下就成功了。后续再看了文档，才有点理解。

解决了functions与service绑定的问题之后，网站终于完整的部署好了。整体过程还是相当流程的，使用workers进行调用的过程也是几乎没有延迟，非常完美。

整个过程中workers的开发环境真的是非常友好，对于简单的需求，完全可以直接在网页上进行开发，并进行调试，整个过程相当流畅，有些简单的需求在workers上可以快速的实现。

