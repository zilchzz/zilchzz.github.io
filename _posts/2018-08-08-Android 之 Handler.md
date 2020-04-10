---
layout:     post
title:      Android 之 Handler
subtitle:   看完就明白 Handler 机制了
date:       2018-08-08
author:     钟瞻忠
catalog: true
tags:
    - Android
---

<blockquote>
    相信每个 Android 开发者都不会对 Handler 感到陌生。在平时开发中， Handler 最常用的场景就是在请求网络数据后通知主线程去刷新视图。在早年 EventBus 跟 RxJava 都尚未出现或者说流行起来的时候，绝大部分开发者都是通过 Handler 去完成线程间通信的。另外，Handler 在面试中也是一块经常问到的知识点。现在，让我们来好好剖析一下 Handler 的源码实现。
</blockquote>

<h2>简介</h2>
  官网 Android Developers 中对于 Handler 的用途有两个描述：
<blockquote>
  1.To schedule messages and runnables to be executed at some point in the future;<br/>
  2.To enqueue an action to be performed on a different thread than your own;
</blockquote>
  也就是说，Handler 可以用来规划未来的任务，或者将任务提交到其他线程中去执行。通常，我们通过 Handler 来完成非 UI 线程跟 UI 线程的通信。但需要注意的是， Handler 并非只能用来完成子线程跟主线程间的通信，准确的说，<code>它可以完成任意两个线程间的通信工作，只是非 UI 线程跟 UI 线程的使用场景更为常见而已</code>。  
  除此之外，Android Framework 中也大量使用 Handler 来完成线程间的通信工作。常见的如 ActivityThread 中，会使用继承自 Handler 的内部类 H 来完成大部分当前 App 的子、主线程切换工作。

```java
public final class ActivityThread extends ClientTransactionHandler {
	final H mH = new H(); //Handler 对象的初始化

	class H extends Handler {
    //省略部分源码
    public static final int RECEIVER                = 113;
    public static final int STOP_SERVICE            = 116;
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case RECEIVER:
                handleReceiver((ReceiverData)msg.obj);
                break;
            case STOP_SERVICE:
                handleStopService((IBinder)msg.obj);
                break;
        }
    }
}
```

  可以看到，ActivityThread 会在收到广播以及停止服务的请求时，通过内部类 H 的实例来将任务切换到主线程中去执行。因此，我们注册的广播其实都是在主线程中执行 onReceive 方法的。  
  除了 ActivityThread 外，IntentService 跟本地广播中都有使用到 Handler 。可见，Handler 在 Android 中占据了非常重要的地位，是 Androlid 中消息通信机制的基础。
  

<h2>源码解析</h2>
<h3>概览</h3>
  在剖析源码前，我们需要对 Handler 体系的组成有一些粗略的了解。Handler 体系包含了 Looper、Message、MessageQueue 这三块，其中：

>
  Message 是携带数据的载体；  
  MessageQueue 用来负责存储消息；  
  Looper 负责在一个线程里从 MessageQueue 中取出消息并交给 Handler 处理；  
  Handler 则同时负责投递消息以及处理消息；

  这样严肃的讲可能一部分不熟悉 Handler 的同学不好理解，我们来举个通俗的例子。TestHandler 的实例 mTestHandler 在 A 线程将一个 Message 放到了 MessageQueue 中，注意哦，此时 Message 还是在线程 A 中。与此同时，Looper 在 B 线程中已经在不断的从 MessageQueue 中取消息了，一旦取到消息，就会将信息在 B 线程中交给刚刚用来提交消息的那个实例 mTestHandler 来处理。这样，我们就完成了线程间的通信，同时也完成了数据实体的传递。  
  那么，这些实例之间是如何联系起来的呢？接下来我们将以上的例子拆分细致之后带着问题来看源码。
  

