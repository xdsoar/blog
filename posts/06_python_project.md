---
title: python项目
date: 2019-12-17 08:21:00
permalink: python_project
---

GTD, 或者说todo类的产品, 是我一直有频繁使用的一类应用. 不管是哪个平台, 从桌面系统windows\mac到移动端ios\android, 我都有不少尝试. 其中不乏佼佼者, 遗憾的是总是因为跨平台的一些原因, 在这个平台上好用的应用, 换了平台要么不好用了要么就是根本没有.(比如仅限mac系统的omnifocus, 仅限windows系统的mlo).

于是这次筹划[干脆自己写一个吧](https://github.com/xdsoar/TaskCommander). 初期打算先写一个cli版本的, 有时间了会再研究用诸如electron来写一个跨平台gui版本. 当然移动端依旧是个问题, 这个问题就交给以后去烦恼了.

这个应用的架构会按照这一年里实践了很多次的**洋葱架构**来写. 关于洋葱架构, 以后有时间会具体来写.

这次主要搞定的是python项目发布的问题. 既然是做成cli应用, 那么能用pip安装自然是最好的.

python的项目结构参考了<https://python-packaging.readthedocs.io/en/latest/minimal.html>这里的说明. 最小化发布至少需要有一个setup.py文件, 其中包含如发布的名字, 版本号, 依赖等信息. 这里的依赖配置还没研究透, 原来的习惯是写一个`requirement.txt`文件, 然后在部署的时候`pip install`, 不过这种做法对于发布成pip包的应用大概不是很友好.

发布的环节与上面网页上描述的略有不同. 因为在使用`sdist upload`时看到提示建议使用`twine`来做upload. 因此发布前先`pip install twine`, 然后使用`twine uploa`命令发布制品. **发布前需要先去pypi网站上注册账号, 然后才能发布.**
