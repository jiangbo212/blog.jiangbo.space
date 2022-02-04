---
title: "前公司OA生产事故分析"
date: 2016-05-16T01:58:51+08:00
draft: false
categories: 历史迁移
---

1. 现象 

    17:27分OA重启服务器之后，系统日志显示重启完成，前台少部分页面可以打开，大部分页 面打不开或报错。后台日志显示It cannot be assigned to jsonProcDefKeys。如下错误： 

    ![错误信息](/img/WX20220125-162140@2x-1024x382.png)

2. 分析。

    a. 日志信息中的报错原因在于jsonProcDefKeys为NULL。jsonProcDefKeys是在FormManagement中进行定义的，且在static块中进行初始化。判断static块初始化未执行，或未执行完成。

    b. FormManagement进行初始化的情况。FormManagement的初始化时在web容器加载成功之后，另起线程掉用FormManagement的空得静态方法来执行static块内容，进行初始化的。
    ![](/img/WX20220125-162200@2x-1024x802.png)

    c. 即使在web容器加载成功之后并没有调用的FormManagement的静态方法init进行初始化。在用户访问时，只要引用的地方调用到FormManagement的静态变量或静态方法，还是执行static块进行初始化的，因此可以排除FormManagement未执行static进行初始化。于是判断是static块未被执行完成

    d. 之前步骤的分析没有考虑邮件服务的问题，虽然知道邮件服务也有问题，但一直认为邮件系统是即使有问题也是会报错的，为考虑的假死的状况。在明确邮件服务假死之后，查看重启之后thread dump日志，发现大量的线程(86个)由于邮件服务假死而一直runnable状态，但被锁定在邮件服务区域。
    ![](/img/WX20220125-162757@2x-1024x579.png)

    e. 由于d步骤的分析，认为大量线程处于阻塞，且邮件服务假死会导致大量IO资源被占用。同时jsonProcDefKeys的初始化过程会需要使用大量IO资源，这样回导致jsonProcDefKeys的初始化速度太慢，而一直未执行完成。因此会导致前台页面或jsonProcDefKeys会因为空而报错。但这个原因无法被证明，于是在thread dump日志中查找FormManagement的信息。

    f. FormManagement在thread dump日志中的信息如下： 
    ![](/img/WX20220125-162302@2x-1024x444.png)

    g. 从thread dump中可以清晰的看出FormManagement在已经在执行static块中的初始化方法。但是由于static块中会使用SystemManagement.healthCheck()方法执行同步发邮件操作，导致线程被锁死在邮件服务区域，未完成初始化。

    h. 查看thread dump中的http线程可以发现，有376个的http线程在代码层面已经开始执行(Runnable)，但是线程缺是等待状态（Object.wait()）。查看打印出的方法为StartProcAction.preStartProc(StartProcAction.java:165)，WaitProcessDto.getProcTitleTaskUrl(WaitProcessDto.java:449)，这两个方法的具体行数都是调用FormManagement的静态方法或静态变量。但是由于FormManagement的static块都未执行完毕，所以FormManagement不能返回值，因此这些线程都处于等待状态，从而导致不能响应前台。http线程处于等待状态示例：  
    ![](/img/WX20220125-162324@2x-1024x256.png)
    ![](/img/WX20220125-162352@2x.png)

3. 结论


    分析中在第g点从thread dump日志的已经分析出是因为邮件服务的假死导致FormManagement中static未执行完，从而导致jsonProcDefKeys的值获取不到。而第h点则验证了这一分析：由于FormManagement未初始化完成，导致所有调用FormManagement的http线程都因为获取不到返回值而在执行后处于等待状态。
