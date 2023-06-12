---
layout: article
title: 'Android 源码分析之Handler'
tags: Android handler Java
---

## 简介

  **Handler是线程间通讯的载体,它通过异步操作将Message事务入队到目标线程的消息队列中,最终目标线程通过出队列的方式处理传递过来的Message事务,从而实现线程间通讯**  
  **这里我们将生产Message事务的线程称为生产者线程,处理Message事务的线程称为消费者线程**

## 源码流程分析

![UML_class.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/273500327f554faeaa35592c91cd611b~tplv-k3u1fbpfcp-watermark.image?)
**从类图关系中,我们可以看到**  
**1.Looper构建了消息队列MessageQueue**  
**2.Message的成员target关联Handler**  
**3.Handler的成员mQueue关联MessageQueue,成员mLooper关联Looper**  
**4.MessageQueue的成员mMessages关联Message**

### 消费者线程

  **首先我们创建消费者线程并启动消息循环**

这里我们借助**HandlerThread类**

```
HandlerThread consumerThread = new HandlerThread("Consumer");
consumerThread.start();
```
我们分析下HandlerThread的类实现  
首先查看继承关系

```
public class HandlerThread extends Thread
```
由继承关系中,可以得知**该类属于线程类**

然后查看**线程类的执行体run函数**
```
public void run() {
......
    Looper.prepare();    //初始化当前线程的Looper
    Looper.loop();       //启动消息循环
......
}
```
到关键点了,接下来我们分别分析**prepare**和**loop**中干了什么

#### 1. Looper.prepare
```
public static void prepare() {
    prepare(true);
}
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {  /* 如果消费者线程已有Looper绑定了,则抛出异常 */
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed)); /* 创建Looper,并绑定到消费者线程 */
}
```
sThreadLocal与C++的TLS功能一样,具有**一键多值属性:既同一个对象保存不同线程的局部变量**

**Looper构造函数**
```
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed); //创建消息队列
    mThread = Thread.currentThread();       //记录消费者线程
}
```

