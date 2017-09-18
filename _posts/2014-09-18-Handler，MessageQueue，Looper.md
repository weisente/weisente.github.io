---
layout: post
title:  "Handler，MessageQueue，Looper"
date:   2014-09-18 16:14:09 +0800
categories: sound code
---

**深入理解Handler，MessageQueue，Looper**
转：http://www.jianshu.com/p/6d143b8c15ee
前言：
其实讲 Handler 内部机制的博客已经很多了，但是自己还是要在看一遍，源码是最好的资料。在具体看源码之前，有必要先理解一下 Handler、Looper、MessageQueue 以及 Message 他们的关系。
Looper: 是一个消息轮训器，他有一个叫 loop() 的方法，用于启动一个循环，不停的去轮询消息池
MessageQueue: 就是上面说到的消息池
Handler: 用于发送消息，和处理消息
Message: 一个消息对象

源码分析开始：
一切要从Handler的构造函数开始讲起

```
public Handler(Callback callback, boolean async) {
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
我们可以看到，Handler定义了一个MessageQueue对象mQueue和一个Looper对象mLooper。顺着源码继续往下看，跳转到Looper.myLooper()方法。

```
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```
这里涉及了ThreadLocal，暂时不讲。通过ThreadLocal的get方法获取Looper并返回。那么问题来了？我们只看到了get，并没有看到set。如果没有set的话，get出来就会为null，通过Handler的构造函数我们知道，mLooper==null会抛出异常。而我们在使用Handler的过程中并没有遇到该异常。那问题来了，到底在哪里进行了set呢？通过对Looper源码搜索发现，改方法进行set操作：

```
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```
可以看到在prepare()方法中进行了set操作，那么问题又来了，哪里调用了该方法呢？因为prepare方法是私有方法，所以肯定是本类中调用，通过搜索发现以下方法调用了prepare()方法：

```
 * Initialize the current thread as a looper, marking it as an
 * application's main looper. The main looper for your application
 * is created by the Android environment, so you should never need
 * to call this function yourself.  See also: {@link #prepare()}
 */
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```
那一切就是顺利成章了。到这里，有人又会问，那这个方法又是谁调用的呢？看注释发现，该方法在启动app的时候就已经调用了。具体是在ActivityThread的main方法中启动。
到这里为止，我们了解了Handler，Looper的初始化相关知识。接下来，我们需要了解的是如何进行发送和处理Message。
发送Message代码如下：

```
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```
这个方法我们主要是看enqueueMessage()：

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
这里关键的是看懂msg.target = this;msg就是Message的对象，那么看Message源码发现，它的target属性是Handler target;那么，为什么发送消息的时候需要将this(当前Handler对象)带过去呢？咋们暂且继续...
这个方法实际执行的还是queue.enqueueMessage()，我们找到MessageQueue类的相关方法,发现以下代码
msg.next = p; // invariant: p == prev.next
prev.next = msg;
通过这两行代码我们发现，MessageQueue并不是队列，而是单链表。所以下次面试的时候，如果你支出handler的消息队列其实是利用Message的单链表实现的肯定能加分的。
到此，发送消息已经讲完了。下面我们看看处理消息是怎样进行的呢？我们知道，Loope会从MessageQueue中不断拿消息，我们看看Looper.loop()代码：

```
public static void loop() {
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        msg.target.dispatchMessage(msg);
        msg.recycleUnchecked();
    }
}
```
这里取出消息并分发之。是不是有顿悟的感觉，回到上面我们遗留的问题：msg.target = this，将this传递过去，言外之意就是哪个Handler发送的消息就由哪个Handler进行处理。那么我们来看看Handler的dispatchMessage 方法:

```
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```
终于见到我们最常见的handleMessage()了。他首先判断 Message 对象的 callback 对象是不是为空，如果不为空，就直接调用 handleCallback 方法，并把 msg 对象传递过去，这样消息就被处理了,我们来看 Message 的 handleCallback 方法

```
private static void handleCallback(Message message) {
    message.callback.run();
}
```
没什么好说的了，直接调用 Handler post 的 Runnable 对象的 run() 方法。
如果在发送消息时，我们没有给 Message 设置 callback 对象，那么程序会执行到 else 语句块，此时首先判断 Handler 的 mCallBack 对象是不是空的，如果不为空，直接调用 mCallback 的 handleMessage 方法进行消息处理。最终，只有当 Handler 的 mCallback 对象为空，才会执行自己的 handleMessage 方法。