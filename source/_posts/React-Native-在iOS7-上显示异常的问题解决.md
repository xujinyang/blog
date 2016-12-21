---
title: React-Native 在iOS7 上显示异常的问题解决
date: 2016-11-09 23:49:43
tags: React-Native
---

这个坑踩的有点奇怪，值得记录一下，要说怎么个奇怪尼？看下面的图


![](http://7o4zmy.com1.z0.glb.clouddn.com/Simulator%20Screen%20Shot%202016年11月8日%20下午9.15.30.png)

就是很普通的View，为啥会出现奇怪的阴影尼？

各种google，issues 无果

排除法，把奇怪的属性都去掉

果然发现了

	borderWidth: 0.5

这个属性小于1的时候，在iOS平台上有兼容性问题，原因不详.
#### 解决方案:

忍了，把这个值改成了1，然后颜色调浅