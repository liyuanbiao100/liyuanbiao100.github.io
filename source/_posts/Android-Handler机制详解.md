---
title: Android Handler机制详解
categories: Android
tags:
  - Android
  - Handler
abbrlink: fecfb901
---

### 前言

`Handler`是Android的消息机制，他能够很轻松的在线程间传递数据。由于Android开发规范的限制，我们不能在主线程执行耗时操作（如网络，IO操作等），不能在子线程更新UI，所以`Handler`大部分用来在耗时操作与更新UI之间切换。这让很多人误以为`Handler`就是用来更新UI的，其实这只是它的一小部分应用。

### 开始

我相信大多数人对`Handler`的用法已经烂熟于心了，这篇文章不会去探讨`Handler`的使用，而是着重从源码上分析`Handler`的运行机制。

想要了解`Handler`的运行机制，我们需要了解 `MessageQueue` ，`Message`，`Looper` 这几个类。

* `MessageQueue` 的意思就是消息队列，它存储了我们需要用来处理的消息`Message`。
* `Message`是消息类，内部存在一个`Bundle`对象和几个`public`字段存储数据，`MessageQueue`作为一个消息队列不能自己处理消息，所以需要用到`Looper`。
* `Looper`是一个循环装置，他负责从不断从`MessageQueue`里取出`Message`，然后回调给`Handler`的`handleMessage`来执行具体操作。
* `Handler`在这里面充当的角色更像是一个辅助类，它让我们不用关系`MessageQueue`和`Looper`的具体细节，只需要关系如何发送消息和回调的处理就行了。

上面讲了几个关键类在`Handler`运行机制中的职责，相对大家对Handler机制有个粗略的了解。

我相信各位看官在阅读这篇文章前都是带着问题的，我们将通过问题来解答大家的疑惑。

### 分析
#### Looper
在分析`Looper`之前，我们还需要知道`ThreadLocal`这个类，如果对`ThreadLocal`还不太了解，可以去看我的另一篇文章[《ThreadLocal详解》](/2017/10/10/ThreadLocal详解/)。
##### Looper是如何创建？
`Handler`执行的线程和它持有的`Looper`有关。每个`Thread`都可以创建唯一的Looper对象。

```java
 //为当前线程创建Looper对象的方法。
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
    //使用ThreadLocal来存储当前线程的Looper对象，这保证了每个线程有且仅有一个Looper对象。
    //这里做了非空判断，所以在同一个线程prepare方法是不允许被调用两次的
    //第一次创建好的Looper对象不会被覆盖，它是唯一的。
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    
```
那么主线程的`Looper`对象是怎么创建的呢？

```java
public static void prepareMainLooper() {
//其实主线程创建Looper和其他线程没有区别，也是调用prepare()。
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            //但是Looper用sMainLooper这个静态变量将主线程的Looper对象存储了起来
            //可以通过getMainLooper()获取，存储MainLooper其实非常有作用，下面会讲到。
            sMainLooper = myLooper();
        }
    }

    public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }
```
##### Looper是如何从MessageQueue取出消息并分发的？
Looper分发消息的主要逻辑在loop方法里

```java

 /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        //保证当前线程必须有Looper对象，如果没有则抛出异常，调用Looper.loop()之前应该先调用Looper.prepare().
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        //Looper需要不断从MessageQueue中取出消息，所以它持有MessageQueue对象
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
           //这里开始执行死循环，queue通过调用next方法来取出下一个消息。
           //很多人很疑惑死循环不会相当耗费性能吗，如果没有那么多消息怎么办？
           //其实当没有消息的时候，next方法会阻塞在这里，不会往下执行了，性能问题不存在。
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                //这里满足了死循环跳出的条件，即取出的消息为null
                //没有消息next不是会阻塞吗，怎么会返回null呢？
                //其实只有MessageQueue停止的时候（调用quit方法），才会返回null
                //MessageQueue停止后，调用next返回null，且不再接受新消息，下面还有详细介绍。
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            //这里的msg.target是Handler对象，分发消息到Handler去执行。
            //有人问主线程可以创建这么多Handler，怎么保证这个Handler发送的消息不会跑到其它Handler去执行呢？
            //那是因为在发送Message时，他会绑定发送的Handler，在此处分发消息时，也只会回调发送该条消息的Handler。
            //那么分发消息具体在哪个线程执行呢？
            //我觉得这个不该问，那当然是当前方法在哪个线程调用就在哪个线程执行啦。
            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

        //这里对Message对象进行回收，会清空所有之前Message设置的数据。
        //正是因为Message有回收机制，我们在创建消息的时候应该优先选择Message.obtain(). 
        //如果发送的消息足够多，Message缓存的Message对象不够了，obtain内部会调用new Message()创建一个新的对象。
            msg.recycleUnchecked();
        }
    }
```

##### Looper 分发的消息在哪个线程执行？
先给大家展示一段`Looper`文档上的示例代码

```java
class LooperThread extends Thread {
   public Handler mHandler;
  
   public void run() {
        Looper.prepare(); //创建LooperThread的Looper对象
  
        mHandler = new Handler() {
            public void handleMessage(Message msg) {
              //处理发送过来的消息
            }
        };
    
        Looper.loop(); //开始循环消息队列
   }
}
```
上面这段代码相信很多人都写过，这是一段在子线程创建Handler的案例，其中`handleMessage`所执行的线程为`LooperThread`，因为`Looper.loop()`执行在`LooperThread`的`run`方法里。可以在其他线程通过`mHandler`发送消息到`LooperThread`