<h3>源码</h3>
<h4>Handler 的实例化</h4>
  在上面 ActivityThread 的代码中，我们可以看到，初始化一个 Handler 是非常简单的。但是在 new Handler() 后，源码中的逻辑是怎样的呢？
  

```java
public class Handler {
    //省略部分代码
    public Handler() {
        this(null, false);
    }
    public Handler(Callback callback, boolean async) {
        mLooper = Looper.myLooper();
        mQueue = mLooper.mQueue;
    }
}
```



  可以看到，在我们创造一个无参的 Handler 实例时，最终会调用到有两个参数的构造方法。在第二个构造方法中，会拿到 Looper 中的 MessageQueue 对象，但是这时候 Looper 好像并没有被初始化。那 Looper 是在何时初始化的，Looper 内部的 MessageQueue 实例 mQueue 又是何时被赋值的呢？要弄懂这个问题，需要先了解一下 ThreadLocal 的基础知识。
  

<h4>ThreadLocal</h4>
  在上面的代码中，Looper 实例是通过 Looper 类中的一个静态方法 myLooper() 拿到的：
    

```java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```
  看到这里可以发现，<code>ThreadLocal 是一个存在于 Looper 中的静态常量，这意味着它是所有 Looper 对象共享的</code>。接下来我们再看一下 ThreadLocal 的 get() 跟 set() 方法：
```java
public class ThreadLocal {
		public T get() {
  	  Thread t = Thread.currentThread();
    	ThreadLocalMap map = getMap(t);
	    if (map != null) {
  	      ThreadLocalMap.Entry e = map.getEntry(this);
    	    if (e != null) {
      	      @SuppressWarnings("unchecked")
        	    T result = (T)e.value;
          	  return result;
	        }
  	  }
    	return setInitialValue();
		}
  
		public void set(T value) {
    	Thread t = Thread.currentThread();
	    ThreadLocalMap map = getMap(t);
  	  if (map != null)
    	    map.set(this, value);
	    else
  	      createMap(t, value);
		}
}	
```
  从 set() 方法中可以看到，在存储 Looper 对象时，会从当前线程拿出一个 ThreadLocalMap 对象。将 ThreadLocal 自己作为键，将 Looper 作为 value 存储到 map 中。所以虽然所有的 Looper 对象共享同一个 ThreadLocal  对象，但在不同的线程中，存储的 ThreadLocalMap 就不同。因此我们可以理解为，每一个需要用 Handler 来处理消息的的线程中，都存在一个不同的 ThreadlocalMap 对象，在这个对象中，都使用一个相同的 key ，也就是 Looper 中的静态常量 sThreadLocal，而他们的 value 则是不同的 Looper 对象。  
  再看 get() 方法，首先会从当前线程拿到 ThreadlocalMap 的对象，然后从这个对象中将 Looper 取出并返回。这样，就通过 ThreadLocal 对象巧妙的完成了 Looper 对象的存取工作。  
  综上，我们说这个 Looper 是绑定在线程中的，是一个线程作用域变量。而 Looper 就是用来在<code>固定的线程</code>中轮询消息的，所以将它绑定在一个线程中是非常合适的。  
  那 Looper 是如何初始化的呢？
  

<h4>Looper 的初始化</h4>
  这个问题，我们需要分成两种情况去讨论：一是子线程跟主线程之间通信；二是两个子线程之间通信。在这两种情况下，Looper 的初始化过程是不一样的。  
  在上面我们说到过，ActivityThread 会使用 Handler 来从子线程切换到主线程执行一些任务，这些任务是 App 的正常运行所必须完成的，比如保证 Activity 生命周期方法的正常执行。因此主线程中的 Handler 必定早就已经初始化了，而这一点我们可以从 ActivityThread 类中找到答案：
