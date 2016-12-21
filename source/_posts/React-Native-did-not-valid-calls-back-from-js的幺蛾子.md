---
title: 'React-Native:did not valid calls back from js的幺蛾子'
date: 2016-11-08 17:45:10
tags: React-Native
---

	did not valid calls back from js [] [] []

第一次看到这个问题，我是崩溃的。

应为提示的信息基本上没有什么帮助，但是奇怪是，我打开js debug模式，就可以正常跑通。

经过2个小时的各种调试


最后的最后

聚焦到一个时间转换函数

	2015-07-08 23:59:59

这样格式的时间，使用new Date(endTime).getTime()来获取时间戳，会变成NaN,NaN传到了下面就导致了调用了ToastAndroid.show(new String(msg), ToastAndroid.SHORT);就报了did not valid calls back from js的崩溃。。

但是打开JS Debug 模式就发现一切正常。

结论直接说出来，Js Debug 的js运行环境是chrome，使用的是V8引擎，而本地的js跑再jscore上，2015-07-08这个格式时间，在这俩个引擎上有兼容问题，[兼容情况具体看这里](http://dygraphs.com/date-formats.html)，在jscore上需要把 - 换成/,也就是2015-07-08 23:59:59改成2015/07/08 23:59:59,使用下面的代码

```

        var endTime = '2015-07-08 23:59:59';

        endTime = endTime.replace(/\-/g, '/');

        endTime = new Date(endTime).getTime();
                    
```
然后还有个问题ToastAndroid,为啥抛这个错误尼，原来是很弱智的问题，我没有开启debug模式,
so ,遇到did not valid calls back from js 这个bug，先看看是不是没有开debug模式

```
  @Override
        protected boolean getUseDeveloperSupport() {
            return true;
        }
```

然后我天真的以为问题已经搞定了，但是下午开发的时候，有出现了这个问题，而且很明确是的还是ToastAndroid 的原因，toast相关代码我是这样写的

```
export function ToastShort(msg) {
    if (Platform.OS === 'ios') {
        ToastIOS.showMessage(msg);
    } else {
        ToastAndroid.show(new String(msg), ToastAndroid.SHORT);
    }
};
```
就这样一段简单的代码，能出什么幺蛾子？在native那边打断点都跳不过去就崩溃了，难道是new String()的原因？在chrome上打印了一下

```
new String('test')

=>String {0: "t", 1: "e", 2: "s", 3: "t", length: 4, [[PrimitiveValue]]: "test"}
```

我操，换一个在线js 编译器，

```
new String('test')
"test"
```

难道又是js引擎兼容的问题？打个线上的包，试一下，没有问题，debug就跪。。。

这样也解释的通，new String传给原生一个object，可是方法中第一个参数应该是String，那肯定就did not valid calls back from js [] [] []，那为啥线上包就没有问题尼，还没有想明白，如果有明白的请不吝赐教。
