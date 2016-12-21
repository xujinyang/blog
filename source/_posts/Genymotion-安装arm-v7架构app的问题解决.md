title: Genymotion 安装arm-v7架构app的问题解决
date: 2016-02-24 22:52:53
tags: Android

---

关于arm架构app安装在x86上的原因可以参考这篇[博客](http://blog.csdn.net/iaiti/article/details/41720039)

说来惭愧，人家已经给出了解决方案，但是这个问题我搞了半天，一直都报

		an error occurred while deploying a file install_failed_no_machine_abis

为什么尼，我下载了无数遍Genymotion_ARM_Translation,
[Genymotion_ARM_Translation_v1.1.zip](http://download.csdn.net/detail/iaiti/8224603)
把插件拖动到模拟器中，然后重启模拟器，可是怎么都还是不能成功安装apk。
。
。。
。。。
。。。。
。。。。。
最后的最后，我特么终于发现了哪里姿势错了，原来每次下载Genymotion_ARM_Translation_v1.1.zip，我都是通过safari，而safari会自动的把zip文件解压，然后我有自己又手动的压缩成zip，某些信息的改变导致每次我拖动到模拟器中都是copy文件，而不是安装。

这样的错误让我真是无地自容。。。
