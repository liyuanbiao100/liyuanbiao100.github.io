---
title: Android Handler机制详解 - ThreadLocal
date: 2017-10-10 12:59:59
categories: Android
tags: 
    - Android
    - Handler
---

### 前言
ThreadLocal类位于java.lang包下，jdk1.2开始引入。ThreadLocal为每个使用该类变量的线程提供单独的副本，线程之间不会互相影响。就是说在A线程设置的值只能在A线程读取到，在B线程是读取不到的。

在多线程环境下，每个线程都有自己的数据。一个线程使用自己的局部变量比使用全局变量好，因为局部变量只有线程自己能看见，不会影响其他线程，而全局变量的修改必须加锁。
但是局部变量也有问题，就是在函数调用的时候，传递起来很麻烦。


### 基本使用

这个类一共提供了四个方法。

* `set(T value)` 将此线程局部变量的当前线程副本中的值设置为指定值。
* `get()`  返回此线程局部变量的当前线程副本中的值。
* `remove()`  移除此线程局部变量当前线程的值。
* `initialValue()` 返回此线程局部变量的当前线程的“初始值”。该实现返回 null；如果程序员希望线程局部变量具有 `null` 以外的值，则必须为 ThreadLocal **创建子类**，并**重写**此方法。通常将使用**匿名内部类**完成此操作。

下面看看ThreadLocal是如何解决这一问题的：


```java
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class ThreadLocalDemo {

    private ThreadLocal<String> threadLocal = new ThreadLocal<>();

    public void test() {
        threadLocal.set("hello");
        printValue(); //打印 main -> hello

        new Thread(new Runnable() {
            @Override
            public void run() {
                printValue(); //打印 Thread-0 -> null
            }
        }).start();
    }

    private void printValue() {
        System.out.println(Thread.currentThread().getName() + " -> " + threadLocal.get());
    }

    public static void main(String[] args) throws InterruptedException {
        ThreadLocalDemo demo = new ThreadLocalDemo();
        demo.test();

        Thread.sleep(1000); //防止程序结束
    }
}
```
从上面代码中可以看到不同线程获取到的值是不一样的，在主线程设置的值只能在主线程获取到，在子线程返回了一个`null`。同时把**ThreadLocal**生明为全局变量，也不会影响各线程之间的值。

### 源码分析

下面的代码取自**ThreadLocal**类，展示了最基本的几个方法。

```java
//每个Thread都有自己的ThreadLocalMap
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

 public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    //在不同线程调用这个方法获取到的ThreadLocalMap对象是不一样的，
    //所以从Map取到的对象更加不可能是一样了。
    if (map != null)
        //用当前对象作为key存储value
        map.set(this, value); 
    else
        createMap(t, value);
 }


 public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //用当前对象作为key获取value
        //由于是用当前对象作为key，所以一个ThreadLocal对象只能存储一个值。
        //如果想要存储几个值可以多用几个ThreadLocal。
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
    
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        //用当前对象作为key移除value
         m.remove(this);
}
    
```

上面的代码可以很容易看出**ThreadLocal**的原理，但是由于**ThreadLocal**的代码太多，这只是很小的一部分，为了便于大家理解问题，我写了一个类模拟**ThreadLocal**的实现。

```java
public class ThreadLocal<T> {

    //每个线程有单独的Map来保证变量在不同线程之间互不影响
    private static final Map<Thread, Map<ThreadLocal, Object>> tMap = new ConcurrentHashMap<>();

    public void set(T value) {
        Thread thread = Thread.currentThread();
        Map<ThreadLocal, Object> map = tMap.get(thread);
        if (map != null) {
            map.put(this, value);
        } else {
            map = new ConcurrentHashMap<>();
            tMap.put(thread, map);
            map.put(this, value);
        }
    }

    public T get() {
        Thread thread = Thread.currentThread();
        Map<ThreadLocal, Object> map = tMap.get(thread);
        if (map != null) {
            return (T) map.get(this);
        }
        return null;
    }

    public void remove() {
        Thread thread = Thread.currentThread();
        Map<ThreadLocal, Object> map = tMap.get(thread);
        if (map != null) {
            map.remove(this);
        }
    }
}
```

### 总结

每个**Thread**都有自己的**ThreadLocalMap**，存储**value**的时候会存储到自己的**ThreadLocalMap**，所以在哪个线程设置的**value**就只能在哪个线程获取到，其他**Thread**是获取不到的。因为**ThreadLocal**类用`this`当做**key**，所以每个**ThreadLocal**最多存储一个**value**。如果想要存储几个值可以多用几个**ThreadLocal**。

