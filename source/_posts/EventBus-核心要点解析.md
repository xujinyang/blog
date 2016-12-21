title: EventBus 核心要点解析
date: 2016-05-24 23:06:30
tags:
---
关注EventBus核心要点
### 使用流程

	register(object)

	eventBus.post(event)
    

举个简单例子

基类Activity
```
public class CommonActivity extends AppCompatActivity {
    protected EMEventBus eventBus;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        eventBus = EMEventBus.getDefault();
        eventBus.register(this);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        eventBus.unregister(this);
    }

    //EventBus至少有一个onEvent方法
    public void onEvent(String defaultEvent) {

    }

}

```
登录页面只订阅一个LoginEvent
```
public class LoginActivity extends CommonActivity {

    public void onEventMainThread(LoginEvent event) {

    }
}

```
当在另一个页面 eventBus.post(new LoginEvent()) 的时候发生了什么
- # register

当LoginActivity页面register的时候

	subscriberMethodFinder.findSubscriberMethods(subscriber.getClass());


 先找出所有LoginActivity中订阅的Event

     {
        LoginActivity.onEventMainThread(me.ele.crowdsource.event.LoginEvent)
        CommonActivity.onEvent(java.lang.String)
     }
然后for循环生成EventBus中第一重要的映射

    private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;

