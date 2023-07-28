---
title: "用rust调用pytorch的一点记录"
date: 2023-07-28T09:39:55+08:00
draft: false
categories: rust
keywords: pytorch torch libtorch rust
---

最近在研究openAI的CLIP模型，但是python的水平不够，写起来总感觉别扭。就随手搜了一下，发现正好tch做了一层libtorch的包装，使用rust来调用pytorch还是相当顺手的，就跟着tch的README来做一点尝试。

解释一下：pytorch的底层其实就是libtorch，这个是使用c++写的，可以理解pytorch是libtorch的python包装。

以下所有尝试都是在windows的wsl上操作的，具体linux版本为ubuntu/22:04

使用tch的前提：
 + conda(我使用的是miniconda3)
 + rust(1.71.0)
 + python(3.11.4)
 + pytorch(2.0.1)


我在这里没有使用miniconda3的默认env，而是创建了一个tch的专用env，使用了以下命令

``` shell
    conda create -n rust_pytorch1 pytorch
```


命令的意思就是就是创建一个名字叫做rust_pytorch1的env，并在其中安装pytorch。

下面，我们使用命令激活这个env

``` shell
    conda activate rust_pytorch1
```


按照tch的README我们还需要下载libtorch，这个需要在页面根据自己的环境进行选择，我这边只有cpu环境，因此选择的是libtorch的cpu版本。我们可以在页面[https://pytorch.org/get-started/locally/](https://pytorch.org/get-started/locally/)自行选择自己需要的版本并下载，然后解压到指定目录即可。


我们通过export导入环境变量，以便程序知道libtorch的路径，使用以下命令

``` shell
    export LIBTORCH=/home/jiangbo212/libtorch
```


这个时候我们cargo run运行README上的Tensor的例子时，可能会得到一个cc的编译错误。这时需要执行以下命令，导入一个变量。具体原因暂时未知。

``` shell
    export LIBTORCH_CXX11_ABI=0
```


此时，我们运行cargo run，依旧可能得到以下错误：

``` shell
    error while loading shared libraries: libtorch_cpu.so: cannot open shared object file: No such file or directory
```


这是由于cargo找不到libtorch.so的路径，虽然我们在前面导入了LIBTORCH的位置。需要执行以下命令, 让cargo可以在LD_LIBRARY_PATH下找到libtorch的lib路径

``` shell
    export LD_LIBRARY_PATH=/home/jiangbo212/libtorch//lib:$LD_LIBRARY_PATH
```


最后，我们运行cargo run, 可以看到正常输出了Tensor。


