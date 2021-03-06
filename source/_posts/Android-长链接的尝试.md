title: Android 长链接的尝试
date: 2015-12-18 20:32:46
tags: Android

---

产品需要不停的上传位置，首先想到Service，获取到位置就上传，但是如果时间间隔缩小到3秒一次，那么HTTP的方式就不太适用了，比较用户多的时候，那服务器的压力是成本的增长的，业界通用的方法，比如携程听说整个app就一个TCP通道，使用自定义的协议，所有的请求和返回都走这个通道，携程除了隔三差五的来点事故，其他技术还是不错的，那么我自然想到了长链接，使用TCP三次握手之后的长链接进行位置上传。

关于长连接和短连接的区别，可以在下面文章中找到：
[Http长连接和短连接](http://blog.csdn.net/shine0181/article/details/7799754/)
[TCP长连接和短连接](http://www.cnblogs.com/liuyong/archive/2011/07/01/2095487.html)

这里说的是Http的短连接，和TCP的长链接，既然方向明确了，就开始调研现有的开源项目，发现了Socket.io这个项目，用了半天的时间，实现了Socket.io的版本，经过一段时间的测试，将上传的位置在地图上画出来得到了下面的线：
![这里写图片描述](http://img.blog.csdn.net/20150703090641934)
地图上看效果很不错，基本上和百度地图画的路径差不多，3秒一次的间隔让线很平滑，没有跳跃的点，说明位置上传的到达率很高，有这样的效果让我很高兴，Socket.io非常适合做这样的事情，但是万万没有想到的是，晚上一看手机，发现整个app一条跑了105M的流量，我赶紧用Charlse抓了手机的请求,发现Socket.io 有大量的心跳和重连请求，而且是http形式的，心跳大小在564byte，重连请求很过分，没个请求尽然有近867byte的数据，而且大部分在请求头上。
![重连请求]
(http://img.blog.csdn.net/20150704184255507)
心跳大小
![请求大小](http://img.blog.csdn.net/20150704184330948)

当时就傻眼了，测试的时候，没有发现那么多的心跳和重连，缺乏经验，评审的时候，数据量都是按照上传位置的请求算的，没想到Socket.io内部的心跳机制那么变态。

放弃了Socket.io 准备自己造轮子，按照我们的需求，3秒钟上传一次位置，完全可以用UDP，中间丢失少量的点是被允许的,UDP的好处是不用维护长连接，应该说就没有连接。

```
TCP---传输控制协议,提供的是面向连接、可靠的字节流服务。当客户和服务器彼此交换数据前，必须先在双方之间建立一个TCP连接，之后才能传输数据。TCP提供超时重发，丢弃重复数据，检验数据，流量控制等功能，保证数据能从一端传到另一端。 
```

```
UDP---用户数据报协议，是一个简单的面向数据报的运输层协议。UDP不提供可靠性，它只是把应用程序传给IP层的数据报发送出去，但是并不能保证它们能到达目的地。由于UDP在传输数据报前不用在客户和服务器之间建立一个连接，且没有超时重发等机制，故而传输速度很快
```
又花了点时间，用UDP做了尝试，10秒上传一次位置，得到了下面的图
![UDP10秒每次](http://img.blog.csdn.net/20150704185620299)

效果比预计的要好，大部分的点都很平滑，有部分跳跃的点，是因为使用者进入了建筑丢失了GPS信号，瞬间移动基站给出的点误差较大或者是中间丢失了点，但是这些点可以使用技术手段过滤掉，总体来说，UDP是一个可行的方案，实际使用情况，我们还需要灰度验证。


这是我最近摸索长链接方案尝试，因为你懂的原因，只能简单概括方案，给需要的人一下参考。