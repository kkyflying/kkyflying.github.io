---
layout: post
math: true
title:  "MessageQueue分析"
date:   2022-02-13 14:24:12 +0800
categories: [Android,Handler]
tags: [Android,Handler,MessageQueue]
---


## 1. MessageQueue的简单介绍

```java
/**
 * Low-level class holding the list of messages to be dispatched by a
 * {@link Looper}.  Messages are not added directly to a MessageQueue,
 * but rather through {@link Handler} objects associated with the Looper.
 *
 * <p>You can retrieve the MessageQueue for the current thread with
 * {@link Looper#myQueue() Looper.myQueue()}.
 */
public final class MessageQueue {
    //中间先省略
}
```

1. 先看到MessageQueue的定义和官方的注释介绍，Messages并不会直接添加到MessageQueue，而是需要通过与Looper相关的handler来添加，可以通过```Looper#myQueue()```来获得当前线程下的MessageQueue。
2. 尽管MessageQueue叫消息队列，但实际上它内部是通过一个**单链表**的数据结构的来维护消息队列的，具体的实现是在Message.java中
3. 在MessageQueue最主要就是两个操作：插入和读取，同时读取操作把伴随删除操作，插入操作的是```enqueueMessage();```,读取操作是```next();``` 
4. 在MessageQueue中native将会在另外一篇分析

## 2. MessageQueue构造方法

```java
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit();
}
```

1. 这个构造方法在Looper中调用
2. nativeInit();返回的是在native层的MessageQueue，并且将引用地址返回给java层保存在mPtr变量中

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

因为Looper.java和MessageQueue.java是在同一个包下，所以Looper可以调用到MessageQueue的构造方法。

## 3. 插入操作 enqueueMessage

注：以下代码中删除了一部分不重要的部分

```java
boolean enqueueMessage(Message msg, long when) {
        synchronized (this) {
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                //注释1
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                //注释2
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                // 把消息插入到队列之中，但是一般不会唤醒这个事件队列，除非队列前面有一个barrier(屏障)
                // 和这个消息是队列中最早的异步消息
                // p.target == null是一个barrier，判断是否为异步消息msg.isAsynchronous()
                // mBlocked == true目前是阻塞中，需要同时满足此三个条件才去唤醒
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
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
                //就是一个插入队列的操作
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
### 在enqueueMessage的逻辑中，mMessages代表指向链表的头部，同时用临时变量p来保存一下mMessages，并且把when设置到新来的消息msg中。

#### 1. 当Message是第一条消息，或者是执行时间比mMessages还早的情况

请看在***注释1***的代码：

1. 设第一条消息msg1进来的时候，此时```mMessages==null```，即是```p==null```，因此把msg1插入链表的头部；
2. 设一条新消息msg2进来的时候，此时```when==0```，即是这条msg2需要立马执行，因此把msg2插入链表的头部；
3. 设一条新消息msg3进来的时候，此时```when<p.when```，即是这条msg3比目前在链表头部msg的执行时间更早，因此把msg2插入链表的头部；

![注释1](/img/2022-02-13-MessageQueue_enqueueMessage_1.svg)

#### 2. 当Message不是第一条消息，或者执行时间比队列的消息中要晚的情况

请看在***注释2***的代码：

1. 当不符合***注释1***的代码判断后(既不符合第一种情况下)就会进入这里的逻辑
2.  此时```p=mMessages```，即是p指链表头部，进入for第1次循环，把```prev=p```, 让prev记录链表头部，让```p=p.next```，让p指向链表头部的下一个msg，因此，当第1次循环的时候，设链表头部为第1个消息由，那么prev指向第1个msg，p指向第2个msg， 往后依次类推，当第N次循环的时候，prev指向第n个msg，p指向第n+1个msg
3. 直到n次循环时，p为null或者新消息的执行时间when比p的执行时间p.when更早的话，则退出循环，把新的msg插入到第n个msg的后面的，完成了消息入队的操作

![注释2](/img/2022-02-13-MessageQueue_enqueueMessage_2.svg)

## 4. 读取操作 next()

注：以下代码中删除关于IdleHandler的处理逻辑，后续会补充

```java
Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
    	//判断native层的MessageQueue是否存在，注释3
        if (ptr == 0) {
            return null;
        }
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            //注释4
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                //注释5
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                 //注释6
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message. 表示当前没有阻塞
                        mBlocked = false;
                        //如果prevMsg不为空，表示有消息屏障，
                        if (prevMsg != null) {
                            //异步消息要取出，让prevMsg的next指向msg的next
                            prevMsg.next = msg.next;
                        } else {
                            //prevMsg为空，表示没有消息屏障，更新mMessages指向mgs的next
                            mMessages = msg.next;
                        }
                        //取出消息，并断开在链表中的指向
                        msg.next = null;
                        //把这个消息标记为使用状态
                        msg.markInUse();
                        //返回消息到Looper处理
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }
            }
            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```

1. ***注释3***,判断native层的MessageQueue是否存在

2. ***注释4***,nativePollOnce()是一个native的方法,通过native层的MessageQueue进行阻塞nextPollTimeoutMillis时间

3. ***注释5***,

   3.1 在这里先判断```msg != null && msg.target == null```，一般来说msg.target不会为null，会指向一个Handler，当出现这个情况的时候，即是主动插入了一个消息屏障barrier，(在enqueueMessage()中出现的barrier是相同的概念)
   
   3.2 在通过```postSyncBarrier(long when)```插入一个barrier，并返回一个token来识别插入barrier，使用token传入```removeSyncBarrier()```来移除barrier
   
   3.3 如果发现一个barrier，就会进入循环找到第一个异步消息，在这个异步消息之前的同步消息都会被忽略(毕竟是一个单向链表)
   
4.  ***注释6***

    4.1  如果msg不为null，先判断```now < msg.when```执行时间有没有到，如果没到的话，设置nextPollTimeoutMillis阻塞时间，进入下一次循环，调用```nativePollOnce()```进行阻塞

    4.2 如果执行时间到了，```mBlocked = false;```表示当前没有阻塞
    
    4.3 判断```prevMsg```是否为空：
    
    如果```prevMsg```不为空的情况下，则表示有消息屏障，异步消息要取出，让prevMsg的next指向msg的next;
    
    如果```prevMsg```为空的情况下，则表示没有消息屏障，返回的是同步消息，并更新mMessages的执行msg的next;

![注释2](/img/2022-02-13-MessageQueue_enqueueMessage_3.svg)