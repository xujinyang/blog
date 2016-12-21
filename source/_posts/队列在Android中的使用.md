title: 队列在Android中的使用
date: 2016-03-26 22:36:58
tags:
---

先科普一下队列：
>队列是一种特殊的线性表，特殊之处在于它只允许在表的前端（front）进行删除操作，而在表的后端（rear）进行插入操作，和栈一样，队列是一种操作受限制的线性表。进行插入操作的端称为队尾，进行删除操作的端称为队头。队列中没有元素时，称为空队列。
队列的数据元素又称为队列元素。在队列中插入一个队列元素称为入队，从队列中删除一个队列元素成为出队。因为队列只允许在一段插入，在另一端删除，所以只有最早进入队列的元素才能最先从队列中删除，故队列又称为先进先出（FIFO—first in first out）线性表。[1] 

说到队列就会想到链表,那么他们的关系是怎么样的？

>链表是一种数据结构，而队列是一种抽象的概念，就像栈一样。
	船是一个比较抽象的概念，具体实现有木船、铁船等等。队列好比是船，链表好比是造船的材料。
	队列可以用链表实现，也可以用动态数组实现，这个抽象的概念可以用各种具体的数据结构实现。

   在队列的形成过程中，可以利用线性链表的原理，来生成一个队列。基于链表的队列，要动态创建和删除节点，效率较低，但是可以动态增长。
   
  队列采用的FIFO(first in first out)，新元素（等待进入队列的元素）总是被插入到链表的尾部，而读取的时候总是从链表的头部开始读取。每次读取一个元素，释放一个元素。所谓的动态创建，动态释放。因而也不存在溢出等问题。由于链表由结构体间接而成，遍历也方便。
  
