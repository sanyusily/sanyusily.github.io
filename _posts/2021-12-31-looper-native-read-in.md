---
layout: article
title: 'Android P源码分析之Looper(Native)'
tags: Looper Android AOSP
---


# 一、Looper(Native)

## [序言](https://zhenkunhuang.github.io/2019/05/18/android-Looper-Native/#序言 "序言")序言

  **学习此篇前,请确认掌握了[eventfd](https://zhenkunhuang.github.io/2019/04/20/linux-eventfd/ "eventfd")以及[epoll](https://zhenkunhuang.github.io/2019/04/23/linux-epoll/ "epoll")的使用**

## [](https://zhenkunhuang.github.io/2019/05/18/android-Looper-Native/#简介 "简介")简介

  在[Android P源码分析之Handler(JAVA)篇](https://zhenkunhuang.github.io/2019/05/05/android-handler-java/ "Android P源码分析之Handler(JAVA)篇")中,我们分析了Java层的消息循环处理流程,其中Looper扮演着不断从消息队列中取出消息进行分发处理的重要角色.而在Native层中,也存在着相同作用的Looper.

## [](https://zhenkunhuang.github.io/2019/05/18/android-Looper-Native/#示例 "示例")示例

**类定义**

~~~
class LocalHandler : public MessageHandler, public LooperCallback{
/* MessageHandler为消息事务处理类, LooperCallback为Fd事务处理类 */
public:
    virtual void handleMessage(const Message& message) {
        /* 消息事务处理函数,在消费者线程consumerThread中执行 */
        printf("tid : %d, receive message : %d\n", gettid(), message.what);
    }
    
    virtual int handleEvent(int fd, int events, void* data __unused) {
        /* Fd事务处理函数,在消费者线程consumerThread中执行 */
        char recvMsg[128] = { 0 };

        if (events & Looper::EVENT_INPUT) {
        	if (read(fd, recvMsg, sizeof(recvMsg) / sizeof(*recvMsg)) < 0) {
        		printf("tid : %d, read fail, reason : %s\n",gettid(), strerror(errno));
        		return 0;
        	}
        	printf("tid : %d, receive event : %s\n",gettid(), recvMsg);
        }
        return 1;
    }
};
~~~
**消费者线程执行体**

```
void *consumerThread_func(void *arg __unused) {  
	looper = Looper::prepare(0); /* 创建线程唯一的Looper对象并绑定到消费者线程,见1.1 */

	sem_post(&sem); /* 通知生产者线程producerThread Looper就绪 */
	
	while (consumeThreadExit == 0) {
        /* 循环处理Looper事务,见1.4 */
		looper -> pollOnce(-1);
	}
	return NULL;
}
```

**生产者线程执行体**  

```
void *producerThread_func(void *arg __unused) {  
    char msg[128] = { 0 };
    int fd[2];
    /* 创建事务处理者handler */
    sp<LocalHandler> handler = new LocalHandler();
    /* 创建pipe,其中fd[0]为读取端,fd[1]为写入端 */
    pipe(fd);

    /* 等待消费者线程初始化Looper */
    sem_wait(&sem);
    printf("tid : %d sem_wait looper prepare success\n",gettid());
    /* 发送消息事务Message,唤醒消费者线程consumerThread处理消息事务 */
    looper -> sendMessage(handler, Message());

    /* 添加Fd事务请求:Fd为pipe读取端fd[0],事务类型为事件可读 */
    looper -> addFd(fd[0], 0, Looper::EVENT_INPUT, handler, NULL);
    /* 从终端读取字符串,并写到pipe的写入端fd[1] 
     * 此时pipe的poll状态变为可读,从而唤醒消费者线程consumerThread处理Fd事务
     */
    fgets(msg, sizeof(msg) / sizeof(*msg), stdin);
    write(fd[1], msg, strlen(msg) + 1);

    close(fd[0]);
    close(fd[1]);
    return NULL;
}
```

**运行结果**  
```
tid : 8234 sem_wait looper prepare success /* 生产者线程 */
tid : 8235, receive message : 0            /* 消费者线程处理消息事务 */
Hello World     <---终端输入
tid : 8235, receive event : Hello World    /* 消费者线程处理Fd事务 */
```

**接下来我们将从消费者线程和生产者线程角度分析示例流程**

## [](https://zhenkunhuang.github.io/2019/05/18/android-Looper-Native/#源码分析 "源码分析")源码分析

### [](https://zhenkunhuang.github.io/2019/05/18/android-Looper-Native/#消费者线程 "消费者线程")消费者线程

  **我们首先分析Looper(Native)的初始化函数prepare**

#### [](https://zhenkunhuang.github.io/2019/05/18/android-Looper-Native/#1-1-prepare "1.1 prepare")1.1 prepare

```
sp<Looper> Looper::prepare(int opts) {
......
    /* 获取当前消费者线程线程绑定的Looper对象
     * 由于这里首次调用prepare,还未绑定Looper,因此返回空 
     */
    sp<Looper> looper = Looper::getForThread();
    if (looper == NULL) {
    /* 创建Looper对象(见1.2),然后绑定到当前消费者线程中 */
        looper = new Looper(allowNonCallbacks);
        Looper::setForThread(looper);
    }

    return looper;
......
}
```

#### 1.2 Looper构造函数
```
Looper::Looper(bool allowNonCallbacks) {
......
    /* 创建eventfd文件描述符mWakeEventFd,用于唤醒处于epoll_wait休眠状态下的消费者线程 */
    mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);

    AutoMutex _l(mLock);
    /* 见1.3 */
    rebuildEpollLocked();
......
}
```

#### 1.3 rebuildEpollLocked
```
void Looper::rebuildEpollLocked() {
......
    /* 创建epoll文件描述符,用于监听多路IO状态 */
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);

    /* epoll添加mWakeEventFd监听(可读) */
    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); 
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeEventFd;
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);
......
}
```

**小结**  
  **1.创建Looper对象并绑定到消费者线程**  
  **2.创建eventfd文件描述符mWakeEventFd**  
  **3.创建epoll文件描述符mEpollFd,并将mWakeEventFd添加到监听Fd事项中**

**接下来分析Looper(Native)事务处理函数pollOnce**

#### 1.4 pollOnce

```
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
......
    int result = 0;
    for (;;) {
	/* 当result不为0时,返回 */
        if (result != 0) {
            ......
            return result;
        }
        /* 见1.5 */
        result = pollInner(timeoutMillis);
    }
......
}
```
#### 1.5 pollInner
```
int Looper::pollInner(int timeoutMillis) {
......
    /* 校准epoll超时时间,取timeoutMillis和(mNextMessageUptime - now)之间大于0且较小的那个 */
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            timeoutMillis = messageTimeoutMillis;
        }
    }

    int result = POLL_WAKE;
    /* 清除Fd就绪事务队列 */
    mResponses.clear();
    mResponseIndex = 0;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    /* 消费者线程调用epol_wait超时休眠,等待监听的Fd事项就绪 */
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    /* 如果是超时唤醒,则直接跳过就绪Fd事项数组遍历 */
    if (eventCount == 0) {
        result = POLL_TIMEOUT;
        goto Done;
    }
    /* 遍历就绪Fd事项数组eventItems */
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd) {
        /* 当mWakeEventFd在就绪Fd事项数组中,需调用awoken函数读取mWakeEventFd的内容,
         * 防止下次调用epoll_wait时,mWakeEventFd的poll状态仍为可读导致直接返回(epoll水平触发特性) 
         */
            if (epollEvents & EPOLLIN) {
                awoken();
            } 
        } else {
        /* 以文件描述符fd为索引值取出对应的事务请求Request,用于构建Fd就绪事务Response对象,
         * 然后将Response入队到Fd就绪事务队列mResponses中等待处理 
         */
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                pushResponse(events, mRequests.valueAt(requestIndex));
            }
        }
    }
Done: ;
    /* 遍历消息事务队列mMessageEnvelopes */
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
        /* 当前时间 > 消息事务messageEnvelope分发时间,
         * 消息事务messageEnvelope出队并执行事务处理(handleMessage)
         */
            { 
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                handler->handleMessage(message);
            } 
        } else {
        /* 未到消息事务分发时间,
         * 设置mNextMessageUptime为消息事务分发时间uptime,并跳出遍历 
         */
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }
    /* 遍历Fd就绪事务队列mResponses */
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
        /* Fd事务请求的标识ident为POLL_CALLBACK,
         * 处理Fd就绪事务response(handleEvent)
         * Fd事务处理函数handleEvent返回值为0时,epoll将移除对Fd事务请求request的监听
         */
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                removeFd(fd, response.request.seq);
            }
        }
    }
    return result;
......
}
```
**小结**  
  **1.消费者线程调用epoll_wait检测是否有事务就绪,如果无事务就绪则会进入休眠,等待事务就绪时唤醒**  
  **2.当Fd事项就绪时,消费者线程被唤醒.先处理消息事务MessageEnvelope(handleMessage),最后处理Fd就绪事务Response(handleEvent,返回值为0时epoll将移除对Fd事务请求request的监听)**
  
### 生产者线程

**我们首先分析下发送消息事务sendMessage**

#### [](https://zhenkunhuang.github.io/2019/05/18/android-Looper-Native/#2-1-sendMessage "2.1 sendMessage")2.1 sendMessage
```
void Looper::sendMessage(const sp<MessageHandler>& handler, const Message& message) {
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    /* 见2.2 */
    sendMessageAtTime(now, handler, message);
}
```
#### 2.2 sendMessageAtTime

```
void Looper::sendMessageAtTime(nsecs_t uptime, const sp<MessageHandler>& handler,
        const Message& message) {
......
    size_t i = 0;
	/* 构建消息事务MessageEnvelope并入队到消息事务队列mMessageEnvelopes中(分发时间uptime升序排序) */
    size_t messageCount = mMessageEnvelopes.size();
    while (i < messageCount && uptime >= mMessageEnvelopes.itemAt(i).uptime) {
        i += 1;
    }
    MessageEnvelope messageEnvelope(uptime, handler, message);
    mMessageEnvelopes.insertAt(messageEnvelope, i, 1);

    /* 如果入队的位置为头部,则立即唤醒消费者线程进行处理 */
    if (i == 0) {
        wake(); /* 见2.3 */
    }
......
}
```

#### 2.3 wake
```
void Looper::wake() {
......
    /* 当对mWakeEventFd写入内容时, mWakeEventFd的poll状态变为可读就绪状态
     * 由于epoll监听mWakeEventFd,因此当mWakeEventFd就绪时,消费者线程会从epoll_wait的休眠中被唤醒
     */
    uint64_t inc = 1;
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            LOG_ALWAYS_FATAL("Could not write wake signal to fd %d: %s",
                    mWakeEventFd, strerror(errno));
        }
    }
......
}
```
**小结**  
  **1.生产者线程构建消息事务messageEnvelope,并入队到消费者线程的消息事务队列mMessageEnvelopes中(分发时间uptime升序排序)**  
  **2.如果入队位置为头节点,则调用wake唤醒消费者线程处理事务**

**接下来分析添加Fd事务请求addfd**

#### [](https://zhenkunhuang.github.io/2019/05/18/android-Looper-Native/#2-4-addfd "2.4 addfd")2.4 addfd
```
int Looper::addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data) {
......

    /* 构建Fd事务请求request */
    Request request;
    request.fd = fd;
    request.ident = ident;
    request.events = events;
    request.seq = mNextRequestSeq++;
    request.callback = callback;
    request.data = data;

    ssize_t requestIndex = mRequests.indexOfKey(fd);
    if (requestIndex < 0) {
    /* 首次添加Fd事务请求
     * 1.epoll添加该fd监听
     * 2.将request加入到Fd事务请求队列mRequests中
     */
        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, & eventItem);
        mRequests.add(fd, request);
    } else {
    /* 非首次添加Fd事务请求
     * 1. epoll更新该fd监听事项
     * 2. 更新Fd事务请求request
     */
        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_MOD, fd, & eventItem);
        mRequests.replaceValueAt(requestIndex, request);
    }
    return 1;
......
}
```

**小结**  
  **1.生产者线程构建Fd事务请求request,并入队到消费者线程的Fd事务请求队列mRequests中**  
  **2.消费者线程mEpollFd添加fd事项监听**
  
![Looper_UML.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af2977869ccf4f319c291411db207f5d~tplv-k3u1fbpfcp-watermark.image?)
  
![Looper_native.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9af17e01ce864b6da6e8578b422887e7~tplv-k3u1fbpfcp-watermark.image?)
  **1.消费者线程创建Looper(Native),并调用pollOnce处理事务,当无事务处理时,epoll_wait会让消费者线程进入休眠状态**  
  **2.生产者线程传入事务到消费者线程的事务队列中,并唤醒消费者线程进行处理**

# 二、MessageQueue

## [](https://zhenkunhuang.github.io/2019/05/18/android-Looper-Native/#简介-1 "简介")简介

  在[Android P源码分析之Handler(JAVA)篇](https://zhenkunhuang.github.io/2019/05/05/android-handler-java/ "Android P源码分析之Handler(JAVA)篇")中,我们分析了MessageQueue类的取出消息next流程以及入队消息enqueueMessage流程  
  以上两个流程中,都有调用Native方法(nativePollOnce和nativeWake),接下来我们将对这两个Native方法进行分析

## [](https://zhenkunhuang.github.io/2019/05/18/android-Looper-Native/#源码分析-1 "源码分析")源码分析

  **我们首先分析MessageQueue的构造函数**

### [](https://zhenkunhuang.github.io/2019/05/18/android-Looper-Native/#3-1-MessageQueue构造函数 "3.1 MessageQueue构造函数")3.1 MessageQueue构造函数
```
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    /* Native函数,构建NativeMessageQueue对象并返回对象地址,见3.2 */
    mPtr = nativeInit(); 
}
```

### 3.2 nativeInit
```
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
......
    /* 构建NativeMessageQueue对象,构造函数见3.3 */
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();

    /* 返回nativeMessageQueue对象地址 */
    return reinterpret_cast<jlong>(nativeMessageQueue);
......
}
```
### 3.3 NativeMessageQueue构造函数
```
NativeMessageQueue::NativeMessageQueue() {
......
    /* 获取当前消费者线程线程绑定的Looper对象
     * 由于这里首次构建NativeMessageQueue,还未绑定Looper,因此返回空 
     */
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
    /* 构建Looper(Native)对象并绑定到消费者线程,构造函数见1.2 */
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
......
}
```

**小结**  
  **1.MessageQueue(Java)构建了NativeMessageQueue(Native)对象,并记录该对象地址**  
  **2.NativeMessageQueue(Native)构建Looper(Native)对象,并绑定到消费者线程**

**接下来分析MessageQueue取出消息next函数中使用的nativePollOnce方法**

### [](https://zhenkunhuang.github.io/2019/05/18/android-Looper-Native/#3-4-next "3.4 next")3.4 next

```
Message next() {
......
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
    ......
    /* native方法,处理Looper(Native)事务
     * 如无事务处理则会让消费者线程进入超时休眠状态,nextPollTimeoutMillis为超时时间,见3.5
     */
        nativePollOnce(ptr, nextPollTimeoutMillis);	
  
        /* 以下为取消息Message流程,这里不再重复贴出 */
    ......
    }
......
}
```
### 3.5 nativePollOnce
```
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    /* 指针转换,得到从nativeInit构建的NativeMessageQueue对象 */
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    /* 最终调用的是消费者线程Looper(Native)的pollOnce进行事务处理,见1.4 */
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
```
**小结**  
  **1.Looper(Java)启动消息循环,先处理Looper(Native)事务,然后再处理Looper(Java)事务**  
  **2.Looper(Native)和Looper(Java)均无事务处理时,消费者线程会进入超时休眠状态,等待事务就绪时唤醒**

**最后分析入队消息enqueueMessage函数中使用的nativeWake方法**

### [](https://zhenkunhuang.github.io/2019/05/18/android-Looper-Native/#3-6-enqueueMessage "3.6 enqueueMessage")3.6 enqueueMessage

```
boolean enqueueMessage(Message msg, long when) {
......
    /* 以上为消息事务Message入队列流程,这里不再重复贴出 */
    if (needWake) {
        /* native方法,唤醒消费者线程,见3.7 */
        nativeWake(mPtr);
    }
......
}
```
### 3.7 nativeWake
```
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    /* 指针转换,得到从nativeInit构建的NativeMessageQueue对象 */
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    /* 唤醒处于epoll_wait休眠状态下的消费者线程,见2.3 */
    nativeMessageQueue->wake();
}
```
**小结**  
  **生产者线程将消息事务Message(Java)入队到消费者线程的消息队列MessageQueue(Java)中,如果符合唤醒条件则会调用Native方法nativeWake唤醒消费者线程**

# [](https://zhenkunhuang.github.io/2019/05/18/android-Looper-Native/#三、全文总结 "三、全文总结")三、全文总结

![Looper.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/caf834280a0c4558b6acd2aa91e1d7d7~tplv-k3u1fbpfcp-watermark.image?)