由于此篇文章只说Java实现部分,而MessageQueue类涉及到native部分,此部分会在[Native篇](https://sanyusily.github.io/2023/02/11/looper-native-read-in.html "Native篇")中详细讲解,因此这里不具体进行分析,**这里读者只需知道创建了消息队列MessageQueue即可**

##### 总结

  **1.消费者线程创建了Looper以及消息队列MessageQueue**  
  **2.Looper与消费者线程通过sThreadLocal进行唯一绑定**

#### 2. Looper.loop

```
public static void loop() {
......
    for (;;) {
        Message msg = queue.next(); //消息队列出队得到Message事务,见2.1
        if (msg == null) {
            return;
    }
    msg.target.dispatchMessage(msg);//执行具体事务,见2.2
......
}
```
##### 2.1 MessageQueue.next

```
Message next() {
......
    int pendingIdleHandlerCount = -1;
    int nextPollTimeoutMillis = 0;
    for (;;) {
        /* native方法,处理Looper(Native)事务
         * 如无事务处理则会让消费者线程进入超时休眠状态,nextPollTimeoutMillis为超时时间
         */
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
            /* 如果设置了同步屏障,则只允许异步消息出队 */
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {          
            /* 检验事务 */
                if (now < msg.when) {	
                /* 未达到目标时间,则设置nextPollTimeoutMillis超时时间= (目标时间 - now) */
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {    
                /* 达到出队条件 */
                    mBlocked = false;	/* 设置队列阻塞状态为false */
                    
                    /* 此处将就绪的Message事务从消息队列中出队并返回,见出队流程图 */
                    if (prevMsg != null) {		
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
            /* 如果队列为空,则设置nextPollTimeoutMillis超时时间 = -1,既无限时间休眠等待事件唤醒 */
                nextPollTimeoutMillis = -1;
            }

            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
            /* 第一次空闲,并且无就绪Message事务时,查询空闲Handler事务数量 */
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) { 
            /* 如果既没出队的Message事务又没空闲Handler事务,则设置队列阻塞状态为true,并跳转下一轮For循环 */
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) { 
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        for (int i = 0; i < pendingIdleHandlerCount; i++) { 
        /* 遍历空闲Handler事务 */
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; 

            boolean keep = false;
            try {
                keep = idler.queueIdle();   /* 执行空闲Handler事务 */
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {    /* 如果返回值为false,则将该空闲Handler从mIdleHandlers中移除 */
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        pendingIdleHandlerCount = 0;    /* 设置pendingIdleHandlerCount = 0,代表非第一次空闲 */

        /* 设置nextPollTimeoutMillis = 0,跳转下一轮For循环,在nativePollOnce中会立即返回
         * 以防止在执行空闲Handler事务的时候,有Message事务进来
         */
        nextPollTimeoutMillis = 0;      
                                                       
    }
......
}
```

![MessageQueue_dequeue.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f7e0093a3193484d85629564e0a0b0fe~tplv-k3u1fbpfcp-watermark.image?)

**出队列流程图**

##### 2.2 Handler.dispatchMessage
```
public void dispatchMessage(Message msg) {
    if (msg.callback != null) { 
    /*如果message设置了callback,则直接执行callback的run方法并返回 注1 */
        handleCallback(msg);
    } else {
        if (mCallback != null) { 
        /* 如果Handler设置了callback,则直接执行callback的handleMessage方法并返回 注2 */
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg); /* 如果以上条件都不满足,则调用自身handleMessage方法 */
    }
}

private static void handleCallback(Message message) {
	message.callback.run();
}
```
**注1:** 此处callback一般在**Handler.post(Runnable)** 中设置  
**注2:** 此处callback在**Handler构造函数**中设置
```
public Handler(Looper looper, Callback callback)
```

##### 总结

  **1.调用Looper.loop使消费者线程不断循环从消息队列MessageQueue中拿到Message事务并进行处理**  
  **2.如果消息队列MessageQueue为空,则遍历执行空闲Handler事务**  
  **3.如果消息队列MessageQueue以及空闲Handler都为空,则消费者线程进行休眠等待**  
   **生产者线程将Message事务入队列并唤醒消费者线程**  
  **4.Message事务处理优先级: Message.callback > Handler.mCallback > Self(handleMessage)**

### 生产者线程

  **首先我们创建生产者线程并通过Handler的sendEmptyMessage接口将事务Message入队到消费者线程的消息队列MessageQueue中**
  
```
new Thread("Producer") {
    @Override
    public void run() {
        /* 创建Handler并关联消费者线程的消息队列MessageQueue */
        Handler producerHandler = new Handler(consumerThread.getLooper()) {
            @Override
            public void handleMessage(Message msg) {
            }
        };
        /* 将Message事务入队到关联的消息队列,最终调用MessageQueue.enqueueMessage */
        producerHandler.sendEmptyMessage(0);  
    }
};
```

**Handler构造函数**

```
public Handler(Looper looper) {
    this(looper, null, false);
}
public Handler(Looper looper, Callback callback, boolean async) {
......
    mLooper = looper;           //成员mLooper关联消费者线程的looper
    mQueue = looper.mQueue;	//成员mQueue关联消费者线程的消息队列MessageQueue
......
}
```
#### MessageQueue.enqueueMessage

```
boolean enqueueMessage(Message msg, long when) {
......
    if (p == null || when == 0 || when < p.when) {	
    /* 检查入队消息是否为新的头节点mMessages 
     * 1. MessageQueue消息队列头节点mMessages为空
     * 2. 入队消息触发时间为0 (通过Handler.sendMessageAtFrontOfQueue设置)
     * 3. 入队消息的触发时间早于MessageQueue消息队列头节点mMessages的触发时间
     * 当满足以上任一条件时,则认为入队消息为新的头节点mMessages
     */
        msg.next = p;		/* 入队消息next指向原头节点 */
        mMessages = msg;	/* 入队消息成为新的头节点 */
        needWake = mBlocked; /* 如果队列为阻塞状态,则需要唤醒消费者线程 */
    } else {	
    /* 入队消息非头节点
     * 通常入队消息非头节点时不需要唤醒消费者线程,除非同时满足以下两个条件
     * 1. 消息队列处于同步屏障模式和阻塞状态
     * 2. 入队的是异步消息并且触发时间早于队列中其他异步消息
     */
        needWake = mBlocked && p.target == null && msg.isAsynchronous();	
        Message prev;
        /* 根据入队消息的触发时间决定插入队列的位置,见入队列流程图 */
        for (;;) {
            prev = p;
            p = p.next;
            if (p == null || when < p.when) {
                break;
            }
            if (needWake && p.isAsynchronous()) {
                needWake = false;
            }
        }
        msg.next = p;
        prev.next = msg;
    }

    if (needWake) {
        nativeWake(mPtr);   /* native方法,唤醒消费者线程 */
    }
    return true;
......
}
```


![MessageQueue_enqueue.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/70c73b6a285a4796ac63369f6389c198~tplv-k3u1fbpfcp-watermark.image?)

**入队列流程图**

#### 总结

  **1.生产者线程创建Handler,该Handler关联消费者线程的消息队列MessageQueue**  
  **2.生产者线程通过该Handler发送Message,将Message事务入队到关联的消息队列MessageQueue中,**  
   **等待消费者线程进行处理**

## 全文总结


![handler_loop.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bed5214a50ea46c09fab62c6117a71a9~tplv-k3u1fbpfcp-watermark.image?)

**1.消费者线程启动消息循环,不断处理消息队列中的事务**  
**2.生产者线程通过调用Handler的sendMessage接口,将Message事务入队到消息队列MessageQueue中**  
**3.消费者线程从消息队列MessageQueue中取出Message事务,进行处理**