```java
public static void main(String[] args) {
    //省略部分代码
    Looper.prepareMainLooper();
}
```
  哇哦，这里竟然有一个我们熟悉的 main 方法，而且 Looper 看起来就是在这里得到了初始化。其实在执行 Looper.prepareMainLooper() 之后，就已经将一个 Looper 存放到了主线程中的 ThreadLocalMap 中。所以，在主线程中使用 Handler 时，我们不需要再去初始化 Looper，因为主线程中的 Looper 在 App 启动之后就初始化好了。  
  那如果这个 Looper 被退出了会怎么样呢？这点大家放心吧，Android 已经帮我们考虑到了这个问题：
```java
public final class MessageQueue {
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
    void quit(boolean safe) {
      if (!mQuitAllowed) {
          throw new IllegalStateException("Main thread not allowed to quit.");
      }
  }
}
```

  其实，在调用 prepareMainLooper 的时候，会去创建一个 MessageQueue，而且会传递一个 false 到构造函数中并且赋值给了 mQuitAllowed 变量。而调用 Looper 的 quit 方法的时候，会调用到 MessageQueue 中的 quit 方法。从上面的方法中，我们可以看到，MessageQueue 的 quit 方法中判断了这个 mQuitAllowed 变量，如果是 false 的话，说明用户正在尝试停止主线程的 Looper ，这时会直接抛出异常。  
  现在我们来讨论第二种情况，子线程之间通信时 Looper 的初始化过程。其实这个问题可以转化为，如果我们希望使用 Handler 来处理子线程间的通信问题，代码应该如何写。

```java
Looper.prepare()
looper = Looper.myLooper()
handler = MyHandler(looper!!)
```

  上面的代码精简的回答了上面的问题。所以，我们现在只需要来看看这些代码到底做了些什么。
    

```java
public final class Looper {
    public static void prepare() {
        prepare(true);
    }
    private static void prepare(boolean quitAllowed) {
      if (sThreadLocal.get() != null) {
          throw new RuntimeException("Only one Looper may be created per thread");
      }
      sThreadLocal.set(new Looper(quitAllowed));
  }
}
```


  可以看到，Looper 中的 prepare() 方法只干了一件事，先判断当前线程是否已经绑定了一个 Looper ，如果已经绑定了的话，直接抛出异常；如果没有的话，则创建一个新的 Looper 对象并存在 ThreadLocal 中。之后，我们需要拿到 Looper 去创建一个 Handler 对象，即将 Handler 跟 Looper 绑定在一起。
    

<h4>消息的发送及存储</h4>
  现在，我们已经创建好了 Handler 对象，也已经将 Handler 对象跟 Looper 对象绑定在了一起，接下来可以使用 Handler 来发送消息了。  
  首先，我们看看在 Handler 中发送消息的那些方法：
```java
public class Handler {
    public final boolean post(Runnable r) {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }

    public final boolean postAtTime(Runnable r, long uptimeMillis) {
        return sendMessageAtTime(getPostMessage(r), uptimeMillis);
    }

    public final boolean postAtTime(Runnable r, Object token, long uptimeMillis){
        return sendMessageAtTime(getPostMessage(r, token), uptimeMillis);
    }

    public final boolean postDelayed(Runnable r, long delayMillis){
        return sendMessageDelayed(getPostMessage(r), delayMillis);
    }

    public final boolean postAtFrontOfQueue(Runnable r){
        return sendMessageAtFrontOfQueue(getPostMessage(r));
    }

    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }

    private static Message getPostMessage(Runnable r, Object token) {
        Message m = Message.obtain();
        m.obj = token;
        m.callback = r;
        return m;
    }

    public final boolean sendMessage(Message msg) {
            return sendMessageDelayed(msg, 0);
    }

    public final boolean sendEmptyMessage(int what)  {
        return sendEmptyMessageDelayed(what, 0);
    }

    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }

    public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageAtTime(msg, uptimeMillis);
    }

    public final boolean sendMessageDelayed(Message msg, long delayMillis)  {
        if (delayMillis &lt; 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

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

    public final boolean sendMessageAtFrontOfQueue(Message msg) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, 0);
    }

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
}
```
  这些方法可以分为两类，一类是 sendMessagexxx，一类是 postxxx 。其中，带有 post 的方法都会接受一个 Runnable 的参数；而 sendMessage 方法则都会接受一个 Message 参数。  
  进一步我们又可以发现，post 系列方法最终都会将传入的 Runnable 当做 Message 中的 callback 成员变量去构造一个实例，然后最终调用 sendMessage 系列方法。  
  而 sendMessage 系列方法，会接着调用到 Handler 中的 enqueueMessage 方法，并且将当前的 Handler 对象赋值给 Message 中的 target。然后该方法又直接调用 MessageQueue 中的 enqueueMessage 方法。这一过程可以总结如下图（图片取自网络，侵删）：  
