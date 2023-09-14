---
layout: post
math: true
title:  "MessageQueue的Native方法分析"
date:   2023-09-14 23:30:49 +0800
categories: [Android,Handler]
tags: [Android,Handler,MessageQueue,Native]
---

## 前置
Handler并没有直接调用native，而是通过MessageQueue
相关的主要native代码

```java
frameworks/base/core/jni/android_os_MessageQueue.cpp
system/core/libutils/Looper.cpp
```

## MessageQueue.java中native方法的分析
路径 **frameworks/base/core/jni/android_os_MessageQueue.cpp**
可以看到MessageQueue的native方法

```java
private native static long nativeInit();
private native static void nativeDestroy(long ptr);
private native void nativePollOnce(long ptr, int timeoutMillis); /*non-static for callbacks*/
private native static void nativeWake(long ptr);
private native static boolean nativeIsPolling(long ptr);
private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
```

### nativeInit

1. nativeInit在MessageQueue.java中是在构造方法中调用，并返回了一个long的mPtr
```java
private long mPtr;
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit();
}
```

2. 通过在android_os_MessageQueue.cpp中找到nativeInit动态注册对应的android_os_MessageQueue_nativeInit函数
```cpp
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```
新建了一个NativeMessageQueue对象，并设置env，通过[reinterpret_cast](https://zhuanlan.zhihu.com/p/33040213)把nativeMessageQueue得到地址并返回，这样java层和native层就完成了相互绑定了，而后面mPtr也是需要多次用到
### nativeDestroy
1.dispose是在队列退出的时候使用，调用nativeDestroy也是为了销毁native的NativeMessageQueue对象
```java
// Disposes of the underlying message queue.
// Must only be called on the looper thread or the finalizer.
private void dispose() {
    if(mPtr != 0) {
        nativeDestroy(mPtr);
        mPtr = 0;
    }
}
```
```cpp
static void android_os_MessageQueue_nativeDestroy(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->decStrong(env);
}
```
### nativePollOnce
而nativePollOnce可以说是这些native方法中最重要的部分了，在[MessageQueue的next()方法中提到过](https://kkyflying.github.io/posts/MessageQueue/#4-%E8%AF%BB%E5%8F%96%E6%93%8D%E4%BD%9C-next)，这是一个阻塞方法，而timeoutMillis是代表阻塞的时间，在java层代码里并没有给出对这个timeoutMillis的值的说明，需要到native层中分析，先通过nativePollOnce找到对应动态注册的android_os_MessageQueue_nativePollOnce方法
```cpp
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
```
```cpp
class NativeMessageQueue : public MessageQueue, public LooperCallback {
public:
//省略
    void pollOnce(JNIEnv* env, jobject obj, int timeoutMillis);
//省略
};
```

这里可以看到是通过ptr找到对应的nativeMessageQueue对象，再调用pollOnce方法，可以看到pollOnce方法是NativeMessageQueue中的方法，要去找具体实现，如下

```cpp
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis);
    mPollObj = NULL;
    mPollEnv = NULL;

    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```
从这里可以看到，实际上调用是mLooper的pollOnce方法，再来看看这个mLooper是怎么来的

```cpp
#include <utils/Looper.h>

NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```

从NativeMessageQueue的构造方法中，可以看到这个mLooper就是来自system/core/libutils/Looper.cpp
那么实际上走的是Looper里的pollInner

```java
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        //省略
        result = pollInner(timeoutMillis);
    }
}

int Looper::pollInner(int timeoutMillis) {
    //省略
    int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
    //省略
}
```
可以看到核心点是使用到了epoll了，epoll是为了实现IO多路复用，可以通过man手册看到，epoll主要的函数就三个，**epoll_create**，**epoll_wait**，**epoll_ctl**
所以从java层传入的timeoutMillis最后是传递给了epoll_wait，那么通过man手册就可以看到说明

1. 当timeoutMillis<-1的时候，epoll_wait会无限阻塞；
2. 当timeoutMillis=0的时候，epoll_wait会立马返回；
3. 当timeoutMillis>0的时候，epoll_wait会阻塞timeoutMillis时长

综上所述，就可以知道java层的MessageQueue#next中的nativePollOnce为何阻塞了，那么问题就来了，被阻塞后，就需要去唤醒
那就先分析epoll_wait是在等待那个io有响应，那就要看看epoll_ctl，有什么io加入到epoll中被监听
epoll_ctl的第二参数op，有三个值

1. EPOLL_CTL_ADD 添加目标的文件描述符到epoll中
2. EPOLL_CTL_MOD 修改epoll中目标的文件描述符
3. EPOLL_CTL_DEL 从epoll中删除目前的文件描述符

再看Looper的构造方法
```cpp
Looper::Looper(bool allowNonCallbacks)
    : mAllowNonCallbacks(allowNonCallbacks),
      mSendingMessage(false),
      mPolling(false),
      mEpollRebuildRequired(false),
      mNextRequestSeq(WAKE_EVENT_FD_SEQ + 1),
      mResponseIndex(0),
      mNextMessageUptime(LLONG_MAX) {
    mWakeEventFd.reset(eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC));
    LOG_ALWAYS_FATAL_IF(mWakeEventFd.get() < 0, "Could not make wake event fd: %s", strerror(errno));

    AutoMutex _l(mLock);
    rebuildEpollLocked();
}
```
在此构造方法中，重点的是mWakeEventFd和rebuildEpollLocked();
```cpp
void Looper::rebuildEpollLocked() {
    //省略
    epoll_event wakeEvent = createEpollEvent(EPOLLIN, WAKE_EVENT_FD_SEQ);
    int result = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, mWakeEventFd.get(), &wakeEvent);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake event fd to epoll instance: %s",
        strerror(errno));
    //省略
}
```
在rebuildEpollLocked中，可以找到新创建的wakeEvent以及mWakeEventFd.get()通过epoll_ctl加入到epoll的监听中（EPOLL_CTL_ADD标志），对**mWakeEventFd**进行监听，那么再回过来看构造方法中对**mWakeEventFd**的配置
```cpp
mWakeEventFd.reset(eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC));
```
这里又可以看看man手册，
```cpp
NAME
       eventfd - create a file descriptor for event notification

SYNOPSIS
       #include <sys/eventfd.h>

       int eventfd(unsigned int initval, int flags);
```
eventfd就是创建事件通知的一个文件描述符fd，仅仅作为事件通知来使用，和常见的socket不太一样。当对这个fd写入，就会唤醒epoll_wait,那么就找到了，如下wake()
```cpp
void Looper::wake() {
	//省略
    uint64_t inc = 1;
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd.get(), &inc, sizeof(uint64_t)));
	//省略
}
```
### nativeWake
找到对应的native代码,实际调用就是上述的Looper里的wake函数了
```cpp
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake();
}

void NativeMessageQueue::wake() {
    mLooper->wake();
}
```
在java层调用了MessageQueue#enqueueMessage和MessageQueue#removeSyncBarrier都可能会调用到wake，来解除阻塞，驱动Looper的转动从而调用到MessageQueue#next
### nativeIsPolling
找到对应的native，实际上只是返回了一个mPolling变量的值，在构造方法里先设置了默认为false，
```cpp
static jboolean android_os_MessageQueue_nativeIsPolling(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    return nativeMessageQueue->getLooper()->isPolling();
}
```
```cpp
bool Looper::isPolling() const {
    return mPolling;
}
```
在pollInner函数中的改变mPolling的状态，表示当前的looper是否空闲
```cpp
int Looper::pollInner(int timeoutMillis) {
    // We are about to idle.
    mPolling = true;
    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
    // No longer idling.
    mPolling = false;
}
```
### nativeSetFileDescriptorEvents
#### updateOnFileDescriptorEventListenerLocked
先看MessageQueue.java,能找到nativeSetFileDescriptorEvents的调用位置
```java
private void updateOnFileDescriptorEventListenerLocked(FileDescriptor fd, int events,
                                                       OnFileDescriptorEventListener listener) {
    final int fdNum = fd.getInt$();

    int index = -1;
    FileDescriptorRecord record = null;
    if (mFileDescriptorRecords != null) {
        index = mFileDescriptorRecords.indexOfKey(fdNum);
        if (index >= 0) {
            record = mFileDescriptorRecords.valueAt(index);
            if (record != null && record.mEvents == events) {
                return;
            }
        }
    }

    if (events != 0) {
        events |= OnFileDescriptorEventListener.EVENT_ERROR;
        if (record == null) {
            if (mFileDescriptorRecords == null) {
                mFileDescriptorRecords = new SparseArray<FileDescriptorRecord>();
            }
            record = new FileDescriptorRecord(fd, events, listener);
            mFileDescriptorRecords.put(fdNum, record);
        } else {
            record.mListener = listener;
            record.mEvents = events;
            record.mSeq += 1;
        }
        //加入操作
        nativeSetFileDescriptorEvents(mPtr, fdNum, events);
    } else if (record != null) {
        record.mEvents = 0;
        mFileDescriptorRecords.removeAt(index);
        //删除操作
        nativeSetFileDescriptorEvents(mPtr, fdNum, 0);
    }
}

```
updateOnFileDescriptorEventListenerLocked是一个私有方法，被addOnFileDescriptorEventListener和removeOnFileDescriptorEventListener这一对方法调用
#### addOnFileDescriptorEventListener
```java
public void addOnFileDescriptorEventListener(@NonNull FileDescriptor fd,
            @OnFileDescriptorEventListener.Events int events,
            @NonNull OnFileDescriptorEventListener listener) {
        if (fd == null) {
            throw new IllegalArgumentException("fd must not be null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("listener must not be null");
        }

        synchronized (this) {
            updateOnFileDescriptorEventListenerLocked(fd, events, listener);
        }
    }
```
从源码上的注释来看，是添加文件描述符监听器，每对一个文件描述符添加了监听器后，都会替换之前的设置，同时，每一个文件描述符仅有一个监听器，光是从这里的注释就已经能猜想到是和epoll有关系了，那么看下对应的native代码，如下：
```cpp
static void android_os_MessageQueue_nativeSetFileDescriptorEvents(JNIEnv* env, jclass clazz,
jlong ptr, jint fd, jint events) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->setFileDescriptorEvents(fd, events);
}
```
```cpp
void NativeMessageQueue::setFileDescriptorEvents(int fd, int events) {
    if (events) {
        int looperEvents = 0;
        if (events & CALLBACK_EVENT_INPUT) {
            looperEvents |= Looper::EVENT_INPUT;
        }
        if (events & CALLBACK_EVENT_OUTPUT) {
            looperEvents |= Looper::EVENT_OUTPUT;
        }
        mLooper->addFd(fd, Looper::POLL_CALLBACK, looperEvents, this,
                reinterpret_cast<void*>(events));
    } else {
        mLooper->removeFd(fd);
    }
}
```
从这里也能看出来，当在nativeSetFileDescriptorEvents传入为0是调用了removeFd，非0调用了addFd
#### addFd和removeFd
最终还是要看下Looper.cpp的addFd和removeFd
```cpp
int Looper::removeFd(int fd) {
    AutoMutex _l(mLock);
    const auto& it = mSequenceNumberByFd.find(fd);
    if (it == mSequenceNumberByFd.end()) {
        return 0;
    }
    return removeSequenceNumberLocked(it->second);
}

int Looper::removeSequenceNumberLocked(SequenceNumber seq) {
	//省略
    int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_DEL, fd, nullptr);
	//省略
    return 1;
}
```
从removeFd函数到removeSequenceNumberLocked函数中，核心还是在epoll_ctl，操作位标志是EPOLL_CTL_DEL，从epoll中删除对这个fd的监听
再看addFd函数里，核心依旧是epoll_ctl，操作标志位是EPOLL_CTL_ADD，把fd加入到epoll中监听
```cpp
int Looper::addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data) {
	//省略
	int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, fd, &eventItem);
	//省略
    return 1;
}
```
#### 监听如何回调到上层
其实这里从上层追到native或者是从native追上到上层都是可以的。但是现在先顺着上面Looper::addFd里的思路来看更连贯一些
```cpp
int Looper::addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data) {
	//省略
    	const SequenceNumber seq = mNextRequestSeq++;
    	//注释1
        Request request;
        request.fd = fd;
        request.ident = ident;
        request.events = events;
        request.callback = callback;
        request.data = data;
    	//注释2
        epoll_event eventItem = createEpollEvent(request.getEpollEvents(), seq);
        auto seq_it = mSequenceNumberByFd.find(fd);
        if (seq_it == mSequenceNumberByFd.end()) {
            int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, fd, &eventItem);
            if (epollResult < 0) {
                ALOGE("Error adding epoll events for fd %d: %s", fd, strerror(errno));
                return -1;
            }
            //注释3
            mRequests.emplace(seq, request);
            mSequenceNumberByFd.emplace(fd, seq);
        } else {
	//省略
    return 1;
}
```
##### 注释1
这里是构建一个Request，Request定义在**system/core/libutils/include/utils/Looper.h**中的结构体，如下
```cpp
struct Request {
      int fd;
      int ident;
      int events;
      sp<LooperCallback> callback;
      void* data;

      uint32_t getEpollEvents() const;
};
```
在这个结构体中，重要的是fd和callback，fd是文件描述符，而这里传入的callback实际上是NativeMessageQueue，首先NativeMessageQueue是继承了LooperCallback，在setFileDescriptorEvents传入了this，这里就是先实现fd和NativeMessageQueue的绑定（实际上NativeMessageQueue又和上层的MessageQueue绑定），
##### 注释2
这里构建epoll_event,先记着这里seq存入了到data.u64这里，接着来将会用到，
```cpp
epoll_event createEpollEvent(uint32_t events, uint64_t seq) {
    return {.events = events, .data = {.u64 = seq}};
}
```
然后就是通过epoll_ctl把fd加入epoll中监听
##### 注释3
这里就是两个同步的map，seq和request绑定，fd和seq绑定，也就是说有fd就能找到对应的request，也就是说能找到对应的NativeMessageQueue，接着就是能找到对应的MessageQueue了
```cpp
    // Locked maps of fds and sequence numbers monitoring requests.
    // Both maps must be kept in sync at all times.
    std::unordered_map<SequenceNumber, Request> mRequests;               // guarded by mLock
    std::unordered_map<int /*fd*/, SequenceNumber> mSequenceNumberByFd;  // guarded by mLock
```
##### 再看pollInner
这里又要回到pollInner中来看，
```cpp
int Looper::pollInner(int timeoutMillis) {
	//省略
    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

	//省略
    //注释4
    for (int i = 0; i < eventCount; i++) {
        const SequenceNumber seq = eventItems[i].data.u64;
        uint32_t epollEvents = eventItems[i].events;
        if (seq == WAKE_EVENT_FD_SEQ) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {
            //注释5
            const auto& request_it = mRequests.find(seq);
            if (request_it != mRequests.end()) {
                const auto& request = request_it->second;
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                mResponses.push({.seq = seq, .events = events, .request = request});
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x for sequence number %" PRIu64
                      " that is no longer registered.",
                      epollEvents, seq);
            }
        }
    }
Done: ;

//省略

    // Invoke all response callbacks.
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
            ALOGD("%p ~ pollOnce - invoking fd event callback %p: fd=%d, events=0x%x, data=%p",
                    this, response.request.callback.get(), fd, events, data);
#endif
            // Invoke the callback.  Note that the file descriptor may be closed by
            // the callback (and potentially even reused) before the function returns so
            // we need to be a little careful when removing the file descriptor afterwards.
            //注释6
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                AutoMutex _l(mLock);
                removeSequenceNumberLocked(response.seq);
            }

            // Clear the callback reference in the response structure promptly because we
            // will not clear the response vector itself until the next poll.
            response.request.callback.clear();
            result = POLL_CALLBACK;
        }
    }
    return result;
}
```
###### 注释4
假设我当一个fd进行写操作，唤醒了epoll_wait，那么eventCount大于0，而这里的seq的值就是来自 eventItems[i].data.u64和上面的注释2对照上了。
###### 注释5
马上就是使用mRequests通过seq找到request了，并一起push到了mResponses，mResponses的结构体如下
```cpp
struct Response {
        SequenceNumber seq;
        int events;
        Request request;
};
// This state is only used privately by pollOnce and does not require a lock since
// it runs on a single thread.
Vector<Response> mResponses;
```
###### 注释6
这里就是遍历了response，通过request找到了callback，而这里的callback也就是上面所说的NativeMessageQueue了，接下来就是看NativeMessageQueue中实现的handleEvent函数

##### handleEvent
handleEvent实际上是在Looper.h里定义的，但是NativeMessageQueue又继承了LooperCallback，所以handleEvent函数的现实在NativeMessageQueue中
```cpp

class LooperCallback : public virtual RefBase {
protected:
    virtual ~LooperCallback();

public:
    virtual int handleEvent(int fd, int events, void* data) = 0;
};
```

```cpp
int NativeMessageQueue::handleEvent(int fd, int looperEvents, void* data) {
    int events = 0;
    if (looperEvents & Looper::EVENT_INPUT) {
        events |= CALLBACK_EVENT_INPUT;
    }
    if (looperEvents & Looper::EVENT_OUTPUT) {
        events |= CALLBACK_EVENT_OUTPUT;
    }
    if (looperEvents & (Looper::EVENT_ERROR | Looper::EVENT_HANGUP | Looper::EVENT_INVALID)) {
        events |= CALLBACK_EVENT_ERROR;
    }
    int oldWatchedEvents = reinterpret_cast<intptr_t>(data);
    //注释7
    int newWatchedEvents = mPollEnv->CallIntMethod(mPollObj,
            gMessageQueueClassInfo.dispatchEvents, fd, events);
    if (!newWatchedEvents) {
        return 0; // unregister the fd
    }
    if (newWatchedEvents != oldWatchedEvents) {
        setFileDescriptorEvents(fd, newWatchedEvents);
    }
    return 1;
}
```
###### 注释7

1. mPollEnv和mPollObj就是在NativeMessageQueue::pollOnce函数里保存了env和pollObj，因handleEvent是在pollOnce内回调的，保证了mPollEnv和mPollObj不会为null，即使找到了对应的上层MessageQueue
2. gMessageQueueClassInfo的结构体如下
```cpp
static struct {
    jfieldID mPtr;   // native object attached to the DVM MessageQueue
    jmethodID dispatchEvents;
} gMessageQueueClassInfo;
```
gMessageQueueClassInfo就是在register_android_os_MessageQueue动态注册里绑定了MessageQueue.java的dispatchEvents方法了
##### dispatchEvents
###### 注释8
FileDescriptorRecord其实就是一个实体，包含了mListener和fd，而mFileDescriptorRecords就是FileDescriptorRecord的集合，用fd来定位，这点可以在上面的updateOnFileDescriptorEventListenerLocked中找到证实，通过native上报了的fd来找到对应的record
###### 注释9
再调用record中的mListener执行了onFileDescriptorEvents，至此就完成了一次上层设置监听fd，操作fd，监听fd变化，native层上报到上层的回调
```java
private int dispatchEvents(int fd, int events) {
    // Get the file descriptor record and any state that might change.
    final FileDescriptorRecord record;
    final int oldWatchedEvents;
    final OnFileDescriptorEventListener listener;
    final int seq;
    synchronized (this) {
        //注释8
        record = mFileDescriptorRecords.get(fd);
        if (record == null) {
            return 0; // spurious, no listener registered
        }

        oldWatchedEvents = record.mEvents;
        events &= oldWatchedEvents; // filter events based on current watched set
        if (events == 0) {
            return oldWatchedEvents; // spurious, watched events changed
        }

        listener = record.mListener;
        seq = record.mSeq;
    }

    // Invoke the listener outside of the lock.
    // 注释9
    int newWatchedEvents = listener.onFileDescriptorEvents(
        record.mDescriptor, events);
    if (newWatchedEvents != 0) {
        newWatchedEvents |= OnFileDescriptorEventListener.EVENT_ERROR;
    }

    // Update the file descriptor record if the listener changed the set of
    // events to watch and the listener itself hasn't been updated since.
    if (newWatchedEvents != oldWatchedEvents) {
        synchronized (this) {
            int index = mFileDescriptorRecords.indexOfKey(fd);
            if (index >= 0 && mFileDescriptorRecords.valueAt(index) == record
                && record.mSeq == seq) {
                record.mEvents = newWatchedEvents;
                if (newWatchedEvents == 0) {
                    mFileDescriptorRecords.removeAt(index);
                }
            }
        }
    }

    // Return the new set of events to watch for native code to take care of.
    return newWatchedEvents;
}
```

```java
    private FileDescriptor getFileDescriptor(Handler handler){
        FileDescriptor fd = null;
        FileOutputStream fos = null;
        try {
            fos =  new FileOutputStream("/storage/emulated/0/Documents/1.txt");
            fd = fos.getFD();
            handler.getLooper().getQueue().addOnFileDescriptorEventListener(fd, MessageQueue.OnFileDescriptorEventListener.EVENT_OUTPUT,
                    new MessageQueue.OnFileDescriptorEventListener() {
                        @Override
                        public int onFileDescriptorEvents(@NonNull FileDescriptor fd, int events) {
                            Log.i(TAG, "onFileDescriptorEvents="+fd + " events="+events + " "+Thread.currentThread().getName());
                            return 0;
                        }
                    });
            String content = "test_test_123\n";
            byte[] bytes = content.getBytes();
            fos.write(bytes);
            System.out.println("文件写入成功！");
            fos.close();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            return fd;
        }
    }
```


