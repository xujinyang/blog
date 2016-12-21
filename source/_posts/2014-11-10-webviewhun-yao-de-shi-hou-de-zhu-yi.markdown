---
layout: post
title: "webView混淆的时候的注意"
date: 2014-11-10 23:01:08 +0800
comments: true
categories: 技术
---


今天在打release包的时候出现一个问题，打开应用就崩溃，但是debug包就没有这个问题，因为刚转到AndroidStudio下面还没有熟悉，所有很自然的想到哪里配置出了问题，折腾了一个小时，实在没辙了，就请教前辈，然后就被打脸了，我的问题是这样的

		1.出现了这个的问题，怎么才能打出log尼，没有log我追踪不到这个问题
        2.debug包没有问题，release包有问题，难道是我的密码libsinger.so放错了？

前辈是这样回答的：
你如果libsinger.so没有放错，密码也对的，那只有可能是打包配置的问题了，你先把build.gradle中的buildTypes-》release-》debuggable true

照做了，打出了log，一看是出现了a(input...) 的log，一看就是混淆的问题，

你是不是用了webview的js接口？

对啊

哦，那就知道了，你可能是混淆的时候把webview也混淆了，你怎么不看混淆配置的说明，巴拉巴拉。。

我就郁闷了，混淆上面的

    If your project uses WebView with JS, uncomment the following
     and specify the fully qualified class name to the JavaScript interface
	 class:

结果很明显了，就是这个问题，查了一下找到了解决方案

		把注释解除，把fqcn.of.javascript.interface.for.webview换成你自己定义的那个类名（包名也必须有，如果定义的是内部类，则是cn.wj.ui.WebViewActivity$myInterface），在4.1的系统上是没有问题了，但4.2的机子上还是不行，再找找，哦，原来是4.2以上版本调用js接口需要在方法使用声明@JavascriptInterface,然后混淆时可能会弄丢该声明导致，程序无法调用js，需要继续再配置文件中添加条件，

	-keepattributes *Annotation*
	-keepattributes *JavascriptInterface*

混淆的实例：

    -keepclassmembers class cn.xx.xx.Activity$AppAndroid {
      public *;
    }

    -keepattributes *Annotation*
    -keepattributes *JavascriptInterface*
        
        
对技术不求甚解，一直是我的老毛病，这样的细节，如果不碰到，我是自己看不到的，被打脸也是应该的。让我先哭会。。。