##### 如果不调用`Looper.prepare()`直接`new Handler()`会怎么样呢？
我们可以查看`Handler`的源码看看无参构造是如何运行的

```java
public Handler() {
//调用两参构造
    this(null, false);
}
    
public Handler(Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }
//获取当前线程的Looper，如果不创建Looper会抛出异常。
//主线程我也没看到有调用Looper.prepare()啊，怎么在主线程不会抛异常呢？这个看下一个问题。
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
##### 主线程的Looper对象在哪里创建的？
从上一个问题可以看出如果不调用`Looper.prepare()`直接`new Handler()`就会抛出异常` 

```java
throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");`
```
那么主线程的`Looper`在哪里创建的呢？首先它是创建了的，因为`Looper.getMainLooper() != null`，其实`MainLooper`创建的时间比我们想象的早，它在`ActivityThread`类里面，`ActivityThread`是`Android`的启动类，`main`方法就在里面（如果有人问你Android有没有main方法，你应该知道怎么回答了吧），而`MainLooper`就是在`main`方法里面创建的。

上代码：

```java
//android.app.ActivityThread
public final class ActivityThread {
    ...
    public static void main(String[] args) {
         
        SamplingProfilerIntegration.start();
    
        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);
    
        Environment.initForCurrentUser();
    
        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());
    
        Security.addProvider(new AndroidKeyStoreProvider());
    
        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);
    
        Process.setArgV0("<pre-initialized>");
    
        //注意这里，这里创建了主线程的Looper
        Looper.prepareMainLooper();
    
        ActivityThread thread = new ActivityThread();
        thread.attach(false);
    
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
    
        AsyncTask.init();
    
        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }
        //开启消息循环
        Looper.loop();
    
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
    
}
```
##### MainLooper可以用来做什么
###### 判断当前线程是否为主线程
因为Looper是在某一线程唯一的，那么可以在么做。如果

```java
 public static boolean isMainThread() {
        //如果当前线程的Looper和MainLooper是同一个对象，那么可以认为当前线程是主线程
        return Looper.myLooper() == Looper.getMainLooper() ;
}
```

但是也有人说下面这样也可以

```java
public static boolean isMainThread() {
    //这个方法其实是不准确的，线程的名称是可以随便更改的。
    return Thread.currentThread().getName().equals("main");
}
```

所以用`Looper`来判断主线程是很好的做法

###### 创建运行在主线程的Handler
`Handler`除了有无参构造，还有一个可以传入`Looper`的构造。通过指定`Looper`，可以在任意地方创建运行在主线程的`Handler`

```java
    class WorkThread extends Thread{
        private Handler mHandler;

        @Override
        public void run() {
            super.run();
            mHandler = new Handler(Looper.getMainLooper()) {
                @Override
                public void handleMessage(Message msg) {
                    super.handleMessage(msg);
                    //运行在主线程
                }
            };
            mHandler.sendEmptyMessage(0);
        }
    }
```

##### Looper的quit方法和quitSafely方法有什么区别

下面是`Looper`两个方法的源码

```java
public void quit() {
    mQueue.quit(false);
}
public void quitSafely() {
    mQueue.quit(true);
}
```
可以看出实际上是调用的`MessageQueue`的`quit`方法
下面是`MessageQueue`的源码

```java
//android.os.MessageQueue
void quit(boolean safe) {
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {
                return;
            }
            mQuitting = true;
            //如果调用的是quitSafely运行removeAllFutureMessagesLocked，否则removeAllMessagesLocked。
            if (safe) {
            //该方法只会清空MessageQueue消息池中所有的延迟消息，
            //并将消息池中所有的非延迟消息派发出去让Handler去处理，
            //quitSafely相比于quit方法安全之处在于清空消息之前会派发所有的非延迟消息。
                removeAllFutureMessagesLocked();
            } else {
            //该方法的作用是把MessageQueue消息池中所有的消息全部清空，
            //无论是延迟消息（延迟消息是指通过sendMessageDelayed或通过postDelayed等方法发送的需要延迟执行的消息）还是非延迟消息。
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }

```
无论是调用了`quit`方法还是`quitSafely`方法，`MessageQueue`将不再接收新的`Message`，此时消息循环就结束，`MessageQueued`的`next`方法将返回`null`，结束`loop()`的死循环.这时候再通过`Handler`调用`sendMessage`或`post`等方法发送消息时均返回`false`，表示消息没有成功放入消息队列`MessageQueue`中，因为消息队列已经退出了。

#### Message

##### Message.obtain()和new Message()如何选择
`Message`提供了`obtain`等多个重载的方法来创建`Message`对象，那么这种方式和直接`new`该如何选择。下面看看`obtain`的代码。

```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message(); //只有当从对象池里取不出Message才去new
}

void recycleUnchecked() {
        //清除所有使用过的痕迹
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;

    //回收到对象池
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```

从上面代码可以看出，通过`obtain`方法是从对象池取，而`new`是创建了一个新的对象。我们应该使用`obtain`来创建`Message`对象，每次使用完后都会自动进行回收，节省内存。


未完待续......