下面看一个[队列的链表实现demo](http://blog.sina.com.cn/s/blog_81720c020100yqlm.html)
```
class QueueNode {  
    Object data; // 节点存储的数据  
    QueueNode next; // 指向下个节点的指针  
 
    public QueueNode() {  
        this(null, null);  
    }  
 
    public QueueNode(Object data) {  
        this(data, null);  
    }  
 
    public QueueNode(Object data, QueueNode next) {  
        this.data = data;  
        this.next = next;  
    }  
}  

public class QueueLinked {  
    QueueNode front; // 队首指针  
    QueueNode rear;  // 队尾指针  
 
    public QueueLinked() {  
        this.rear = null;  
        this.front = null;  
    }   //将一个对象追加到队列的尾部
    public void enqueue(Object obj) {  
        //如果队列是空的  
        if (rear == null && front == null) {  
            rear = new QueueNode(obj);  
            front = rear;  
        } else {  
            QueueNode node = new QueueNode(obj);  
            rear.next = node;  
            rear = rear.next;  
        }  
    }     
     //队首对象出队
     //return 出队的对象，队列空时返回null      
    public Object dequeue() {  
        //如果队列空  
        if (front == null) {  
            return null;  
        }  
        //如果队列中只剩下一个对象  
        if (front == rear && rear != null) {  
            QueueNode node = front;  
            rear = null;  
            front = null;  
            return node.data;  
        }  
        Object obj = front.data;  
        front = front.next;  
        return obj;  
    }   
    public static void main(String[] args) {  
        QueueLinked q = new QueueLinked();  
        q.enqueue("张三");  
        q.enqueue("李斯");  
        q.enqueue("赵五");  
        q.enqueue("王一");  
        for (int i = 0; i < 4; i++) {  
            System.out.println(q.dequeue());  
        }  
    }  
}
```
队列善于处理插入和弹出的操作，再简化点说，队列适合用于流水线式的任务调度，比如大家熟悉的Handle消息队列处理，EventBus中的event分发，Afinal中Http request请求发送,下面就从源码角度看看队列的应用。
### Handle消息队列
Handle 异步处理中用来存放Message对象的数据结构，按照“先进先出”的原则存放消息。存放并非实际意义的保存，而是将Message对象以链表的方式串联起来的。MessageQueue对象不需要我们自己创建，而是有Looper对象对其进行管理，一个线程最多只可以拥有一个MessageQueue。在Lopper方法中：出现了一个死循环，从队列中不断的取出message，执行msg.target.dispatchMessage(msg);

```
public static final void loop() {  
  
        Looper me = myLooper();  
  
        MessageQueue queue = me.mQueue;  
  
        while (true) {  
  
            Message msg = queue.next(); // might block  
  
            //if (!me.mRun) {  
  
            //    break;  
  
            //}  
  
            if (msg != null) {  
  
                if (msg.target == null) {  
  
                    // No target is a magic identifier for the quit message.  
  
                    return;  
  
                }  
  
                if (me.mLogging!= null) me.mLogging.println(  
  
                        ">>>>> Dispatching to " + msg.target + " "  
  
                        + msg.callback + ": " + msg.what  
  
                        );  
  
                msg.target.dispatchMessage(msg);  
  
                if (me.mLogging!= null) me.mLogging.println(  
  
                        "<<<<< Finished to    " + msg.target + " "  
  
                        + msg.callback);  
  
                msg.recycle();  
  
            }  
  
        }  
  
    }  
```
当添加Message的时候，调用MessageQueue 的enqueueMessage(msg, uptimeMillis)方法

```
final boolean enqueueMessage(Message msg, long when) {  
  
        if (msg.when != 0) {  
  
            throw new AndroidRuntimeException(msg  
  
                    + " This message is already in use.");  
  
        }  
  
        if (msg.target == null && !mQuitAllowed) {  
  
            throw new RuntimeException("Main thread not allowed to quit");  
  
        }  
  
        synchronized (this) {  
  
            if (mQuiting) {  
  
                RuntimeException e = new RuntimeException(  
  
                    msg.target + " sending message to a Handler on a dead thread");  
  
                Log.w("MessageQueue", e.getMessage(), e);  
  
                return false;  
  
            } else if (msg.target == null) {  
  
                mQuiting = true;  
  
            }  
  
   
  
            msg.when = when;  
  
            //Log.d("MessageQueue", "Enqueing: " + msg);  
  
            Message p = mMessages;  
  
            if (p == null || when == 0 || when < p.when) {  
  
                msg.next = p;  
  
                mMessages = msg;  
  
                this.notify();  
  
            } else {  
  
                Message prev = null;  
  
                while (p != null && p.when <= when) {  
  
                    prev = p;  
  
                    p = p.next;  
  
                }  
  
                msg.next = prev.next;  
  
                prev.next = msg;  
  
                this.notify();  
  
            }  
  
        }  
  
        return true;  
  
    }  
```
这里可以看到添加一个message的时候，尾指针指向的改变。
### EventBus中队列的使用
在eventbus 中队列节点的结构

```

final class PendingPost {
    private final static List<PendingPost> pendingPostPool = new ArrayList<PendingPost>();

    Object event;
    Subscription subscription;
    PendingPost next;

    private PendingPost(Object event, Subscription subscription) {
        this.event = event;
        this.subscription = subscription;
    }
}
```
队列的实现

```
package de.greenrobot.event;

final class PendingPostQueue {
    private PendingPost head;
    private PendingPost tail;

    synchronized void enqueue(PendingPost pendingPost) {
        if (pendingPost == null) {
            throw new NullPointerException("null cannot be enqueued");
        }
        if (tail != null) {
            tail.next = pendingPost;
            tail = pendingPost;
        } else if (head == null) {
            head = tail = pendingPost;
        } else {
            throw new IllegalStateException("Head present, but no tail");
        }
        notifyAll();
    }

    synchronized PendingPost poll() {
        PendingPost pendingPost = head;
        if (head != null) {
            head = head.next;
            if (head == null) {
                tail = null;
            }
        }
        return pendingPost;
    }

    synchronized PendingPost poll(int maxMillisToWait) throws InterruptedException {
        if (head == null) {
            wait(maxMillisToWait);
        }
        return poll();
    }

}

```
event分发的逻辑

```
final class HandlerPoster extends Handler {

    private final PendingPostQueue queue;
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
    private boolean handlerActive;

    HandlerPoster(EventBus eventBus, Looper looper, int maxMillisInsideHandleMessage) {
        super(looper);
        this.eventBus = eventBus;
        this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
        queue = new PendingPostQueue();
    }

    void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);
            if (!handlerActive) {
                handlerActive = true;
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }

    @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            while (true) {
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
}
```
同样有个死循环，在不断的取出event，通过eventBus.invokeSubscriber(pendingPost);分发出去

看了上面的源码很容易就会发现，队列很方便的就实现的一个工作池的模型，一边在往池子里面放任务，一边在不断的从池子里面取出任务，那么在我们的实际项目中应该如何应用队列尼？

不久前，我在实现语音合成功能的时候，就发现如果用户连续的触发了两句话的发声，就会出现一起朗读的现象，那么就需要实现，不管同时触发多少个发声任务，都让他们按照先进先出的原则，先触发的先读，依次朗读，这样才是合理的，那么我就使用队列实现了一个线程池，线程池？对，说到线程池，它的内部实现也是利用队列，不信请看：ThreadPoolExecutor 的内部变量    

```
private final BlockingQueue<Runnable> workQueue;
|
|
->public interface BlockingQueue<E> extends Queue<E>
```
也是一个队列,使用的时候同样是个死循环

```
 final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated();
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }

```
想到了这里，我忽然发现自己好像掌握了一门异步任务处理的大杀器。

看见多任务，不怕不怕啦，队列搞死他，不怕不怕不怕啦。





