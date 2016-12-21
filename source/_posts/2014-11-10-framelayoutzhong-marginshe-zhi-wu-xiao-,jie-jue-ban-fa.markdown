---
layout: post
title: "FrameLayout中Margin设置无效，解决办法"
date: 2014-11-10 19:59:09 +0800
comments: true
categories: 技术
---

今天突然发现一个奇葩的问题，在3.0以下的手机中，在FrameLayout内部放了一个ListView，给ListView 设置了一个margin_Left,发现没有起作用，反而在右边出现了10dp的margin，实验了几次后发现，无论是margin_Left还是margin_Right都是在右边出现margin，俩个同时设置的时候发现还是叠加的显示在右边。

解决方案：

			在内部布局中设置 android:layout_gravity="top"

就可以解决2.x上兼容的问题


想起来用RelativeLayout内部布局中设置布局也会出现MarginBottom设置无效的情况，必须在RelativeLayout中设置paddingBottom来代替。