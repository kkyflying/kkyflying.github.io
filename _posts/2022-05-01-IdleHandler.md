---
layout: post
math: true
title:  "IdleHandler分析"
date:   2022-05-1 11:16:33 +0800
categories: [Android,Handler]
tags: [Android,Handler,MessageQueue,IdleHandler]
---

## 1. IdleHandler的简单介绍
```java
/**
* Callback interface for discovering when a thread is going to block
* waiting for more messages.
*/
public static interface IdleHandler {
/**
* Called when the message queue has run out of messages and will now
* wait for more. Return true to keep your idle handler active, false
* to have it removed. This may be called if there are still messages
* pending in the queue, but they are all scheduled to be dispatched
* after the current time.
*/
boolean queueIdle();
}
```
1. 先看到IdleHandler的定义和官方的注释介绍,可以看到IdleHandler是MessageQueue的静态内部接口,这个接口用于等待发现线程阻塞等待着更多消息时候进行回调
2. 当messagequeue中消息已经next出去后,在等待enqueue的时候,queueIdle被会调用,如果queueIdle返回true着继续保存在mIdleHandlers中,如果返回false,则移除这个IdleHandler
3. 当消息队列中还有等待的消息的时候,queueIdle也可能被调用,因为在队列中等待的消息还没到时间执行

## 2. 调用流程分析
```java
@UnsupportedAppUsage
    Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        //注释0
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            nativePollOnce(ptr, nextPollTimeoutMillis);
            /**
            *这部分代码在上一篇讲解过了
            */               
                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                //注释1
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                //注释2
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }
                //注释3
                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }
            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            // 注释4
            for (int i = 0; i < pendingIdleHandlerCount; i++) {            
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler
                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }
            //注释5 Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;
            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }

```

1. (注释1) 执行next方法时,先处理掉消息队列里的msg,当出现指向链表头部的mMessages为null或是执行时间还未到的情况下就会进入到IdleHandler的逻辑
2. (注释2) 若pendingIdleHandlerCount<=0,则mIdleHandlers里的没添加过IdleHandler,退出本次循环
3. (注释3) 把mIdleHandlers里的IdleHandler复制mPendingIdleHandlers数组,mPendingIdleHandlers数组是临时的,用于后面进行循环调用queueIdle
4. (注释4) 遍历一遍mPendingIdleHandlers里的IdleHandler,并调用queueIdle方法,并且根据返回值,当false的时候,会把这个IdleHandler从mIdleHandlers里移除
5. (注释5) 把pendingIdleHandlerCount设置为0,防止在整个大的for循环(注释0)中重复,把nextPollTimeoutMillis设置为0,是因为有可能在执行idlehandler的时候,有新的message加入并需要执行,让下一次循环不进行休眠

## 3. 使用场景
1. 通过Looper.getMainLooper().getQueue().addIdleHandler在activity的onCreate方法中添加,这样就能实现延迟任务的执行,会在onResume跑完后执行,如果queueIdle设置返回false,这样就只执行一次,如果需要每次回到这个activity都需要执行一次这个延迟任务的,设置返回true即可
2. 源码中ActivityThread里有GcIdler,Idler,PurgeIdler三个任务

## 4. 使用注意事项:
1. 如果调用Looper.getMainLooper()的话,请不要在idlehandler做耗时操作,因为还是在主线程中执行,例如网络请求,读写文字等操作要避免
2. queueIdle的返回值如果是true的话,记得要使用removeIdleHandler移除掉,如果是返回false则不需要,因为在执行完后,就会立马删除了