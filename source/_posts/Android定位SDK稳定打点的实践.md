title: Android定位SDK稳定打点的实践
date: 2016-01-19 23:20:11
tags: Android
---

		去年我和定位sdk打交道比较多，用过了腾讯定位，百度，现在换成了高德，说实话，腾讯的sdk在普通业务中还行，如果在重定位的o2o应用中，那么准确性，开发体验，文档api，耗电，耗流量方面都和百度，高德有点差距，腾讯最新的sdk没有体验，所以不知道现在如何。百度定位sdk，在api完善程度，电量，流量，精准度都和高德不相上下。下面来说一说，使用定位sdk稳定打点的体验。 这里说的稳定打点，是指相对稳定的间隔定位然后上传服务器。     

大家如果用过定位的sdk都应该知道sdk默认有个轮询的方法，只要设置setInterval（n）这个方法，就可以每隔n毫秒定位一次。那么稳定打点的难度又在哪里尼？ 图样图深破。。。如果世间万物都按照我们设想那样，那么就没有那么多的烦恼。在这个问题上，国内众手机厂商有话要说。首先小米祭出了大杀器-[神隐模式](http://www.chinaz.com/mobile/2015/0824/437808.shtml)，听名字就感觉神神叨叨的。

神隐模式在哪里？
![](http://upload.chinaz.com/2015/0824/chinaz_39c52e9dea86298b4809d80ea7ac6b60.jpg)

何为神隐模式？

		神隐，顾名思义是隐藏起来，MIUI 7把一些耗电、耗流量的APP隐藏起来了。被加入在神隐模式列表中的应用，进入后台之后会禁止使用网络，禁止使用GPS，加强版的对齐唤醒等等。
为什么要做神隐模式？

		目前安卓app鱼龙混杂，质量也参差不齐，为了保证续航，就必须从底层着手。MIUI 7对最热门的500款APP进行人工审查，逐一设定专门的省电策略，其中神隐模式就发挥了重要作用，在省电模式下，神隐模式会对后台APP进行一些限制。

那么很明显大家做的app都不在白名单中，经过测试，发现在小米手机熄灭屏幕之后，没有在白名单中的应用就会自动切断网络，停止定位，而且定位sdk的轮询也会在几分钟内自动停止。

那么我们的产品需要随时知道配送员的位置，那么就有了合情合理的稳定打点需求，对应专业配送员来说这并不流氓，到底怎么破解？？？

我们首先发现定位sdk 的文档中有这样一句话

		● 目前手机设备在长时间黑屏或锁屏时CPU会休眠，这导致定位SDK不能正常进行位置更新。若您有锁屏状态下获取位置的需求，您可以应用alarmManager实现1个可叫醒CPU的Timer，定时请求定位。

很快我们实现了一个alarmManager，唤醒了cpu，在大多数的手机上可以正常的打点上传，但是在小米手机上我们收到了网络请求失败的反馈，再测试发现，神隐模式把我们的网络给关掉了。

这个怎么破解？？

经过绞尽脑汁，尝试了守护进程，常驻通知栏都失败之后，发现PowerManager.WakeLock 的PowerManager.SCREEN_DIM_WAKE_LOCK 锁可以把屏幕点亮，屏幕亮了之后竟然就可以联网了，太棒了，小米的判断逻辑竟然真的是屏幕开关。

        mWakelock = pm.newWakeLock(PowerManager.ACQUIRE_CAUSES_WAKEUP | PowerManager.SCREEN_DIM_WAKE_LOCK, "target");
       
     
各种锁的类型对CPU 、屏幕、键盘的影响：

    PARTIAL_WAKE_LOCK:保持CPU 运转，屏幕和键盘灯有可能是关闭的。
    SCREEN_DIM_WAKE_LOCK：保持CPU 运转，允许保持屏幕显示但有可能是灰的，允许关闭键盘灯
    SCREEN_BRIGHT_WAKE_LOCK：保持CPU 运转，允许保持屏幕高亮显示，允许关闭键盘灯
    FULL_WAKE_LOCK：保持CPU 运转，保持屏幕高亮显示，键盘灯也保持亮度
    ACQUIRE_CAUSES_WAKEUP：正常唤醒锁实际上并不打开照明。相反，一旦打开他们会一直仍然保持(例如来世user的activity)。当获得wakelock，这个标志会使屏幕或/和键盘立即打开。一个典型的使用就是可以立即看到那些对用户重要的通知。
    ON_AFTER_RELEASE：设置了这个标志，当wakelock释放时用户activity计时器会被重置，导致照明持续一段时间。如果你在wacklock条件中循环，这个可以用来减少闪烁
 
叹兴了，只找到了这个方案，只能先试试了，在定位之前mWakelock.acquire();上传位置之后mWakelock.release(); 这样屏幕会闪一下，但是想小米手机上也可以稳定的打点上传了，本着先实现功能再优化体验的态度，现在把功能实现了，再想想如何让用户察觉不到屏幕闪一下。以后再更新这个解决方案，这里使用PowerManager.WakeLock 确实可以实现稳定打点的目的。
 