为什么说这个映射重要，因为他对应着一个Event->List订阅这个Event的对象，有了这个映射，当你要post(event)的时候，就可以轻松找到所有响应的订阅者。注意，这个list的类型是[CopyOnWriteArrayList](http://my.oschina.net/jielucky/blog/167198) 

	   CopyOnWriteArrayList是ArrayList 的一个线程安全的变体，其中所有可变操作（add、set等等）都是通过对底层数组进行一次新的复制来实现的。
	 CopyOnWriteArrayList适合使用在读操作远远大于写操作的场景里，比如缓存。发生修改时候做copy，新老版本分离，保证读的高性能，适用于以读为主的情况。

接着生成了EventBus中第二重要的映射
	private final Map<Object, List<Class<?>>> typesBySubscriber;
他对应着注册对象->List监听事件，这个映射关系，unRegistered(),isRegistered()方法中能快速的找到一个activity中所有的onEvent。 

这里EventBus代码中一个奇怪的习惯，就是把一个list先添加到map，然后再给这个list add数据，这个思路有异常人,比如下面这样
```
 List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<Class<?>>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
  subscribedEvents.add(eventType);

```
生成了subscriptionsByEventType和typesBySubscriber 这俩个map，register的任务就结束了。

- # post

试想如果你来实现EventBus，当Post一个event的时候，怎么搞？很简单是根据subscriptionsByEventType找到所有订阅的方法，然后执行。再看EventBus如何实现的，哈哈，也就是这个思路。只是多了个查找Event所有的父类和实现的接口过程.
```
 /** Looks up all Class objects including super classes and interfaces. Should also work for interfaces. */
    private List<Class<?>> lookupAllEventTypes(Class<?> eventClass) {
        synchronized (eventTypesCache) {
            List<Class<?>> eventTypes = eventTypesCache.get(eventClass);
            if (eventTypes == null) {
                eventTypes = new ArrayList<Class<?>>();
                Class<?> clazz = eventClass;
                while (clazz != null) {
                    eventTypes.add(clazz);
                    addInterfaces(eventTypes, clazz.getInterfaces());
                    clazz = clazz.getSuperclass();
                }
                eventTypesCache.put(eventClass, eventTypes);
            }
            return eventTypes;
        }
    }
```

寻找订阅的核心实现在这里
```
 private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
              ···
             postToSubscription(subscription, event, postingState.isMainThread);
               ···
        }
        return false;
    }
```
当真正去响应各个订阅方法的时候，出现了EventBus的核心实现-用

      private final HandlerPoster mainThreadPoster;
      private final BackgroundPoster backgroundPoster;
      private final AsyncPoster asyncPoster;

三个队列来处理所有的方法。关于这里队列的介绍之前[总结过一次](http://blog.csdn.net/mobilexu/article/details/50989962)
有兴趣的可以看这里，类似的实现handle中也有，自己的项目中也可以用。

postToSubscription 函数中会判断订阅者的 ThreadMode，从而决定在什么 Mode 下执行事件响应函数。ThreadMode 共有四类：

- PostThread：默认的 ThreadMode，表示在执行 Post 操作的线程直接调用订阅者的事件响应方法，不论该线程是否为主线程（UI 线程）。当该线程为主线程时，响应方法中不能有耗时操作，否则有卡主线程的风险。适用场景：对于是否在主线程执行无要求，但若 Post 线程为主线程，不能耗时的操作；


- MainThread：在主线程中执行响应方法。如果发布线程就是主线程，则直接调用订阅者的事件响应方法，否则通过主线程的 Handler 发送消息在主线程中处理——调用订阅者的事件响应函数。显然，MainThread类的方法也不能有耗时操作，以避免卡主线程。适用场景：必须在主线程执行的操作；


- BackgroundThread：在后台线程中执行响应方法。如果发布线程不是主线程，则直接调用订阅者的事件响应函数，否则启动唯一的后台线程去处理。由于后台线程是唯一的，当事件超过一个的时候，它们会被放在队列中依次执行，因此该类响应方法虽然没有PostThread类和MainThread类方法对性能敏感，但最好不要有重度耗时的操作或太频繁的轻度耗时操作，以造成其他操作等待。适用场景：操作轻微耗时且不会过于频繁，即一般的耗时操作都可以放在这里；


- Async：不论发布线程是否为主线程，都使用一个空闲线程来处理。和BackgroundThread不同的是，Async类的所有线程是相互独立的，因此不会出现卡线程的问题。适用场景：长耗时操作，例如网络访问。

post方法中关于线程的处理也耐人寻味，尤其是这个currentPostingThreadState
```
    private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
        @Override
        protected PostingThreadState initialValue() {
            return new PostingThreadState();
        }
    };

```
每次post可能在不同的线程，这个时候使用ThreadLocal使得各线程能够保持各自独立的一个对象，从源码可以看出
```
	public T get() { 
        Thread t = Thread.currentThread(); 
        ThreadLocalMap map = getMap(t); 
        if (map != null) { 
            ThreadLocalMap.Entry e = map.getEntry(this); 
            if (e != null) 
                return (T)e.value; 
        } 
        return setInitialValue(); 
    }

```
这说明ThreadLocal确实只有一个变量，但是它内部包含一个map，针对每个thread保留一个entry，如果对应的thread不存在则会调用initialValue。
再回到EventBus
```
       final static class PostingThreadState {
            final List<Object> eventQueue = new ArrayList<Object>();
            boolean isPosting;
            boolean isMainThread;
            Subscription subscription;
            Object event;
            boolean canceled;
        }
```
每个线程维护一个eventQueue,每次post就循环的去发送
```
 	PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);
 	while (!eventQueue.isEmpty()) {
         postSingleEvent(eventQueue.remove(0), postingState);
       }

```


- # unregister

unregister思路很简单，就是清除typesBySubscriber里面这个Activity的记录和subscriptionsByEventType中对应的记录，但是这里有个删除list中元素的技巧很有趣。因为之前兆轩刚在上面踩过坑,看EventBus如何删除subscriptionsByEventType中一个event对应的订阅list中某个记录。
```
    private void unubscribeByEventType(Object subscriber, Class<?> eventType) {
        List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions != null) {
            int size = subscriptions.size();
            for (int i = 0; i < size; i++) {
                Subscription subscription = subscriptions.get(i);
                if (subscription.subscriber == subscriber) {
                    subscription.active = false;
                    subscriptions.remove(i);
                    i--;
                    size--;
                }
            }
        }
    }

```
重点就是i--和size--,当命中的时候通过--让游标回退一位，size也减一，这样就可以让剩下的元素也正常循环完成。


到此为止，EventBus我认为比较有意思的地方就都列出来了，还是学到了蛮多的东西。