<a href="https://i.loli.net/2019/03/07/5c80ca2335cd6.png"><img src="https://i.loli.net/2019/03/07/5c80ca2335cd6.png" alt="" /></a>
  接下来，我们分析一下 MessageQueue 中的 enqueueMessage 方法：
    

```java
boolean enqueueMessage(Message msg, long when) {
       //删除部分代码
        synchronized (this) {
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when &lt; p.when) {
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when &lt; p.when) {
                        break;
                    }
                }
                msg.next = p;
                prev.next = msg;
            }
        }
        return true;
    }
}
```

  在这一段删减后的代码中，我们可以清晰看到消息在插入消息队列时的逻辑。在该段逻辑中，存在 prev 以及 next，因此，我们可以推测 MessageQueue 其实并不是一个队列，而是一个单向链表。  
  具体看代码，在第一次插入消息时，该消息直接存在于头部。而在插入第二条以及之后的消息时，会拿新消息的执行时间跟当前头部消息的执行时间做对比。如果新消息需要更早执行，则直接将新消息变成新的头部。否则，会从第二条消息开始迭代，一直迭代到链表最后或者新消息的执行时间比当前迭代到的消息的执行时间更前的时候，这时候就将新消息插入到这个位置。  
  这样，整个发送、存储消息的流程就已经分析完了。但是在没有开启消息循环之前，发送出去的消息是无法得到处理的，因为 Looper 还没有彻底运行起来，即没有调用 Looper 对象中的 loop()。
  

<h4>开启消息循环 -- loop()</h4>
  在创建好 Looper 对象并且发送消息之后，我们还需要使用这个 Looper 对象去开始消息循环，如果不开启循环的话，Looper 将不会从消息队列中取出消息给 Handler 处理。那 loop() 方法又干了些什么呢？
    

```java
public static void loop() {
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
        // No message indicates that the message queue is quitting.
        return;
        }
        try {
            msg.target.dispatchMessage(msg);
            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
}
```



  loop() 方法有大段的内容，上面我只截取了最重要的逻辑。可以看到，loop() 中存在一个无限 for 循环，并且是一个阻塞方法。当 MessageQueue 中已经没有消息时，便退出循环。在拿到消息之后，便会从 Message 中拿到 target ，并调用它的 dispatchMessage 方法。  
  上面我们已经分析过了，这个 target 变量就是发送消息的 Handler 对象。因此，取出消息后，会在取消息的那个线程使用 Handler 对象去将消息处理掉。
    

```java
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
private static void handleCallback(Message message) {
        message.callback.run();
}
```



  在这个方法中，首先会判断消息中是否有一个 callback 实例，这个 callback 实例也就是上面我们分析过的通过 postxxx() 方法提交过来的 Runnable 对象。当 callback 存在时， 直接调用 handleCallback(Message message) ，也就是直接调用 Runnable 实例的 run 方法。注意，这里并没有新开一个线程去执行任务，而是直接调用了 run 方法。  
  而如果这个 Message 对象中没有 callback 对象，则会直接使用内部的 mCallback 对象去处理这个消息，它是一个接口对象：    
