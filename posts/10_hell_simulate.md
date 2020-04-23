---
title: 深渊模拟器
date: 2020-04-23 08:42:37
permalink: hell_simulate
---

[发布地址](http://lab.goatman.me/hell-simulate/), 以及github的[链接](https://github.com/xdsoar/hell-simulate).

本身是在这个月初的时候做的东西, 初版发布的时候还在[nga发了帖子](https://nga.178.com/read.php?tid=21132573). 起因是dnf新的100级版本是个深渊版本, 想大致算一下需要投入的时间.

周末某天早上的时候突发奇想, 用python写了点代码用蒙特卡洛来算概率. 算了几组以后发觉对界面操作的需求还挺大的, 于是决定gui重构一版. 因为要画界面, 因此也就不想用python了, 于是盯上了vue. 本来是打算用vue+electron做一个跨平台的客户端版本的. electron也一直是我想了解的一项技术, 用vue只是单纯觉得更轻量一点, 上手可能可以快一点.

最后整套技术栈用的就是vue+typescript+electron(可有可无). 实际开发中基本感知不到electron, 包括调试和发布最后其实也只是基于传统web的形式. 开始趟了点小坑, 用的`electron-vue`的模板, 结果这玩意连启动hello world都会报错......后面改用`vue-cli`的`electron-builder`, 总算是顺利多了. 改typescript稍微花了点时间, 主要是vue本身的教程都是基于js的, 用ts的思路来理解的话多少有点不顺, [这篇博客](https://blog.logrocket.com/how-to-write-a-vue-js-app-completely-in-typescript/)看看还是蛮有收获的.

最后发布用了和博客类似的基础设施. 只不过之前博客用的是hexo的s3插件, 这次是更通用的npm插件, 以后如果有类似实验性质的项目, 应该都可以用类似的方式来发布. 我特意做了这个`lab`的二级域名, 也是希望以后能再往里面填一些东西. 因为原来用的就是s3, 所以这次发布也还是在s3, 然后用cloudfront做加速, 国内访问确实算不上非常快, 后面有机会再看是不是迁到阿里云之类国内的服务商吧. 本次部署的时候遇到了一些坑, 基本都是我在[这篇博文](../s3_blog_setup)里提到的, 想不到完全适用, 还是挺意外的.

发布以后我想着针对像多选, 更复杂的计算条件做了一些优化. 不过本身这个应用的使用场景还是蛮有限的, 后续是否继续更新, 就看心情了.....不过作为vue的上手项目, 还算不错.