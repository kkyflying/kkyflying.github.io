---
layout: post
math: true
title:  " ThreadLocal分析"
date:   2021-03-24 23:26:31 +0800
categories: [Android,Handler]
tags: [Android,Handler,ThreadLocal]
---

## 1. ThreadLocal的简单介绍

```java
package java.lang;
import java.lang.ref.*;
import java.util.Objects;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

/**
 * This class provides thread-local variables.  These variables differ from
 * their normal counterparts in that each thread that accesses one (via its
 * {@code get} or {@code set} method) has its own, independently initialized
 * copy of the variable.  {@code ThreadLocal} instances are typically private
 * static fields in classes that wish to associate state with a thread (e.g.,
 * a user ID or Transaction ID).
 */

public class ThreadLocal<T> {
    //中间先省略
}
```

先看到ThreadLocal的定义和官方的注释介绍，ThreadLocal本身只是一个简单的class并且可以传入一个泛类，官方的意思是这个class可以提供线程本地变量的存储，不同于一般的变量，因为每一个线程只能get和set到当前对应线程的值，这些变量是相互独立的，可以很粗狂的理解为这是一个map，而用线程来当做key，独立的值为value。

## 2. ThreadLocal使用
1. **demo**

   ```java
   class ThreadLocalTest {
       public static void main(String[] args) {
           ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
           threadLocal.set(0);
           System.out.println(Thread.currentThread().getName() + " value : " +threadLocal.get());
            new Thread("thread-1"){
               @Override
               public void run() {
                   threadLocal.set(1);
                   System.out.println(Thread.currentThread().getName() + " value : " + threadLocal.get());
               }
           }.start();
           new Thread("thread-2"){
               @Override
               public void run() {
                   threadLocal.set(2);
                   System.out.println(Thread.currentThread().getName() + " value : " + threadLocal.get());
               }
           }.start();
           new Thread("thread-3"){
               @Override
               public void run() {
                   System.out.println(Thread.currentThread().getName() + " value : " + threadLocal.get());
               }
           }.start();
       }
   }
   ```

2. **运行结果**

   ```
   main value : 0
   thread-1 value : 1
   thread-2 value : 2
   thread-3 value : null
   ```