```java
public interface Callback {
        public boolean handleMessage(Message msg);
}
```
  当这个接口对象也不存在时，直接使用 Handler 对象中的 handleMessage 方法。这样，就完成了消息的处理。同时，我们也已经将整个 Handler 消息机制从源码的角度剖析了一遍。
    
<h4>处理内存泄漏</h4>
  在写 Handler 的时候，很多人一开始都是直接在 Activity 中创建一个继承自 Handler 的内部类，这样做可以非常简单的完成任务。但是相信这样做的人也会得到一个非常显眼的警告信息，那就是<code>This Handler class should be static or leaks might occur</code>。在解决这个可能存在的泄漏之前，我们需要分析一下，为什么这里存在内存泄漏。  
  我们知道，一个非静态的内部类在初始化后，必定会持有外部类的引用。所以问题就在这里，当内部类 Handler 初始化好之后，会持有 Activity 的引用。当我们使用这个 Handler 去发送一个延时消息时，这个 Handler 会被长久的引用，可能会长过 Activity 的生命周期。因此，会导致 Activity 无法被销毁，产生内存泄漏。  
  在知道问题后，我们如何去解决这个问题呢？这个问题也就是如果解决内部类造成的内存泄漏问题。  
  要解决这个问题，无非有两个方法：

>
  1.将这个 Handler 声明成静态内部类；  
  2.将这个 Handler 抽取出来，定义成一个单独的类；

<h3>常见问题</h3>
<h4>主线程会因为 Looper.loop() 卡死吗？</h4>

  在 ActivityThread 的 main 方法中，会在最后开启 loop 循环，而 loop 方法又是阻塞的，那为什么不会卡死呢？  
  首先，loop() 中的阻塞跟 UI 线程上执行耗时操作时卡死，是完全不同的两个概念。Looper 上的阻塞，前提是没有输入事件，MessageQueue 为空，Looper 空闲状态。这时候线程会进入阻塞状态，释放 CPU 执行权，处于等待唤醒的状态。而在 UI 中执行耗时操作导致卡死，前提是要有输入事件，MessageQueue 不为空，Looper 正常轮询，线程没有阻塞。而如果该事件在主线程执行超过 5s 的话，在这 5s 内都没有办法执行其他事件，所以会导致卡死。  
  所以，主线程中 loop() 之后如果一直没有事件过来的话会被挂起，释放了 CPU 执行权。同时一个 app 不可能只存在一个线程，肯定还存在其他线程，比如说 Binder 线程。这些线程都可以发送消息到主线程去唤醒它，从而处理消息。所以 Looper 中的 loop() 并不会导致 App 卡死，也不会导致 CPU 资源的浪费。
  

