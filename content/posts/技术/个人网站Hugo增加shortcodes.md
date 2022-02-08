---
title: "个人网站Hugo增加shortcodes"
date: 2022-02-08T15:22:54+08:00
draft: false
categories: 技术
keywords: 个人网站 hugo shortcodes md markdown
---
前几天个人网站从wordpress迁移到了hugo。虽然是纯人工的，但是一切还算顺利。美中不足的文章原来图片上面的描述不见。之前图片描述是通过在markdown中自定义html实现的。没想到同样的方式到了hugo就不生效了。类似于下面这种代码直接加到md的。

```
    <p style="text-align:center;font-size:0.5em;">图片描述</p>
```

当时着急迁移，也就没细究，今天正好有空研究了下。

Hugo并不支持在md文档中直接添加html代码，我使用的主题paperMod更是直接将markdown中的html直接忽略。还好hugo已经考虑到了这种情况，它使用Shortcodes来解决这种扩展性的问题。

hugo添加Shortcodes的方式如下：

+ 找到你对应的hugo主题下的layouts>shortcodes目录。在目录下新建你自定义的html文件, 并在里面添加自定义的html代码。例如新加了文件imgda.html.

    ```
        <p style="text-align:{{ index .Params 0 }};font-size:0.5em;">{{ index .Params 1 | markdownify }}</p>
    ```

+ 直接在md中使用的shortcodes即可。
    
    ```
        {{<imgda center 测试>}}
    ```

+ 最终在页面上展示的代码将变为如下形式。
  
  ```
        <p style="text-align:center;font-size:0.5em;">测试</p>
  ```
这样一波下来，完美解决我的图片描述不显示的问题。有相同需求的同学都可以用我这种方式试试。