3. **使用说明**

   通过这个简单的demo可以看出来,在不同线程下的get和set的值是互相独立不影响的,使用十分方便.一般来说,使用场景有两种,第一情况是:**当某个数据需要以线程为作用域(或是key),并且不同线程下的数据必须独立不被影响的时候**,例如在Android的Handler中就是使用了ThreadLocal来进行对Looper的管理,第二情况是:**在十分复杂的逻辑下进行对象的传递**,例如当需要把一个接口在一个线程中多次传递,并且在不同的线程下使用不同的接口来监听回调,正常下,需要为一个个方法添加这个接口的参数,进行一层层的传递,但是在后面修改逻辑,维护等工作上就会十分折磨,这个时候通过ThreadLocal现在线程初始start后,set保存一个接口,在逻辑需要回调接口的时候,直接get到这个接口,这样就不必在方法上添加接口参数,大大减少了工作量,具体实现在Java后台的实现比较多,可以参考[stackoverflow上的一个回答](https://stackoverflow.com/questions/7625922/purpose-and-use-of-threadlocal-class)


## 3. ThreadLocal分析

1. **ThreadLocal#set()**和**ThreadLocal#get()**
   ```java
   public void set(T value) {
       	//获得当前的线程对象
           Thread t = Thread.currentThread();
       	//通过当前的线程获取到ThreadLocalMap
           ThreadLocalMap map = getMap(t);
           if (map != null)
               //如果map存在,则value存入到map,以当前的ThreadLocal为key
               map.set(this, value);
           else
               //如果map不存在,创建一个新的ThreadLocalMap,并存入value
               createMap(t, value);
   }
   ```

   ```java
   public T get() {
       	//获得当前的线程对象
           Thread t = Thread.currentThread();
      		//通过当前的线程获取到ThreadLocalMap
           ThreadLocalMap map = getMap(t);
           if (map != null) {
               //如果map不为空,则通过当的ThreadLocal为key获取e
               ThreadLocalMap.Entry e = map.getEntry(this);
               if (e != null) {
                   //若e不为空,则通过这e获得value并返回
                   @SuppressWarnings("unchecked")
                   T result = (T)e.value;
                   return result;
               }
           }
       	//如果map为空或者e为空,则返回设置的默认初始值
           return setInitialValue();
   }
   ```

   通过查看get()和set()方法,可以发现逻辑十分简单,重点在于ThreadLocalMap类(是ThreadLocal的静态内部类)以及getMap(),createMap(),setInitialValue()等方法.

2. **ThreadLocal#getMap()**和**ThreadLocal#createMap()**以及**ThreadLocal#setInitialValue()**
   
   ```java
   /**
   * Get the map associated with a ThreadLocal. Overridden in
   * InheritableThreadLocal.
   *
   * @param  t the current thread
   * @return the map
   */
   ThreadLocalMap getMap(Thread t) {
   	return t.threadLocals;
   }
   ```
   
   ```java
   /* ThreadLocal values pertaining to this thread. This map is maintained
   * by the ThreadLocal class. */
   ThreadLocal.ThreadLocalMap threadLocals = null;
   ```
   
   从getMap()中看到这个ThreadLocalMap竟然就是来自传入的Thread中的一个变量,而查看Thread中对这个变量的注解简单翻译来看,*这个ThreadLocal的值属于这个thread,这个map被ThreadLocal所维护持有*.
   
   ```java
   /**
   * Create the map associated with a ThreadLocal. Overridden in
   * InheritableThreadLocal.
   *
   * @param t the current thread
   * @param firstValue value for the initial entry of the map
   */
   void createMap(Thread t, T firstValue) {
   	t.threadLocals = new ThreadLocalMap(this, firstValue);
   }
   ```
   
   根据注解来看,*通过当前ThreadLocal来构建一个ThreadLocalMap,并且把传入一个初始值firstValue来为map构建一个初始的entry*,而createMap()只在get(),set(),setInitialValue()中被调用.
   
   ```java
   /**
   * Variant of set() to establish initialValue. Used instead
   * of set() in case user has overridden the set() method.
   *
   * @return the initial value
   */
   private T setInitialValue() {
   	T value = initialValue();fenge
   	Thread t = Thread.currentThread();
   	ThreadLocalMap map = getMap(t);
   	if (map != null)
   		map.set(this, value);
   	else
   		createMap(t, value);
   	return value;
   }
   ```
   
   setInitialValue()和set()的逻辑几乎一模一样,可以理解为setInitialValue()干了一个事,获取了一个默认的value并传入set()中,在这里值得关注就是initialValue()了

3. **ThreadLocal#initialValue()**

   ```java
   /**
   * Returns the current thread's "initial value" for this
   * thread-local variable.  This method will be invoked the first
   * time a thread accesses the variable with the {@link #get}
   * method, unless the thread previously invoked the {@link #set}
   * method, in which case the {@code initialValue} method will not
   * be invoked for the thread.  Normally, this method is invoked at
   * most once per thread, but it may be invoked again in case of
   * subsequent invocations of {@link #remove} followed by {@link #get}.
   *
   * <p>This implementation simply returns {@code null}; if the
   * programmer desires thread-local variables to have an initial
   * value other than {@code null}, {@code ThreadLocal} must be
   * subclassed, and this method overridden.  Typically, an
   * anonymous inner class will be used.
   *
   * @return the initial value for this thread-local
   */
   protected T initialValue() {
   	return null;
   }
   ```

   这initialValue()可以被重写,如果直接使用ThreadLocal,而没有先set()设置参数,那么直接返回null,可以通过上面的demo代码验证了.而initialValue()的第一次调当在线程中通过get()获得fenge参数时调用.除非事先调用了set(),在这种情况下,initialValue()将不会被调用在这个线程里,通常,initialValue()这个方法在每个线程中只会被调用一次,但是initialValue()方法也可能会被再一次调用,在之后调用了remove()再调用get()的情况.

4. **ThreadLocal#remove()**

   ```java
   /**
   * Removes the current thread's value for this thread-local
   * variable.  If this thread-local variable is subsequently
   * {@linkplain #get read} by the current thread, its value will be
   * reinitialized by invoking its {@link #initialValue} method,
   * unless its value is {@linkplain #set set} by the current thread
   * in the interim.  This may result in multiple invocations of the
   * {@code initialValue} method in the current thread.
   *
   * @since 1.5
   */
   public void remove() {
   	ThreadLocalMap m = getMap(Thread.currentThread());
   	if (m != null)fenge
       	m.remove(this);
   }
   ```

   通过当前线程来删除ThreadLocal里的值.在这个线程里,如果这个threadlocal的值在随后调用了get()读取的话,将会被通过initialValue()重新初始化,除非这个值被set()方法被临时赋值,这可能导致在当前线程中多次调用initialValue()

## 4. ThreadLocalMap

通过前面的铺垫分析,这些方法里都围绕着ThreadLocalMap进行操作,可见ThreadLocalMap才是ThreadLocal的核心部分,接下来将会对此进行详细的分析

1. **ThreadLocalMap.Entry**

   ```java
   /**
   * The entries in this hash map extend WeakReference, using
   * its main ref field as the key (which is always a
   * ThreadLocal object).  Note that null keys (i.e. entry.get()
   * == null) mean that the key is no longer referenced, so the
   * entry can be expunged from table.  Such entries are referred to
   * as "stale entries" in the code that follows.
   */
   static class Entry extends WeakReference<ThreadLocal<?>> {
   	/** The value associated with this ThreadLocal. */
       Object value;
   
   	Entry(ThreadLocal<?> k, Object v) {
   		super(k);
   		value = v;
   	}
   }
   ```

   Entry类是ThreadLocalMap的静态内部类，并继承了WeakReference，对ThreadLocal进行弱引用，对value进行强引用

2. **ThreadLocalMap和Entry的关系**
   首先ThreadLocalMap确实是一个map，通过Entry[] table实现，而通过ThreadLocal#set()方法保存的值就到保存到了Entry类中的value，同时因为Entry的value是Object类型，ThreadLocalMap中实际上是可以保存不同类型的数据。
   
3. **ThreadLocalMap**和**ThreadLocal**以及**Thread**三者的关系

   1. Thread类中持有着ThreadLocalMap变量，ThreadLocal通过获得Thread.currentThread()获得当前Thread对象来间接操作ThreadLocalMap

   2. **每一个Thread中都只有一个ThreadLocalMap**(虽然还有一个inheritableThreadLocals变量也是ThreadLocalMap，但是具体实现是InheritableThreadLocal，InheritableThreadLocal是ThreadLocalMap的子类)

   3. 就对于一个ThreadLocal来讲，在不同Thread下是操作不同的ThreadLocalMap(即是操作不同的table)，而对于多个ThreadLocal在同一个Thread下是操作同一个ThreadLocalMap(即是操作相同table，然后每个ThreadLocal实例在table中索引i是不同的)

      ```java
      ThreadLocal<Integer> a = new ThreadLocal<>();
      ThreadLocal<String> b = new ThreadLocal<>();
      ThreadLocal<Boolean> c = new ThreadLocal<>();
      ```

      如上，现在有三个ThreadLocal对象abc,都在主线程下操作，此时都对同一个ThreadLocalMap进行操作，即是对table操作，这个时候的问题就在于，abc在set()和get()过程中如何确定在table数组中的位置，同时保证这个位置不冲突。

4. **ThreadLocalMap构造方法以及set()和get()**

   ```java
   /**
   * Construct a new map initially containing (firstKey, firstValue).
   * ThreadLocalMaps are constructed lazily, so we only create
   * one when we have at least one entry to put in it.
   */
   ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
       //初始table，容量是16
   	table = new Entry[INITIAL_CAPACITY];
       //通过hash来计算出存放的位置，这个算法后面还会具体将到
       int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
       table[i] = new Entry(firstKey, firstValue);
       size = 1;
       setThreshold(INITIAL_CAPACITY);
   }
   ```
	这个构造方法里要传入key和value，ThreadLocalMaps是被懒加载，只有第一次要存入数据的时候才调用，还要计算出第一次要存放在table中的位置

   ```java
   /**
   * Set the value associated with key.
   *
   * @param key the thread local object
   * @param value the value to be set
   */
   private void set(ThreadLocal<?> key, Object value) {
   	// We don't use a fast path as with get() because it is at
   	// least as common to use set() to create new entries as
   	// it is to replace existing ones, in which case, a fast
   	// path would fail more often than not.
       Entry[] tab = table;
       int len = tab.length;
       //通过hash计算在tab中的存储位置
       int i = key.threadLocalHashCode & (len-1);
       //如果获取的e为空，则跳出循环，不为空的情况下，进入循环，通过Entry e.get()中是的
       //ThreadLocal对象和传入的key是否相同，如果相同则更新e.value的值，完成一次更新数据
       //（如何保证更新到正确的位置要依靠hash的计算，这个后面继续讲解）
       for (Entry e = tab[i];
       	e != null;
       	e = tab[i = nextIndex(i, len)]) {
           ThreadLocal<?> k = e.get();        
           if (k == key) {
           	e.value = value;
           	return;
           }
           if (k == null) {
               //如果这个e.get()出来为空，则代替掉这个旧的Entry
           	replaceStaleEntry(key, value, i);
               return;
           }
        }
        //直接跳出上面的循环后，直接把数据保存tab中第i个位置
        tab[i] = new Entry(key, value);
        int sz = ++size;
        //如果有旧的Entry要清理通并且table的大小已经大于等于阀值则要重新计算hash
        //还要进行扩容
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
        	rehash();
   }
   ```
   ```java
    /**
   * Get the entry associated with key.  This method
   * itself handles only the fast path: a direct hit of existing
   * key. It otherwise relays to getEntryAfterMiss.  This is
   * designed to maximize performance for direct hits, in part
   * by making this method readily inlinable.
   *
   * @param  key the thread local object
   * @return the entry associated with key, or null if no such
   */
   private Entry getEntry(ThreadLocal<?> key) {
       //通过key的计算数据存储的位置
   	int i = key.threadLocalHashCode & (table.length - 1);
       Entry e = table[i];
       //判断是否为空，以及key是否相同
       if (e != null && e.get() == key)
       	return e;
       else
           return getEntryAfterMiss(key, i, e);
   }
   ```
   
   ThreadLocalMap的构造方法以及set()和get()的关键点都是在计算table中i的位置
   
   ```java
   //构造方法
   int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
   
   //set（）
   int i = key.threadLocalHashCode & (len-1);
   
   //get()
   int i = key.threadLocalHashCode & (table.length - 1);
   ```
   
   可以看到都和ThreadLocal的threadLocalHashCode有关，通过hashCode和length进行位运算确定出索引值i，i即是在table数组中的位置。
   
   ```java
   private final int threadLocalHashCode = nextHashCode();
   
   private static final int HASH_INCREMENT = 0x61c88647;
   
   private static int nextHashCode() {
   	return nextHashCode.getAndAdd(HASH_INCREMENT);
   }
   ```
   
   在每一次new出ThreadLocal的时候，threadLocalHashCode就被nextHashCode()初始化成为一个常量，并且每次threadLocalHashCode的初始化都会自增一次，增量为0x61c88647。
   
   重点在于这个```HASH_INCREMENT=0x61c88647```，一个如此神奇的参数。



## 5. 散列	

散列(Hash)也称为哈希，通俗点讲，就是无论输入端给什么数据，输出端都是一个数字；专业点，将输入映射到数字。而散列这必须满足一些要去。

1. 必须一致，同样的输入，必须每次得到相同的输出。

2. 不同的输入应该要尽可能得到不同的数字，如果好几个不同输入都得到相同的输出，这就不是一个好的散列，最完美理想的情况下，不同的输入将映射到不同到的数字。

   

## 6. 斐波那契数列

[详细请看李永乐老师的斐波那契数列讲解](https://www.bilibili.com/video/BV1is411E7df)

斐波那契数列的递推式

$$
a_1=a_2=1 
\qquad 
a_n=a_{n-1}+a_{n-2}
$$

例如数列是
```
1 1 2 3 5 8 13 21 ... n 
```
通过前一项除以后一项等到以下，虽然大小是在变化，但是都会趋近于0.618这个数字，实际上就是黄金分割数

```
1/1=1
1/2=0.5
2/3=0.67
3/6=0.6
5/8=0.625
8/13=0.615
```

$$
\frac{a_{n-1}}{a_n}\approx 0.618
$$

## 7. 黄金分割

[详细请看李永乐老师的黄金分割讲解](https://www.bilibili.com/video/BV1Gs411L7Rf)

$$
\overline{a \qquad \qquad c \;\;\;\;\;\;\;b}
$$

如上所示，令ab=1，ac=x，cb=1-x

$$
\frac{1-x}{x}=\frac{x}{1} \\[2ex] 
\Rightarrow  x^2+x-1=0 \\[2ex] 
\Rightarrow x=\frac{-1+\sqrt{5}}{2} \approx 0.618
$$



## 8. 0x61c88647

经过前面的铺垫，开始讲讲为何选择```HASH_INCREMENT=0x61c88647```一个如此神奇的参数

1. 进制的转换
	```
	十六进制
	0x61c88647
	十进制
	1640531527
	二进制
	1100001110010001000011001000111
	```

2. 

   ```java
   long l1 = (long) ((1L << 32) * ((Math.sqrt(5) - 1)/2));
   System.out.println("as 32 bit unsigned: " + l1);
   System.out.println("as 32 bit signed:   " + (int) l1);
   System.out.println("MAGIC = " + 0x61c88647);
   ```

   可以看到`0x61c88647`与`(Math.sqrt(5) - 1)/2`产生了关系，而`(Math.sqrt(5) - 1)/2`即是0.618黄金切割

   ```java
   public static void main(String[] args) {
   	int HASH_INCREMENT = 0x61c88647;
   	int[] data = new int[15];
   	for (int i=0;i<15;i++){
   		data[i]= ((HASH_INCREMENT*(i+1))& 15);
   	}
   	for(int i = 0;i<data.length;i++){
   		System.out.print(data[i] + " ");
   	}
       System.out.println("\n-----------------------");
       //为了方便查看是否有重复，对数据进行一次冒泡排序
       for(int i = 0;i<data.length;i++){
       	for(int j=i+1;j<data.length;j++){
           	if(data[i]<data[j]){
               	int p = data[i];
                   data[i] = data[j];
                   data[j] = p;
            	}
        	}
   	 }
        for(int i = 0;i<data.length;i++){
        	System.out.print(data[i]+" ");
        }
   }
   ```

   结果

   ```
   7 14 5 12 3 10 1 8 15 6 13 4 11 2 9 
   -----------------------
   15 14 13 12 11 10 9 8 7 6 5 4 3 2 1 
   ```

   可以看到当容量是16的时候，均匀的分布在数组里，没有冲突。

## 9. ThreadLocal与内存泄漏

因为ThreadLocalMap.Entry对ThreadLocal进行弱引用和对values进行强引用，但是它不会参考ReferenceQueue去发现那些弱引用被清除，即是ThreadLocal经过一次生命周期被回收了，value也不会被立即回收，因此造成了ThreadLocal的内存泄露。

这种情况毕竟容易出现的当线程属于线程池中，因为线程被重复使用，没有退出的话，而ThreadLocal为空后被回收，当时这个entry已经保存在table，如果不remove的话，就会把这个table变的越来越大，因此当使用完get()应该调用remove()清理掉数据。当然，如果只单独一个线程用完就退出的话，在exit()方法中就为threadLocals=null操作，这个时候就不会出现内存泄露。

来看看remove()的过程

```java
private void remove(ThreadLocal<?> key) {
	Entry[] tab = table;
	int len = tab.length;
	int i = key.threadLocalHashCode & (len-1);
	for (Entry e = tab[i];
		e != null;
		e = tab[i = nextIndex(i, len)]) {
        //通过threadLocalHashCode算计出key对应的index获得tab的entry
        //清理弱引用，并清理旧的entry
		if (e.get() == key) {
			e.clear();
			expungeStaleEntry(i);
			return;
		}
	}
}
```

```java
private int expungeStaleEntry(int staleSlot) {
	ThreadLocal.ThreadLocalMap.Entry[] tab = table;
    int len = tab.length;
    // 清空指定的value
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

     // Rehash until we encounter null
     ThreadLocal.ThreadLocalMap.Entry e;
     int i;
     for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
          i = nextIndex(i, len)) {
      	ThreadLocal<?> k = e.get();
        if (k == null) {
        	e.value = null;
            tab[i] = null;
            size--;
         } else {
            //重新计算index位置
            int h = k.threadLocalHashCode & (len - 1);
            //若果不在相同的位置，则老位置的清空，找到下个可以的位置，把e插入
            if (h != i) {
            	tab[i] = null;
                 // Unlike Knuth 6.4 Algorithm R, we must scan until
                 // null because multiple entries could have been stale.
                 while (tab[h] != null)
                        h = nextIndex(h, len);
                 tab[h] = e;
                }
            }
        }
     return i;
 }
```