<h4>发送延时消息时，使用什么时间计时？</h4>
  这个问题，我们可以通过源码去找到答案。

 ```java
public class Handler {
    public final boolean sendMessage(Message msg){
        return sendMessageDelayed(msg, 0);
    }
    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis &lt; 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
}
 ```

  可以看到，在发送消息时，会通过 SystemClock.uptimeMillis() 方法去获取时间，注意，这个时间并不是自然时间，而是系统上一次开机到现在的时间，并且在手机休眠时不会累加。而在息屏状态下，手机是容易进入休眠状态的。  
  那么，延时消息是如何被延时执行的呢？在正文中我们说到，将消息插入到 MessageQueue 中的时候，会根据 Message 中的变量 when 来决定消息的先后顺序。然后，loop() 中取消息时，调用的其实是 MessageQueue 中的 next() ，下面看 MessageQueue 中的 next()，已经将注释非常清晰的写到了代码中，看完这个方法应该可以明白大概的逻辑了。

 ```java  
public final class MessageQueue {
    Message next() {
        final long ptr = mPtr;
        if (ptr == 0) { //在执行了 Looper 对象的 quit 方法后，会将这个变量 ptr 置为 0，这时候会返回 null，导致退出 loop() 方法
            return null;
        }
        int pendingIdleHandlerCount = -1;
        int nextPollTimeoutMillis = 0; //第一次休眠 0S
        for (;;) {
            //上一条消息需要睡眠的时间（或者第一次不休眠）
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                final long now = SystemClock.uptimeMillis(); //拿到当前开机时间
                Message prevMsg = null;
                Message msg = mMessages;
                //如果这条消息没有 target ，那么这是刻意设置的一个屏障，在这个屏障之后的所有消息都不会被执行
                if (msg != null &amp;&amp; msg.target == null) {
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                        //如果有屏障的话，找到第一条异步消息，记住这时候 prevMsg 不为 null
                    } while (msg != null &amp;&amp; !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now &lt; msg.when) { //如果这条消息还没到时间执行的话，阻塞以下时间
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        mBlocked = false;//有消息，没有被阻塞
                        if (prevMsg != null) { //如果 prevMsg 不为 null 的话，说明遇到了一个屏障
                            //因为有屏障，而且下面的 msg.next 会被置空，所以这里将上一条消息连接到本消息的下一条消息，防止链表断掉
                            prevMsg.next = msg.next;
                        } else {//没有屏障的话，直接将 mMessages 赋值为下一条消息
                            mMessages = msg.next; //给 mMessages 这个成员变量赋值成下一个要处理的消息
                        }
                        msg.next = null; //将当前消息的下一个指针置为 null
                        msg.markInUse();
                        return msg; //到时间执行了，直接执行
                    }
                } else {
                    nextPollTimeoutMillis = -1; //msg 为 null，长时间休眠，但是在插入一条消息后，会在 enqueueMesaage 中唤醒该消息
                }
            }
            //pendingIdleHandlerCount 这个数字是所有添加到 mIdleHandlers 中 IdleHandler 的数量
            //这些 IdleHandler 其实就当当前拿到的 msg 为 null 的时候（也即接下来没啥事的时候），才会开始执行的任务的包装类
            for (int i = 0; i &lt; pendingIdleHandlerCount; i++) { 
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null;
                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }
                if (!keep) { //如果不需要保持的话，在执行完成后直接丢弃掉这个任务
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }
        }
            //重置两个数量
            pendingIdleHandlerCount = 0;
            nextPollTimeoutMillis = 0;
}
 ```

  关于 nativePollOnce 跟 nativeWake 方法，底层使用了管道（Pipe）技术。在 Linux 中，当有两个进程需要通信时，主进程拿到读取的描述符等待读取，没有内容则阻塞住。而另一个进程则拿到写入描述符去写内容，同时唤醒主进程，主进程会拿到读取描述符读取到的内容，继续执行。

<h4>有几种方式获取 Message 实例？</h4>
  Message 有一个 public 的无参构造方法，因此我们可以很简单的去创建一个 Message 实例。但是 Message 同时也提供了一系列 obtain 方法去获取，这些方法最终都会通过 obtain() 去获取一个消息池中的消息：

 ```java
public final class Message implements Parcelable {
    public static Message obtain() {
        synchronized (sPoolSync) { //同步方法
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; //清除消息正在使用的 flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
}
 ```
  这里的 sPool 是一个私有静态对象，当 Looper 对象在 loop() 方法中将一个消息交给 Handler 处理完毕后，会调用将这个消息的 recycleUnchecked() 。在调用这个方法之后，这个消息就会擦除所有信息，赋值给 sPool 这个静态对象。  
  因此，在我们获取消息时，可以使用这个方法，避免重新创建一个 Message ，浪费内存。

<h2>结语</h2>
  至此，Handler 体系的源码已经大体分析完了。彻底消化完这部分内容后，相信大家不管在日常使用中还是面试中都可以应付的过来。  
  谢谢你来看我的文章，同时也希望能给你带来帮助。
  
>
  本文作者：ZhanZhong  
  版权声明：本博客所有文章均采用 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/">CC BY-NC-SA 3.0</a>  许可协议，如需转载，请注明出处！


